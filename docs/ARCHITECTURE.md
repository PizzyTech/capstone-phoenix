# Architecture

## 1. Topology diagram
Internet
                             │
                             │ DNS: taskapp.13.61.142.226.nip.io
                             ▼
                ┌─────────────────────────┐
                │  AWS Security Group      │
                │  sg-0828699765e052f80    │
                │  80/443 open to 0.0.0.0/0 │
                │  6443 CLOSED to internet │
                └────────────┬─────────────┘
                             │
                             ▼
          ┌──────────────────────────────────┐
          │   Traefik Ingress Controller       │
          │   (k3s built-in, kube-system ns)   │
          │   TLS terminated here              │
          │   cert: taskapp-tls-cert            │
          │   (cert-manager + Let's Encrypt)    │
          └───────┬──────────────────┬─────────┘
                   │ path: /          │ path: /api
                   ▼                  ▼
    ┌──────────────────────┐  ┌──────────────────────┐
    │ frontend-service      │  │ backend               │
    │ ClusterIP :80         │  │ ClusterIP :5000       │
    └──────────┬────────────┘  └──────────┬─────────────┘
               │                            │
 ┌─────────────┼─────────────┐              │
 ▼             ▼             ▼              ▼
## 2. Node & network

- **Nodes**: 3x AWS EC2 `t3.small`, all in `eu-north-1a` (single AZ — see trade-offs below).
  1 control-plane (`ip-10-0-1-228`) running the k3s server, 2 workers (`ip-10-0-1-31`,
  `ip-10-0-1-53`) running k3s agents.
- **CIDR**: pod network `10.42.0.0/16` (k3s default, Flannel VXLAN backend), one /24 per
  node (e.g. `10.42.0.0/24`, `10.42.1.0/24`, `10.42.2.0/24`). Service network `10.43.0.0/16`.
- **Firewall (AWS Security Group `sg-0828699765e052f80`)**: only ports `80` and `443` are
  open to `0.0.0.0/0`. Port `6443` (Kubernetes API) is **not** exposed to the internet —
  it's only reachable from inside the VPC / via SSH tunnel, satisfying the hard constraint
  in the brief. SSH (`22`) is restricted to the operator's IP.

## 3. Request flow

A browser resolves `taskapp.13.61.142.226.nip.io` via nip.io's wildcard DNS (which maps
any `<ip>.nip.io` subdomain straight to that IP — no DNS provider needed). The request
hits the EC2 instance on port 443, where Traefik (k3s's built-in ingress controller)
terminates TLS using the certificate `taskapp-tls-cert`, issued by Let's Encrypt via
cert-manager's `letsencrypt-prod` ClusterIssuer over the HTTP-01 challenge. Traefik then
routes based on path: `/` goes to the `frontend-service` ClusterIP, which load-balances
across 3 frontend Pods; `/api` goes directly to the `backend` ClusterIP, which
load-balances across 2 backend Pods. The Flask backend talks to Postgres via the headless
`postgres-service` (ClusterIP: None, so DNS resolves directly to the single `postgres-0`
Pod IP). All inter-Pod traffic within the namespace is governed by NetworkPolicy
(see §5).

## 4. The single-server assumptions we fixed

| Single-server assumption | Why it breaks at scale | How we fixed it |
|---|---|---|
| Migrations run in the container entrypoint on boot | With 2+ backend replicas, both would race to run `alembic upgrade head` simultaneously against the same database | Migrations run as a separate one-shot Kubernetes `Job` (`backend-migration`), applied once via GitOps, decoupled from the replica entrypoint |
| Named Docker volume on one host | Pods reschedule to any node; a local volume tied to one machine would be lost or inaccessible after rescheduling | Postgres uses a `StatefulSet` with a `PersistentVolumeClaim` (`local-path` storage class) and `persistentVolumeClaimRetentionPolicy: Retain`, so a Pod delete/reschedule reattaches the same volume. Verified live: deleted `postgres-0`, all task/user data intact after restart |
| `ports:` published directly on the Docker host | Many Pods across many nodes need one single front door, not N separate host-port bindings | A single `Ingress` (Traefik) with one TLS cert routes both `/` and `/api` to the correct Service, which load-balances across all healthy Pods regardless of node |
| Container restarts on crash (`restart: always`) is "self-healing enough" | A crashed dependency (e.g. Postgres down) can leave an app process alive but non-functional; naive restart-on-crash doesn't catch this | `livenessProbe`/`readinessProbe`/`startupProbe` on every workload check real endpoints (`/api/health` includes a live DB connectivity check, `/healthz` for frontend, `pg_isready` for Postgres) — demonstrated live when Postgres briefly failed during a probe misconfiguration and backend correctly self-healed via its own liveness probe |
| One deployment step = brief downtime is acceptable | With real traffic, even a few seconds of downtime per deploy compounds; single-server deploys assume that's fine | `RollingUpdate` with `maxUnavailable: 0` on both backend and frontend; verified zero dropped requests (continuous 1-second health polling, all `200`s) through both an ordinary rollout and a live worker-node drain/failover |
| Plaintext env vars / `.env` file committed alongside code | A single-server `.env` file is manageable by hand; at cluster scale, secret sprawl across manifests risks accidental commits | Non-secret config lives in a `ConfigMap` (`backend-config`); real secrets (`DATABASE_USER`, `DATABASE_PASSWORD`, `SECRET_KEY`) live in a Kubernetes `Secret` created **out-of-band** (never committed) — only a placeholder template (`docs/templates/backend-secret.example.yaml`) is in git, deliberately kept outside the `manifests/` folder so Argo CD's sync path never touches the real Secret |
| Flat, "everything can reach everything" networking is fine on one host | On one machine, all processes share the same network namespace by default; on a cluster, uncontrolled Pod-to-Pod traffic is an unnecessary blast radius | Default-deny `NetworkPolicy` across the whole `taskapp` namespace, with narrow explicit allows: frontend reachable from any ingress; backend reachable only from frontend Pods **and** from Traefik (a gap we found and fixed — Traefik lives in `kube-system` and proxies `/api` directly to the backend Service, bypassing frontend, so it needed its own explicit allow rule); Postgres reachable only from backend Pods on port 5432 |

## 5. Choices & trade-offs

- **Raw YAML vs Helm vs kustomize**: raw YAML, one file per resource, all under `manifests/`.
  Chosen for transparency and because the app's shape is fixed for this project — Helm's
  templating value (multiple environments, parameterized installs) wasn't needed here, and
  raw YAML keeps every resource's diff readable in `git log` and in Argo CD's UI.

- **k3s built-in Traefik vs ingress-nginx**: kept Traefik (k3s's default). It already
  handled TLS termination and path-based routing without needing a separate controller
  installed, and switching to ingress-nginx would have added complexity with no
  functional benefit for this app's simple routing needs.

- **Same-origin `/api` path vs separate `api.<domain>` subdomain**: chose the same-origin
  `/api` path (`taskapp.<ip>.nip.io/api` vs `taskapp.<ip>.nip.io/`) on a single Ingress
  and a single TLS certificate. This avoids needing a second DNS entry and a second
  cert-manager Certificate resource, and matches how the frontend was already built
  (an nginx reverse-proxy config expecting `/api` on the same origin).

- **CNI / NetworkPolicy enforcement**: k3s ships Flannel (VXLAN) as the default CNI, which
  by itself does **not** enforce NetworkPolicy. However, k3s also runs a built-in
  lightweight NetworkPolicy controller (kube-router-based) alongside Flannel by default,
  unless explicitly disabled at server start. We verified this empirically rather than
  assuming it: applying a default-deny policy actually broke `/api` traffic from Traefik
  until we added an explicit allow rule, proving enforcement was real and active — not a
  policy sitting inertly in etcd with no effect.

- **Secrets approach**: out-of-band Secret creation (`kubectl create secret generic
  backend-secret ...`, run once, never in git), rather than Sealed Secrets or External
  Secrets. This satisfies the brief's stated fallback ("create the Secret out-of-band and
  let Argo ignore it") without adding a second controller/dependency. The trade-off: a
  fresh cluster rebuild from this repo alone would need the operator to manually recreate
  `backend-secret` before the app can start — this is documented explicitly in
  `docs/RUNBOOK.md`.

- **Single-replica Postgres, no PDB**: the brief explicitly allows a single-node control
  plane and doesn't require HA Postgres for Core credit. We deliberately did **not** add a
  PodDisruptionBudget for `postgres-0`, because with only one replica, `minAvailable: 1`
  would block voluntary eviction entirely and hang a node drain indefinitely. This is a
  known, accepted gap: draining the specific worker node that hosts `postgres-0` would
  cause real (if temporary) database downtime. Our live failover demo deliberately drained
  a different worker to avoid this — a genuine limitation of the current architecture, not
  something we've silently worked around.

- **Single availability zone**: all 3 nodes sit in `eu-north-1a`. This was a deliberate
  simplification for cost and complexity — a full AZ-redundant setup would need
  cross-AZ networking, a different storage class (since `local-path` is node-local and
  doesn't span AZs), and likely a managed load balancer. For a coursework capstone, the
  single-AZ trade-off keeps the infrastructure and PVC story simple, with the explicit
  understanding that an AZ outage would take down the whole cluster.
