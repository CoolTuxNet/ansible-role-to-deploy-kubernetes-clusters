```shell
helm repo add cilium https://helm.cilium.io/
helm repo update

helm upgrade --install cilium cilium/cilium --version 1.19.4 --namespace kube-system --set kubeProxyReplacement=true --set k8sServiceHost=10.6.8.210 --set k8sServicePort=6443 --set ipam.mode=kubernetes --set l2announcements.enabled=true --set gatewayAPI.enabled=true
```

```yaml
---
apiVersion: "cilium.io/v2alpha1"
kind: CiliumLoadBalancerIPPool
metadata:
  name: gateway-ippool
spec:
  # Du kannst hier CIDR-Blöcke definieren, aus denen Cilium IPs bezieht.
  # Wichtig: Diese IPs müssen in deinem lokalen Netzwerk erreichbar sein, 
  # dürfen aber NICHT von deinem Router/DHCP-Server vergeben werden!
  blocks:
    # Ein kleines Subnetz (z.B. 4 IPs)
    - cidr: "192.168.178.200/30" 
    
    # Eine völlig freie Range (11 IPs)
    - start: "192.168.178.210"
      stop: "192.168.178.220"
      
    # Eine spezifische, einzelne IP
    - cidr: "192.168.178.240/32"
---
apiVersion: "cilium.io/v2alpha1"
kind: CiliumL2AnnouncementPolicy
metadata:
  name: l2-announcement-policy
spec:
  # Legt fest, welche Nodes die LoadBalancer-IPs im Netzwerk announcen dürfen.
  # Ein leeres matchLabels (oder Weglassen von nodeSelector) wählt alle Nodes aus.
  nodeSelector:
    matchExpressions:
      - key: node-role.kubernetes.io/control-plane
        operator: DoesNotExist # Optional: Lässt nur Worker-Nodes den Traffic annehmen

  # Optional: Spezifiziert das Netzwerk-Interface auf den Nodes.
  # Reguläre Ausdrücke sind erlaubt. Bei Debian-VMs ist es oft "ens18" oder "eth0".
  interfaces:
    - ^ens[0-9]+
    - ^eth[0-9]+

  # Muss aktiviert sein, damit Services vom Typ LoadBalancer announced werden
  loadBalancerIPs: true
  
  # Optional, falls du auch ExternalIPs announcen möchtest
  externalIPs: true
```

```shell
kubectl -n cilium-system rollout restart deployment/cilium-operator
kubectl -n cilium-system rollout restart ds/cilium
```