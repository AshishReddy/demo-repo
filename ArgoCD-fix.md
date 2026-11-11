# ArgoCD DNS Troubleshooting on SNO 

The core reason is a Go runtime DNS resolver mismatch inside the Argo CD container binaries, combined with how OpenShift's ndots:5 search path behaves on a local laptop network.
Even though standard Linux tools (curl, git, nslookup) inside the pod resolve github.com properly via the system C-library (glibc), the compiled Go binaries running Argo CD (argocd-repo-server, argocd-server) bypass standard system calls and hit a DNS query failure.

1. Go's Built-In DNS Resolver vs. glibc

• Why curl and git worked: Standard command-line utilities use the container's OS C-library (glibc) to read /etc/resolv.conf. They handle local router quirks, slow responses, or IPv6 fallbacks gracefully.
• Why Argo CD failed: Argo CD is written in Go. On Alpine/UBI minimal containers, Go often uses its own pure-Go internal DNS resolver instead of the system resolver. This internal resolver sends concurrent A (IPv4) and AAAA (IPv6) queries directly to 172.30.0.10 (CoreDNS). If your laptop's virtual network or home router drops IPv6 queries or responds slowly to dual queries, Go's resolver times out or throws a no such host error—even while curl succeeds.

2. The Kubernetes ndots:5 Search Domain Flood

In OpenShift, every pod's /etc/resolv.conf defaults to options ndots:5.
• Because github.com only has one dot (fewer than 5), the Go resolver does not treat it as a complete domain name at first.
• Instead, it appends every internal cluster search domain before trying the real internet address (e.g., github.com.openshift-gitops.svc.cluster.local, github.com.svc.cluster.local, github.com.cluster.local, github.com.openshift.local.lan).
• On a local laptop SNO setup (like VMware/VirtualBox/libvirt), your host hypervisor's NAT DNS forwarder (dnsmasq or home router) often chokes on this rapid burst of NXDOMAIN requests or rate-limits UDP port 53, causing Go's strict lookup timer to fail.

3. Why hostAliases Fixed It Completely

When you patched hostAliases onto the deployments, Kubernetes injected 20.207.73.82 github.com directly into /etc/hosts inside those pods. Both glibc and Go's internal DNS resolver always check /etc/hosts before sending any UDP packets to CoreDNS (172.30.0.10), completely bypassing the ndots:5 search flood and IPv6/NAT resolver quirks.

How to Fix It Permanently Without Hardcoding IPs

'''
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
'''

Since GitHub's IP addresses occasionally rotate, hardcoding 20.207.73.82 can eventually break. You can force Go to behave properly or stop the ndots flood cleanly on your SNO cluster:
• Option A: Force Argo CD to use the OS (glibc) DNS resolver
Add the GODEBUG=netdns=cgo environment variable to the Argo CD custom resource so Go stops using its picky internal resolver and uses the exact same DNS path as curl and git:bash
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
Use code with caution.
• Option B: Lower ndots on the Argo CD pods
Add a dnsConfig to your deployments to set ndots: "2", stopping the cluster from spamming your laptop's virtual router with internal search domains for external websites.
Would you like to test removing the hardcoded hostAliases and applying the GODEBUG=netdns=cgo patch so your cluster automatically tracks GitHub's IP changes going forward?

