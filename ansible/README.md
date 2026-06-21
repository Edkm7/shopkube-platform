# Ansible

Ce dossier contient les playbooks et roles Ansible qui automatisent le provisioning du cluster Kubernetes. L'exécution complète installe les prérequis système, containerd, kubeadm, kubelet et kubectl sur les trois nœuds, initialise le control plane, déploie Calico et joint les workers au cluster.

## Prérequis

Depuis admin-vm (192.168.1.100), vérifier que les éléments suivants sont en place avant de lancer les playbooks :

- Ansible installé sur admin-vm
- Accès SSH depuis admin-vm vers les trois nœuds (controlplane-c1, worker1-c1, worker2-c1)
- Droits sudo sur les trois nœuds

## Configuration

Copier le fichier d'inventaire exemple et renseigner les IPs de vos nœuds :

```bash
cp inventory/hosts.example.yaml inventory/hosts.yaml
```

```yaml
# inventory/hosts.yaml
all:
  vars:
    ansible_user: <votre_utilisateur>
  children:
    kube_cluster:
      children:
        master:
          hosts:
            controlplane-c1:
              ansible_host: <ip_control_plane>
        workers:
          hosts:
            worker1-c1:
              ansible_host: <ip_worker1>
            worker2-c1:
              ansible_host: <ip_worker2>
```

Le fichier `ansible.cfg` configure les chemins vers les roles et l'inventaire. Vérifier que `remote_user` correspond à votre utilisateur sur les nœuds.

## Déployer le cluster

Depuis admin-vm (192.168.1.100) :

```bash
ansible-playbook playbooks/cluster.yml
```

Ce playbook exécute dans l'ordre :

1. Le role `common` sur tous les nœuds : mise à jour des paquets, modules kernel, sysctl, désactivation du swap
2. Le role `kubernetes` sur tous les nœuds : installation de containerd, kubeadm, kubelet et kubectl
3. Initialisation du control plane avec kubeadm et déploiement de Calico
4. Join des workers au cluster

## Réinitialiser le cluster

Si vous avez besoin de repartir de zéro, ce playbook remet les nœuds dans leur état initial.

Depuis admin-vm (192.168.1.100) :

```bash
ansible-playbook playbooks/reset.yml
```

Il exécute `kubeadm reset` sur chaque nœud, supprime les interfaces CNI, les fichiers de configuration Kubernetes et nettoie les règles iptables.

## Structure des roles

| Role | Rôle |
|---|---|
| `common` | Prérequis système (kernel modules, sysctl, swap) |
| `kubernetes` | Installation containerd, kubeadm, kubelet, kubectl, initialisation du cluster et join des workers |

## Variables

Les variables de version Kubernetes se trouvent dans `roles/kubernetes/vars/main.yml`. Pour changer la version du cluster, modifier ces deux valeurs :

```yaml
k8s_major_minor: "1.33"
k8s_version_full: "1.33.0-1.1"
```

## Auteur

_Eric, disponible sur [GitHub](https://github.com/Edkm7) et [Linkedin](https://www.linkedin.com/in/eric-dacier-8a3b2518a/)_
