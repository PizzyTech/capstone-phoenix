# Cost

## Monthly itemized cost

| Item | Spec | Qty | $/mo (est.) |
|---|---|---:|---:|
| Control-plane VM | t3.small (2 vCPU, 2GB RAM), eu-north-1 | 1 | ~$17 |
| Worker VMs | t3.small (2 vCPU, 2GB RAM), eu-north-1 | 2 | ~$34 |
| Block storage (PVC) | local-path on root EBS volume, 2GB used | included | ~$0 extra |
| Elastic IP / load balancer | none, using the control-plane's public IP directly, no ELB | - | $0 |
| DNS / domain | nip.io wildcard DNS (free, no registrar cost) | - | $0 |
| Data transfer out | first 100GB/mo free tier, low traffic for a coursework demo | - | ~$0-2 |
| Total | | | ~$51-53/mo |

Note on accuracy: these are estimates based on published AWS on-demand rates for
t3.small in eu-north-1 (Stockholm) at the time of writing. The actual AWS Cost
Explorer or billing console for this account is the authoritative source and should be
checked directly before relying on this figure for any real decision.

## Compared to the single-server Compose+Portainer deploy

- That stack (from the earlier Docker/Portainer lesson): 1 EC2 instance, similar spec
  (t3.small or comparable), roughly $17-20/month. No redundancy, no autoscaling, no
  automatic failover.
- This cluster: ~$51-53/month, roughly 3x the cost of the single-server setup.
- What the extra spend buys: high availability (the app survived a real worker node
  drain with zero dropped requests, verified live), horizontal autoscaling under load
  (HPA configured on the backend), zero-downtime rolling deploys (verified: continuous
  health-check polling stayed at 200 through an actual rollout and an actual node
  failover), and GitOps-driven change management (every change is auditable in git history,
  with automatic drift correction via Argo CD's self-heal).
- When it's not worth it: for a genuinely low-traffic personal project, an internal
  tool with a handful of users, or anything where a few minutes of downtime during a
  restart is truly a non-event, the single-server setup is the right call. 3x the
  infrastructure cost for redundancy nobody will notice is money better spent elsewhere.
  The crossover point is roughly: once downtime has a real cost (lost signups, lost
  revenue, SLA obligations, or simply enough concurrent users that a restart is
  noticeable), the cluster's cost becomes easy to justify.

## How I'd halve this

The single biggest lever is dropping to 2 nodes instead of 3 by collapsing the
control-plane and one worker's role, but the brief explicitly requires 3 real nodes with
genuine multi-node scheduling, so that's off the table for meeting Core requirements.
Within the 3-node constraint, the most realistic savings are: (1) switching all 3 nodes to
Spot Instances instead of On-Demand, which for t3.small typically runs 60-70%
cheaper, appropriate here since this is a coursework/demo workload that can tolerate
occasional Spot interruption (and would even double as an interesting extra failover test);
(2) using a Savings Plan or 1-year Reserved pricing if this were a longer-lived
deployment rather than a 3-week capstone, cutting on-demand rates by roughly 30-40%; and
(3) confirming the root EBS volumes are sized minimally (the current 2GB Postgres PVC and
default OS disk footprint are already small, but oversized default AMI root volumes are a
common hidden cost worth checking). Combining Spot pricing with right-sized storage alone
would likely bring the monthly total from ~$51-53 down to somewhere in the $20-25 range.
