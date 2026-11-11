# Troubleshooting Argo CD GitHub DNS Resolution Issues on Single Node OpenShift (SNO)

## Overview

When running **Red Hat OpenShift GitOps (Argo CD)** on a local **Single Node OpenShift (SNO)** cluster (such as on a laptop using VMware, VirtualBox, or libvirt), Argo CD pods often fail to pull or sync public GitHub repositories with a `no such host` or DNS lookup timeout error.

Interestingly, running standard networking commands like `curl github.com`, `git clone`, or `nslookup` from inside the `openshift-gitops-repo-server` pod works without issue. 

This guide explains the root cause behind this discrepancy and provides both an immediate workaround and a permanent fix.

---

## Root Cause Analysis

### 1. Go's Built-in DNS Resolver vs. System `glibc`
* **Why `curl` and `git` succeed:** Standard Linux utilities use the container's OS C-library (`glibc`) to resolve domains via `/etc/resolv.conf`. They handle local router quirks, slow responses, and IPv6 fallbacks gracefully.
* **Why Argo CD fails:** Argo CD components (`argocd-repo-server` and `argocd-server`) are compiled **Go** binaries. In minimal container images (UBI/Alpine), Go defaults to its pure-Go internal DNS resolver instead of calling the system resolver. This internal resolver sends concurrent `A` (IPv4) and `AAAA` (IPv6) DNS queries directly to CoreDNS (`172.30.0.10`). If your laptop's hypervisor NAT network or home router drops IPv6 queries or responds slowly to dual requests, Go's strict lookup timer fails.

### 2. Kubernetes `ndots:5` Search Domain Flood
By default, OpenShift configures pod `/etc/resolv.conf` files with `options ndots:5`. 
* Because `github.com` contains only **one dot** (fewer than 5), Go's resolver does not treat it as a fully qualified domain name (FQDN) initially.
* Instead, it appends every internal cluster search domain before querying the external internet (e.g., `github.com.openshift-gitops.svc.cluster.local`, `github.com.svc.cluster.local`, `github.com.cluster.local`).
* Local laptop NAT DNS forwarders (`dnsmasq` or consumer routers) frequently choke on or rate-limit this rapid burst of `NXDOMAIN` lookups over UDP port 53.

### 3. Why `hostAliases` Resolves the Issue
Injecting `hostAliases` into the pod specifications adds a static entry directly into `/etc/hosts` inside each container. Both `glibc` and Go's internal DNS resolver read `/etc/hosts` **before** sending any queries to CoreDNS, completely bypassing the `ndots:5` search flood and IPv6/NAT timeout issues.

---

## Solution 1: Immediate Workaround (`hostAliases`)

You can inject GitHub's IP address directly into the `openshift-gitops` workloads using `oc patch`. Because the OpenShift GitOps Operator preserves user-defined `hostAliases`, these patches will persist across pod restarts.

> **Note:** Verify the current IP address for `github.com` using `nslookup github.com` before running these commands.

```bash
# 1. Patch the Repo Server (handles Git fetching)
oc patch deployment/openshift-gitops-repo-server -n openshift-gitops --type='json' \
  -p='[{"op": "add", "path": "/spec/template/spec/hostAliases", "value": [{"ip": "20.207.73.82", "hostnames": ["github.com"]}]}]'

# 2. Patch the API Server UI
oc patch deployment/openshift-gitops-server -n openshift-gitops --type='json' \
  -p='[{"op": "add", "path": "/spec/template/spec/hostAliases", "value": [{"ip": "20.207.73.82", "hostnames": ["github.com"]}]}]'

# 3. Patch the Application Controller (manages sync states)
oc patch statefulset/openshift-gitops-application-controller -n openshift-gitops --type='json' \
  -p='[{"op": "add", "path": "/spec/template/spec/hostAliases", "value": [{"ip": "20.207.73.82", "hostnames": ["github.com"]}]}]'

# 4. Patch the Redis Cache
oc patch deployment/openshift-gitops-redis -n openshift-gitops --type='json' \
  -p='[{"op": "add", "path": "/spec/template/spec/hostAliases", "value": [{"ip": "20.207.73.82", "hostnames": ["github.com"]}]}]'
```

---

## Solution 2: Permanent Fix (`GODEBUG=netdns=cgo`)

Because GitHub's IP addresses can rotate over time, hardcoding IPs via `hostAliases` is best used as a temporary measure. 

To fix DNS resolution permanently without hardcoding IPs, instruct the Argo CD Go binaries to use the OS system resolver (`cgo`) instead of Go's internal DNS resolver:

```bash
oc patch argocd openshift-gitops -n openshift-gitops --type merge -p '
spec:
  repo:
    env:
    - name: GODEBUG
      value: "netdns=cgo"
  server:
    env:
    - name: GODEBUG
      value: "netdns=cgo"
'
```

Once the pods finish rolling out, Argo CD will resolve `github.com` dynamically using the exact same reliable path as `curl` and `git`.

