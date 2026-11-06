# Day 01 — K3s Cluster Setup and Kubernetes Networking

**Date:** November 6, 2026  
**Project:** Project 1 — K3s Observability Platform  
**Environment:** Ubuntu 22.04.5 LTS  
**Status:** Completed initial Kubernetes setup and validation

---

## 1. Objective

Prepare the Ubuntu server as a single-node Kubernetes environment using K3s.

The goals for this session were:

- Verify the server's readiness for Kubernetes.
- Install and start K3s.
- Understand the Kubernetes cluster architecture.
- Validate system components and resource metrics.
- Test pod networking, DNS, and Kubernetes Service discovery.
- Document the results and identify the next steps.

## 2. Environment Details

| Component | Configuration |
|---|---|
| Operating System | Ubuntu 22.04.5 LTS |
| Kernel | 5.15.0-191-generic |
| Architecture | x86_64 |
| CPU | Intel Core i7-3632QM |
| Logical CPUs | 8 |
| RAM | 7.6 GiB |
| Swap | 4 GiB |
| Container runtime | containerd |
| Kubernetes distribution | K3s |
| K3s version | v1.37.2+k3s1 |
| Node name | ubuntu |
| LAN IP | 192.168.1.10 |
| Tailscale IP | 100.80.0.3 |

## 3. K3s Installation

Installed K3s using the official installation script:

```bash
curl -sfL https://get.k3s.io | sh -
```

Verified the service:

```bash
sudo systemctl status k3s --no-pager
```

**Result:** K3s service successfully started and is active.

Verified Kubernetes client and server versions:

```bash
sudo k3s kubectl version
```

**Result:**
- Client Version: v1.37.2+k3s1
- Server Version: v1.37.2+k3s1

Verified the Kubernetes node:

```bash
sudo k3s kubectl get nodes -o wide
```

**Result:** Node `ubuntu` reported `Ready`.

The node uses containerd as its container runtime.

## 4. Kubernetes System Components

Inspected all cluster pods:

```bash
sudo k3s kubectl get pods -A -o wide
```

The following system components reached their expected states:

| Component | Final status |
|---|---|
| CoreDNS | Running |
| Local Path Provisioner | Running |
| Metrics Server | Running |
| Traefik | Running |
| Traefik LoadBalancer pod | Running |
| Helm installation jobs | Completed |

The Helm installation jobs completed successfully. They are one-time installation jobs rather than permanent application pods.

## 5. Kubernetes Networking

Inspected network interfaces:

```bash
ip -br addr
```

Observed interfaces:

| Interface | Address | Purpose |
|---|---|---|
| wlp2s0 | 192.168.1.10 | Physical LAN/Wi-Fi |
| tailscale0 | 100.80.0.3 | Private Tailscale network |
| flannel.1 | 10.42.0.0/32 | Flannel overlay network |
| cni0 | 10.42.0.1/24 | Kubernetes pod network bridge |
| docker0 | 172.17.0.1/16 | Docker bridge; not used by K3s workloads |

Also observed virtual Ethernet (`veth`) interfaces created for pod networking.

### Network concepts learned

- Flannel provides the Kubernetes pod network.
- CNI connects pods to the host networking infrastructure.
- Pods receive their own IP addresses.
- Kubernetes Services provide stable virtual network addresses.
- CoreDNS resolves Kubernetes Service names.

## 6. Metrics Server Validation

Checked node resource usage:

```bash
sudo k3s kubectl top nodes
```

Observed:

| Metric | Value |
|---|---:|
| CPU usage | 147m |
| CPU percentage | 1% |
| Memory usage | 2195Mi |
| Memory percentage | 28% |

Checked pod resource usage:

```bash
sudo k3s kubectl top pods -A
```

**Result:** CPU and memory metrics were successfully returned for Kubernetes system pods.

### Concept learned

Kubernetes CPU is expressed in millicores:

- 1000m = 1 CPU core
- 147m = 0.147 CPU cores

Metrics Server provides resource metrics to Kubernetes. It is not a replacement for a full historical monitoring system such as Prometheus.

## 7. DNS Validation

Created a temporary diagnostic pod:

```bash
sudo k3s kubectl run netcheck \
  --image=busybox:1.36 \
  --restart=Never \
  --command -- sleep 3600
```

Tested Kubernetes DNS resolution:

```bash
sudo k3s kubectl exec netcheck -- \
  nslookup kubernetes.default.svc.cluster.local
```

**Observed result:**

```text
Server: 10.43.0.10

Name: kubernetes.default.svc.cluster.local
Address: 10.43.0.1
```

**Result:** The diagnostic pod successfully resolved the Kubernetes API Service through CoreDNS.

## 8. Deployment and Service Networking Test

Created a temporary Nginx Deployment:

```bash
sudo k3s kubectl create deployment web --image=nginx:alpine
```

Verified rollout:

```bash
sudo k3s kubectl rollout status deployment/web
```

**Result:** Deployment successfully rolled out.

Exposed the Deployment through a Kubernetes Service:

```bash
sudo k3s kubectl expose deployment web \
  --port=80 \
  --target-port=80
```

Inspected the pod and Service:

```bash
sudo k3s kubectl get pods -o wide
sudo k3s kubectl get svc web
```

Observed:

- Nginx pod IP: `10.42.0.11`
- Service ClusterIP: `10.43.83.141`
- Service port: `80/TCP`

