---
title: "Condor Enphase, v2: From Pi → Kubernetes (with Tailscale & Codespaces)"
date: 2025-09-15
tags: [enphase, kubernetes, tailscale, grafana, influxdb, codespaces, claude-code]
description: "Turning the Raspberry Pi prototype into a cloud‑native deployment on Kubernetes using Tailscale for zero‑trust networking and GitHub Codespaces + Claude Code for dev."
---
In [the last post](https://sftechguy.com/2025/08/16/Condor_Enphase.html) I
proved the idea on a Raspberry Pi with Docker Compose, InfluxDB, and
Grafana—plus Tailscale for secure remote access. This post is the story of
turning that prototype into a cloud-native deployment on **Kubernetes**,
developed end-to-end in **GitHub Codespaces** with **Claude Code** as my
coding co-pilot.

The open-source repo continues to evolve here:
**[byronicle/condor-enphase](https://github.com/byronicle/condor-enphase)**.

---

## Why Kubernetes for this project?
This project demonstrates hosting an application from end-to-end on Kubernetes. 
Everything from the python application on docker to building with Terrafrom 
and actuating the environment on Google Cloud using GitHub Actions.

---

## Dev environment: GitHub Codespaces + Claude Code

I built and iterated directly in Codespaces. I used Claude Code for initial 
iteration and development of Terraform. Using Claude Code saved countless 
hours looking the Terraform provider and writing the rersouces. This allowed 
me to focus more on troubleshooting what was written and directing Claude to 
refactor sections based on deployment errors and making sure it was not 
hallucinating by directing Claude to correct and current documentation.

### Being a good AI Supervisor
In order to use and direct Claude effeciently I had to come up with the base 
architecture: host in Kubernetes and use Tailscale as the VPN. Once I knew 
what I needed, it was straightforward to ask Claude what to do. 
Claude was effecient at creating and structuring terraform resources for 
Google Cloud, but needed help to make sure everything worked correctly. For 
example, in Claude's first iteration, it was able to make the deployments 
quickly, but did not integrate Tailscale correctly. I had to direct 
Claude the specfic 
[Tailscale documentation](https://tailscale.com/kb/1438/kubernetes-operator-cluster-egress#access-an-ip-address-behind-a-subnet-router) 
and ask it to implement that. 

---

## System Overview (K8s edition)

**Core components**  
- **Ingestor** (Python): polls Enphase Cloud API and writes time-series points.  
- **InfluxDB v2**: long-term storage.  
- **Grafana**: dashboards and alerts.  
- **Tailscale Operator**: secure access (ingress) to dashboards; optional 
egress to private tailnet services.

**Kubernetes primitives**  
- **Namespace**: `condor-enphase`  
- **Deployments/StatefulSets**:  
  - `ingestor` (Deployment)  
  - `influxdb` (StatefulSet + PVC)  
  - `grafana` (Deployment + PVC)  
- **Config & Secrets**: Enphase credentials, Influx token stored as K8s 
`Secret` (or cloud secret manager + CSI driver).  
- **Networking via Tailscale**:  
    - **Ingress**: Tailscale IngressClass or LoadBalancerClass to reach Grafana
      over your tailnet. Port 443 only, Prefix path type.  
    - **Egress (optional)**: ExternalName `Service` with Tailscale annotations
      to reach any device/IP in your tailnet (e.g., if you still hit a local
      Envoy or a TS-only service).

**High-level diagram**

```
+----------------------------- Kubernetes Cluster -----------------------------+
| Namespace: condor-enphase                                                    |
|                                                                              |
|  [Deployment] ingestor  --->  [Service] influxdb  --->  [StatefulSet] Influx |
|        |                                ^                                    |
|        |                                |                                    |
|        +---->  [Service] grafana  <-----+----  [Deployment] Grafana          |
|                        ^                                                     |
|                        |  (Tailscale Ingress or LoadBalancer)                |
|          +-------------+-------------+                                       |
|          |    Tailscale Operator     |  -> creates proxy Pods / CRDs         |
|          +-------------+-------------+                                       |
|                        |                                                     |
|             (tailnet: secure access)                                         |
+------------------------------------------------------------------------------+
```

---

## Network Overview with the Tailscale Operator


The cluster never exposes a public LoadBalancer. All operator‑assisted
networking rides the tailnet. Two flows matter: user ingress (you → Grafana)
and device egress (ingestor → Envoy on the home LAN).

**Ingress (user → Grafana)**
- `helm_release.tailscale_operator` installs the operator (namespace
  `tailscale`). Once running, any `Service` annotated with
  `tailscale.com/expose=true` becomes reachable on your tailnet as its
  assigned ephemeral DNS name / the explicit hostname you set.
- The Grafana `Service` (`kubernetes_service.grafana`) carries:
  - `tailscale.com/expose: "true"` → tell operator to create a proxy Pod / Funnel
  - `tailscale.com/hostname: grafana-k8s-cluster` → stable tailnet DNS name
  - `tailscale.com/tags: tag:k8s` → apply ACL tag scoping instead of user key.
- No LoadBalancer object is created; the operator sidecar handles the TCP
  listener inside the tailnet. Access pattern:
  `https://grafana-k8s-cluster.tailnet-YOURORG.ts.net` (or just the short
  magic DNS name inside devices already on the tailnet). TLS comes from your
  local trust if you enable Funnel / certs later; for now I accept the default
  (Grafana still speaks HTTP internally; you can front it with Tailscale HTTPS
  if you enable Funnel in the admin console).

**Egress (ingestor → home Envoy)**
- Requirement: the Python ingestor needs to talk to the *local* Envoy at a
  192.168.x.y RFC1918 address that lives behind a *subnet router* already
  advertising that route into the tailnet.
- Rather than baking Tailscale into the ingestor Pod, I let the operator
  supply routing by creating a synthetic `Service` named `home-network`.
- Terraform resource `kubernetes_service.home_network_egress`:
  - Type `ExternalName` (no cluster IP, just DNS indirection)
  - Annotation `tailscale.com/tailnet-ip = var.envoy_host` is the IP of the
    Envoy as seen on the *home LAN* (not a tailnet 100.x address). The operator
    creates a proxy Pod that can reach that IP over the tailnet via the
    subnet router and exposes it *inside the cluster* at a stable Pod IP.
  - Annotation `tailscale.com/proxy-class = accept-routes` ties to the
    `ProxyClass` CRD `accept-routes` (`kubernetes_manifest.accept_routes_proxy_class`)
    whose spec enables `acceptRoutes=true`, allowing the proxy Pod to accept
    advertised subnets from the router.
- The ingestor Deployment sets `ENVOY_HOST = home-network.enphase.svc.cluster.local`.
  So a normal cluster‑internal DNS lookup hits the operator’s proxy Pod which
  then forwards over WireGuard to the subnet router, onto the physical Envoy.

**Why an ExternalName?**
- This pattern avoids managing headless Services or additional sidecars.
- We do not rely on kube‑DNS for an IP that might change; the operator owns
  the lifecycle of the proxy and backends.

**Data path summary**
```
User (tailnet) → Tailscale operator proxy (Grafana svc) → Grafana Pod

Ingestor Pod → DNS: home-network.enphase.svc → Tailscale proxy Pod
  → tailnet WireGuard → Subnet Router → 192.168.x.y (Envoy) → metrics JSON
```

**Security controls**
- Tailnet ACLs + `tag:k8s` tag govern who can reach Grafana.
- No public IPs; GKE cluster firewall only allows egress.
- Secrets for tokens / passwords are standard K8s `Secret`s (optionally swap
  to Secret Manager CSI later).

**Operational wins**
- Zero cloud load balancers (cost + attack surface).
- Stable DNS naming on tailnet without extra cert management.
- Clean separation of *app containers* and *connectivity fabric*.

---

## Tailscale Operator: the parts that were tricky (and how I fixed them)

**1. Picking the right exposure model (Ingress vs sidecar vs proxy Service)**  
Early on I considered running a Tailscale sidecar in every Pod (Grafana,
ingestor, etc.). It works, but multiplies key management and makes ACL
reasoning harder. The operator’s `tailscale.com/expose` pattern centralizes
that and let me drop per‑pod auth keys. Decision: one operator + annotated
Services instead of per‑pod sidecars.

**2. Subnet routing wasn’t automatic**  
Even after the operator was up, the ingestor couldn’t reach the Envoy’s
192.168.x.y address. Root cause: the default proxy Pods don’t accept
advertised routes. Fix: create a `ProxyClass` with `acceptRoutes=true`
(`kubernetes_manifest.accept_routes_proxy_class`) and reference it via
`tailscale.com/proxy-class: accept-routes` on the egress Service. After that
the proxy Pod picked up the LAN route the home subnet router was already
advertising in the tailnet admin panel.

**3. ExternalName + tailnet-ip annotation subtlety**  
Documentation examples mostly show exposing in‑cluster workloads. I needed
the reverse: reach *out* to a LAN device. Using an `ExternalName` Service and
setting `tailscale.com/tailnet-ip` to the LAN IP looks odd (it’s not a
tailnet 100.x address). Internally the operator still launches a proxy Pod;
that Pod uses its WireGuard interface to talk to the LAN via subnet routing.
Kube‑DNS just needs a stable name; the underlying CNAME target value is
irrelevant because the operator intercepts. Terraform keeps it reproducible.

**4. Dependency ordering in Terraform**  
If you create the Service before the operator CRDs exist, the annotations are
ignored until a reconcile loop later. I added explicit `depends_on` chains:
Service → operator Helm release → ProxyClass. That removed a flaky first
apply where the proxy Pod occasionally never materialized.

**5. Secret & token sprawl avoidance**  
I started with a legacy `ts_authkey`. Migrated to OAuth client ID/secret
(`tailscale_oauth_client_*`) because the operator prefers that and it lets me
bind ACL tags cleanly. The legacy key is still in the secret for now but will
be removed once I confirm no fallback paths rely on it.

**6. DNS target inside application code**  
Hard‑coding the literal Envoy IP in the ingestor would defeat the purpose of
abstracting via tailnet routing (and break if it ever changes). Setting
`ENVOY_HOST = home-network.<ns>.svc.cluster.local` (in Terraform env vars)
keeps the Python layer oblivious to networking topology changes.

**7. Debugging toolkit**  
- `kubectl get pods -n tailscale` to watch for the proxy Pod that matches the
  Service.
- `kubectl exec` into that Pod and `ping` / `curl` the 192.168.x.y address.
- Tailnet admin panel: confirm the proxy node shows the right tags & routes.
- If routes missing: check subnet router device is still advertising.

**8. Performance considerations (so far fine)**  
Single hop WireGuard latency was negligible relative to Envoy poll interval
(60s). If I ever push live streaming at higher frequency I may benchmark a
direct sidecar approach, but for now operator proxy is simpler.

**9. Future improvements**  
- Move K8s secrets → GCP Secret Manager + CSI driver
- Enable Funnel + TLS for Grafana externally (if I ever want to share a link)
- Add NetworkPolicies (once I enable CNI supporting them) to scope egress
- Replace legacy auth key entirely; rotate OAuth client
- Possibly add a second Tailscale proxy for a different home subnet

## Summary  
I hope you found this insightful--it really helped me develop my end-to-end 
application hosting skills. 
