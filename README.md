# Ansible-Rollen für Kubernetes-Deployments (kubeadm & RKE2)

Dieses Repository enthält Ansible-Rollen zur automatisierten Bereitstellung von hochverfügbaren (HA) Kubernetes-Clustern. Es werden zwei verschiedene Distributionen unterstützt:

- **kubeadm**: Der Standard-Weg, um Kubernetes-Cluster zu erstellen und zu verwalten.
- **RKE2**: Eine von Rancher entwickelte, sicherheitsgehärtete Kubernetes-Distribution.

## Übersicht

Die Rollen sind so strukturiert, dass sie je nach gewählter Distribution die entsprechenden Knoten konfigurieren und dem Cluster hinzufügen.

## Voraussetzungen

- Ansible ist auf dem Control-Host installiert.
- Ein korrekt konfiguriertes Ansible-Inventar, das die Zielknoten in Gruppen einteilt.
- Eine virtuelle IP (VIP), die als stabiler Endpunkt für die Kubernetes-API dient. Diese wird von den Control-Plane-Knoten gemeinsam genutzt.

## Konfiguration

Die Konfiguration ist auf mehrere `group_vars`-Dateien aufgeteilt.

### Allgemeine Konfiguration (`group_vars/all.yml`)

Hier werden Variablen definiert, die für beide Cluster-Typen gelten:
- `pod_network_cidr` / `service_network_cidr`: Die IP-Bereiche für Pods und Services.
- `kube_vip_address` / `kube_vip_interface`: Konfiguration für Kube-VIP, um den HA-API-Endpoint bereitzustellen.
- `cni_plugin`: Definiert das zu installierende Netzwerk-Plugin (z.B. `cilium`).

---

## 1. Deployment mit `kubeadm`

Dieses Setup verwendet `kubeadm`, um einen Cluster zu initialisieren und zu erweitern.

### `kubeadm` Konfiguration (`group_vars/kubeadm_cluster.yml`)

- `k8s_version`: Die exakte Kubernetes-Version, die installiert werden soll (z.B. `"1.35.7"`).
- `control_plane_endpoint`: Der gemeinsame Endpunkt für die API-Server, der auf die VIP zeigt.

### `kubeadm` Rollen

#### `control_plane_init`
Initialisiert den ersten Control-Plane-Knoten.
1.  **Prüfung**: Stellt sicher, dass der Cluster nicht bereits initialisiert ist (`/etc/kubernetes/admin.conf`).
2.  **Konfigurations-Templating**: Erstellt eine `kubeadm-init.yml` mit den Cluster-Parametern.
3.  **Cluster-Initialisierung**: Führt `kubeadm init --upload-certs` aus. Der `--upload-certs`-Flag ist entscheidend für den HA-Betrieb.
4.  **Join-Befehle extrahieren**: Generiert Join-Befehle für Worker und weitere Control-Plane-Knoten (inkl. `--certificate-key`).
5.  **Befehle speichern**: Speichert die Join-Befehle als Fakten im `K8S_TOKEN_HOLDER`-Host, um sie für andere Knoten verfügbar zu machen.
6.  **`super-admin.conf` sichern**: Holt die `super-admin.conf` auf den Ansible-Controller. Diese wird von den anderen Control-Plane-Knoten für den Join-Prozess benötigt.

#### `control_plane_join`
Fügt weitere Control-Plane-Knoten hinzu.
1.  **Prüfung**: Stellt sicher, dass der Knoten nicht bereits Teil des Clusters ist.
2.  **`super-admin.conf` bereitstellen**: Kopiert die zuvor gesicherte `super-admin.conf` vom Ansible-Controller auf den neuen Knoten.
3.  **Cluster-Beitritt**: Führt den Join-Befehl aus, der vom `K8S_TOKEN_HOLDER`-Host bezogen wird.
4.  **`kubectl`-Konfiguration**: Richtet die `kubeconfig` für den `root`-Benutzer ein.

### `kubeadm` Beispiel-Workflow

**Inventar-Beispiel (`inventory.ini`):**
```ini
[kube_control_plane_initial]
master1.example.com
[kube_control_plane_join]
master2.example.com
master3.example.com
[all:vars]
ansible_user=your_ssh_user
kube_vip_address=192.168.0.10
```
**Playbook-Beispiel (`playbook-kubeadm.yml`):**
```yaml
- name: Initialize the first Kubernetes control plane node
  hosts: kube_control_plane_initial
  become: yes
  roles:
    - role: kubeadm_kubernetes/control_plane_init
- name: Join additional Kubernetes control plane nodes
  hosts: kube_control_plane_join
  become: yes
  roles:
    - role: kubeadm_kubernetes/control_plane_join
```

**Inventar-Beispiel:**

```ini
[kube_control_plane_initial]
master1.example.com

[kube_control_plane_join]
master2.example.com
master3.example.com

[all:vars]
ansible_user=your_ssh_user
kube_vip_address=192.168.0.10
```

### Ausführungsreihenfolge

1.  Führen Sie das Playbook zuerst nur für die `kube_control_plane_initial`-Gruppe aus. Dies kann mit `--limit` geschehen:
    ```bash
    ansible-playbook playbook.yml --limit kube_control_plane_initial
    ```
2.  Nachdem der erste Knoten erfolgreich initialisiert wurde, führen Sie das Playbook für die `kube_control_plane_join`-Gruppe aus:
    ```bash
    ansible-playbook playbook.yml --limit kube_control_plane_join
    ```

Dieser zweistufige Prozess stellt sicher, dass die Join-Token und Zertifikatsschlüssel vom ersten Knoten generiert werden, bevor die anderen Knoten versuchen, dem Cluster beizutreten.