Tested HTTP connectivity from the diagnostic pod:

```bash
sudo k3s kubectl exec netcheck -- \
  wget -qO- http://web.default.svc.cluster.local
```

**Result:** Received the Nginx welcome HTML page.

This validated:

1. DNS resolution of the Nginx Service.
2. Service-to-pod traffic routing.
3. Pod-to-pod communication.
4. HTTP response from the Nginx application.

## 9. EndpointSlice Validation

Inspected the Service endpoints:

```bash
sudo k3s kubectl get endpoints web
```

Observed:

```text
web    10.42.0.11:80
```

Also inspected the newer EndpointSlice API:

```bash
sudo k3s kubectl get endpointslices \
  -l kubernetes.io/service-name=web
```

Observed:

```text
web-5zxvx   IPv4   80   10.42.0.11
```

### Concept learned

A Kubernetes Service uses backend endpoints to direct traffic to eligible pods.

EndpointSlices are the newer API for representing Service backends. The older Endpoints API is deprecated in Kubernetes v1.33 and later.

## 10. Resource and Memory Assessment

Checked host memory:

```bash
free -h
```

Observed:

| Resource | Value |
|---|---:|
| Total RAM | 7.6 GiB |
| Used RAM | 1.7 GiB |
| Free RAM | 2.9 GiB |
| Buff/cache | 3.0 GiB |
| Available RAM | 5.6 GiB |
| Total swap | 4.0 GiB |
| Used swap | 0 B |

### Concept learned

Linux uses available memory for filesystem caching. The `available` value is a better indication of memory that can be used by new applications than the `free` value alone.

This memory assessment will guide the resource allocation for Prometheus, Grafana, Loki, Tempo, and the application services.

## 11. Firewall Configuration

UFW remains enabled.

Observed configuration:

- Default incoming policy: Deny
- Default outgoing policy: Allow
- Default routed policy: Deny

Relevant rules:

```text
22/tcp on tailscale0       ALLOW IN
22/tcp from 192.168.1.0/24 ALLOW IN
6443/tcp on tailscale0     ALLOW IN
```

The Kubernetes API port 6443 is permitted through the Tailscale interface.

**Pending validation:** Pod-to-pod, pod-to-Service, and other cluster networking paths have been tested for basic connectivity, but the complete firewall forwarding behavior has not yet been independently audited.

No broad Pod CIDR or Service CIDR firewall rules were added.

## 12. Temporary Resource Cleanup

Removed the temporary Nginx test resources:

```bash
sudo k3s kubectl delete service web
sudo k3s kubectl delete deployment web
sudo k3s kubectl delete pod netcheck
```

Verified cleanup:

```bash
sudo k3s kubectl get pods
sudo k3s kubectl get svc
```

**Result:**
- No application pods remain in the default namespace.
- No temporary Nginx Service remains.
- Kubernetes' built-in API Service remains as expected.

## 13. Issues and Observations

| Observation | Status |
|---|---|
| Initial pods showed ContainerCreating | Resolved; pods reached Running |
| Initial Metrics Server readiness errors | Resolved; metrics successfully returned |
| Initial CoreDNS readiness error | Resolved; DNS test passed |
| Initial InvalidDiskCapacity event | Observed during startup; no current disk pressure reported |
| PossibleMemoryBackedVolumesOnDisk warning | Documented; requires further security consideration |
| Deprecated Endpoints API warning | Understood; use EndpointSlices going forward |

No persistent cluster failure was identified during this session.

## 14. Key Concepts Learned

- K3s architecture and Kubernetes node readiness.
- containerd as a Kubernetes container runtime.
- CoreDNS and Kubernetes service discovery.
- Flannel, CNI, pod IPs, and virtual Ethernet interfaces.
- Metrics Server and Kubernetes resource metrics.
- Deployments and Services.
- ClusterIP and backend endpoints.
- EndpointSlices.
- Linux memory usage and available memory.
- Basic firewall considerations for Kubernetes networking.

## 15. Current Project Status

**Completed:**
- Ubuntu environment prepared.
- K3s installed and running.
- Single-node Kubernetes cluster is Ready.
- Core system pods are running.
- Metrics Server verified.
- Kubernetes DNS verified.
- Temporary Nginx Deployment and Service successfully tested.
- EndpointSlice inspected.
- Temporary resources cleaned up.
- Host memory baseline recorded.

**Not yet completed:**
- Python microservices.
- PostgreSQL deployment.
- Prometheus and Grafana.
- Node Exporter and PostgreSQL Exporter.
- OpenTelemetry instrumentation and Collector.
- Loki and Tempo.
- SLI/SLO dashboards and alerts.
- Load testing and failure drills.
- CI/CD validation.

## 16. Next Session Plan

1. Review the difference between Deployment and Service.
2. Create the first Python application in `app/service-a/`.
3. Build and test the application locally.
4. Create its Dockerfile and container image.
5. Deploy the application to K3s.
6. Expose it through a Kubernetes Service.
7. Validate the application endpoint from inside the cluster.

---

**Day 01 outcome:** Successfully installed and validated a single-node K3s cluster, including resource metrics, DNS, pod networking, Service discovery, and HTTP communication.

**Engineering principle:** Validate the infrastructure layer by layer before building applications on top of it.
