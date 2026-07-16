# Runbook

All commands assume you're SSH'd into the k3s control-plane node and using
sudo k3s kubectl (or your own kubectl with the fetched kubeconfig).

## 1. Provision from zero

1. Run Terraform (infra/terraform/) to provision 3 EC2 nodes, network, and firewall.
   Note the outputs (public/private IPs).
2. Run the Ansible playbook (infra/ansible/) to install k3s server on the control-plane
   and k3s agent on both workers, using the node token from the server.
3. Confirm the cluster is up:
   sudo k3s kubectl get nodes -o wide
   All 3 nodes should show Ready.
4. Confirm metrics-server is running (needed for HPA later):
   sudo k3s kubectl get deployment metrics-server -n kube-system
   sudo k3s kubectl top nodes

## 2. Install the platform (one-time, not managed by GitOps)

cert-manager (for TLS):
   sudo k3s kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
Then apply your ClusterIssuer (letsencrypt-prod) pointing at Let's Encrypt.

Argo CD:
   sudo k3s kubectl create namespace argocd
   sudo k3s kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
Wait for all pods to be Ready:
   sudo k3s kubectl get pods -n argocd -w
Get the initial admin password (one-time, rotate after first login):
   sudo k3s kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

## 3. Create the real Secret (out-of-band, NOT in git)

The repo only contains a placeholder template at docs/templates/backend-secret.example.yaml.
Before the app can start, create the real Secret directly against the cluster:
   sudo k3s kubectl create namespace taskapp
   sudo k3s kubectl create secret generic backend-secret -n taskapp --from-literal=DATABASE_USER=REPLACE_ME --from-literal=DATABASE_PASSWORD=REPLACE_ME --from-literal=SECRET_KEY=REPLACE_ME
This Secret is intentionally excluded from manifests/, so Argo CD's automated sync will
never overwrite or delete it.

## 4. Point Argo CD at this repo

   sudo k3s kubectl apply -f gitops/application.yaml
Check status:
   sudo k3s kubectl get application taskapp -n argocd -o wide
Expect SYNC STATUS Synced, HEALTH STATUS Healthy. If it shows OutOfSync and doesn't
self-correct within a few minutes, force a sync:
   sudo k3s kubectl patch application taskapp -n argocd --type merge -p '{"operation":{"sync":{"revision":"HEAD","prune":true}}}'

## 5. Deploy a change (the only supported way, GitOps)

1. Edit the relevant file(s) under manifests/.
2. Commit and push to main:
   git add manifests/FILENAME.yaml
   git commit -m "describe the change"
   git push origin main
3. Argo CD will detect and sync automatically. To force it immediately:
   sudo k3s kubectl patch application taskapp -n argocd --type merge -p '{"operation":{"sync":{"revision":"HEAD","prune":true}}}'
4. Verify:
   sudo k3s kubectl get application taskapp -n argocd -o wide
   sudo k3s kubectl get pods -n taskapp -o wide

Never run kubectl apply or kubectl delete directly against resources in manifests/
for anything meant to persist. Argo's selfHeal will silently revert manual changes
back to whatever is in git.

## 6. Scale a workload

Edit replicas in the relevant Deployment file, commit, and push (see section 5).
Do not scale via kubectl scale directly; Argo will revert it.
See the HPA (manifests/backend-hpa.yaml, min 2 / max 5 replicas on 70% CPU) for
automatic backend scaling under load.

## 7. Roll back a bad deploy

1. Find the last known-good commit:
   git log --oneline -- manifests/
2. Revert the specific file to that commit, or use git revert BADCOMMITHASH.
3. Commit and push. Argo syncs the reverted state automatically, same as any other
   deploy (section 5).

## 8. Recover from: a dead worker node

This was tested live (see docs/EVIDENCE/). If a worker node fails or is drained:
   sudo k3s kubectl drain NODENAME --ignore-daemonsets --delete-emptydir-data --timeout=120s

Expected behavior: Pods on that node are evicted and rescheduled onto remaining nodes,
respecting each Deployment's PodDisruptionBudget (backend-pdb: minAvailable 1,
frontend-pdb: minAvailable 2) and anti-affinity rules. The app should remain reachable
throughout (verified: continuous /api/health polling stayed at 200 during a real drain).

Known limitation: if the drained node happens to host postgres-0 (the single Postgres
replica, no PDB), the database will be briefly unavailable until it reschedules. This is
a documented trade-off in ARCHITECTURE.md section 5, not a bug.

To restore the node:
   sudo k3s kubectl uncordon NODENAME

## 9. Recover from: a dead backend pod

Kubernetes handles this automatically. The Deployment's livenessProbe will restart a
pod that fails its /api/health check, and the ReplicaSet will recreate any pod that
dies outright. To verify current health:
   sudo k3s kubectl get pods -n taskapp -l app=backend
   sudo k3s kubectl describe pod -n taskapp -l app=backend

No manual intervention should be required. If pods are persistently crash-looping, check
logs first:
   sudo k3s kubectl logs -n taskapp -l app=backend --previous --tail=50

## 10. Recover from: a bad migration

Migrations run as a one-shot Job (backend-migration), not in the app's entrypoint.
Kubernetes Jobs are immutable once created. To re-run a migration, delete the old
Job first, then let Argo CD recreate it from the current git-defined spec:
   sudo k3s kubectl delete job backend-migration -n taskapp
   sudo k3s kubectl patch application taskapp -n argocd --type merge -p '{"operation":{"sync":{"revision":"HEAD","prune":true}}}'
   sudo k3s kubectl get jobs -n taskapp -w

Confirm the new Job completed successfully and used the correct pinned image tag:
   sudo k3s kubectl get job backend-migration -n taskapp -o jsonpath='{.spec.template.spec.containers[0].image}'

## 11. Verify data survived a Postgres restart

   sudo k3s kubectl exec -n taskapp postgres-0 -- psql -U taskapp -d taskapp -c "SELECT * FROM tasks;"
   sudo k3s kubectl exec -n taskapp postgres-0 -- psql -U taskapp -d taskapp -c "SELECT id, username FROM users;"

This should return the same rows before and after any pod restart, since data lives on
the PVC (local-path storage class), not in the container filesystem.

## 12. View Argo CD's UI

argocd-server is only exposed as a ClusterIP (not public) by design. To view it:
   sudo k3s kubectl port-forward svc/argocd-server -n argocd 8080:443 --address 0.0.0.0 &

Then, from your own machine, open an SSH tunnel to the control-plane:
   ssh -L 8080:localhost:8080 ubuntu@CONTROLPLANEPUBLICIP

Open https://localhost:8080 in your browser (accept the self-signed cert warning, this
is Argo's internal UI cert, unrelated to the app's real Let's Encrypt cert). Log in with
admin and the password from section 2.
