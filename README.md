**ShopKube Platform**

ShopKube est un projet personnel de plateforme Kubernetes construite sur des machines virtuelles bare-metal. L'objectif est de reproduire un environnement de production réaliste en partant de zéro : provisioning des nœuds, déploiement d'une application microservices, gestion des certificats, packaging et observabilité.

L'application déployée est [microservices-demo](https://github.com/GoogleCloudPlatform/microservices-demo), un projet open-source de Google Cloud Platform (Online Boutique, 12 microservices). Tous les droits sur le code applicatif appartiennent à Google. Ce repository contient uniquement la plateforme Kubernetes construite autour de cette application.

**Ce que fait ce projet**

Le cluster est composé de trois nœuds (un control plane et deux workers) provisionnés avec Ansible et kubeadm. Une fois le cluster en place, l'application est exposée en HTTPS via Traefik, avec des certificats signés par une CA interne gérée par HashiCorp Vault. Le stockage persistant est assuré par un serveur NFS via le CSI Driver. L'application est packagée en chart Helm avec plusieurs environnements (dev, prod). La stack de monitoring repose sur Prometheus, Grafana et Alertmanager.

**Stack technique**

 Cluster = Kubernetes v1.33, kubeadm, Ubuntu 22.04
 Provisioning = Ansible 
 CNI = Calico 
 Runtime = containerd 
 Load Balancer = MetalLB 
 Ingress = Traefik v3
 PKI et TLS =  HashiCorp Vault, cert-manager 
 Stockage = NFS CSI Driver 
 Packaging = Helm v3 
 Observabilite = Prometheus, Grafana, Alertmanager 
 GitOps = ArgoCD + GitLab CI (en cours)

**Structure du repo**


shopkube-platform/
├── ansible/          # Provisioning automatisé du cluster
├── infrastructure/   # MetalLB, Traefik, cert-manager, Vault, NFS CSI
├── helm/shopkube/    # Chart Helm complet des 12 microservices
├── monitoring/       # kube-prometheus-stack, Ingress Grafana/Prometheus
└── docs/             # Documentation par module


**Comment déployer**

**Prérequis**

Trois VMs Ubuntu 22.04, Ansible installé sur la machine admin, Helm v3 et un accès SSH aux nœuds.

**Provisionner le cluster**


cd ansible
# Renseigner les IPs dans hosts.yaml
cp inventory/hosts.example.yaml inventory/hosts.yaml
# Déployer le cluster kube
ansible-playbook playbooks/cluster.yml 

#Déployer ShopKube en production
helm install shopkube-prod ./helm/shopkube \
  -f helm/shopkube/values-prod.yaml \
  -n shopkube-prod --create-namespace

**Installer le monitoring**

helm install monitoring prometheus-community/kube-prometheus-stack \
  -f monitoring/prometheus-values.yaml \
  -n monitoring --create-namespace

**Points notables**

Le scheduling est configuré pour isoler les workloads critiques (redis, checkout, payment) sur worker1 via des taints et tolerations. Le frontend est contraint sur worker2 via une node affinity. Le loadgenerator ne peut pas se retrouver sur le même nœud que le frontend grâce à une pod anti-affinity.

La PKI est gérée par Vault comme autorité de certification interne. Les certificats sont émis et renouvelés automatiquement par cert-manager via le Kubernetes auth method, sans token statique à gérer.

Le chart Helm supporte plusieurs environnements depuis un seul jeu de templates. Les contraintes de scheduling, le nombre de replicas et le hostname Ingress varient selon le fichier de values chargé.

Le monitoring est isolé de la production sur sa propre instance Traefik avec une IP dédiée, accessible via FQDN en HTTPS.

* Roadmap

- [x] Provisioning cluster avec Ansible
- [x] Stockage persistant NFS CSI
- [x] Scheduling avancé (Taints, Affinity, PriorityClass)
- [x] Ingress TLS avec Vault PKI et cert-manager
- [x] Chart Helm multi-environnement
- [x] Observabilite Prometheus, Grafana, Alertmanager
- [ ] Alerting (PrometheusRule)
- [ ] GitOps ArgoCD avec GitLab CI
- [ ] Autoscaling HPA et VPA
- [ ] Network Policies et RBAC
- [ ] Logs avec Loki

**Auteur**

Eric, disponible sur [GitHub](https://github.com/Edkm7) et https://www.linkedin.com/in/eric-dacier-8a3b2518a/
