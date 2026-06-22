# Monitoring

Ce dossier contient la configuration de la stack d'observabilité basée sur kube-prometheus-stack. Elle déploie Prometheus, Grafana et Alertmanager dans le namespace `monitoring`, avec un accès HTTPS via des FQDN dédiés.

## Architecture d'accès

Le monitoring est isolé de la production sur sa propre instance Traefik avec une IP dédiée assignée par MetalLB. Ce choix garantit que si le Traefik de production tombe, l'accès aux outils de monitoring reste disponible.

| Service | URL | IP |
|---|---|---|
| Grafana | https://grafana.shopkube.local | 192.168.1.202 |
| Prometheus | https://prometheus.shopkube.local | 192.168.1.202 |
| Alertmanager | https://alertmanager.shopkube.local | 192.168.1.202 |

Les certificats TLS sont signés par Vault et gérés automatiquement par cert-manager.

## Installation

Depuis admin-vm (192.168.1.100) :

```bash
# Ajouter le repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Installer la stack
helm install monitoring prometheus-community/kube-prometheus-stack \
  -f monitoring/prometheus-values.yaml \
  -n monitoring --create-namespace

# Installer l'instance Traefik dédiée au monitoring
helm install traefik-monitoring traefik/traefik \
  --namespace monitoring \
  --set service.type=LoadBalancer \
  --set providers.kubernetesIngress.ingressClass=traefik-monitoring

# Appliquer les Ingress
kubectl apply -f monitoring/ingress/ingress-monitoring.yaml
```

## Certificats TLS

Les certificats sont générés par cert-manager via Vault. Les créer avant d'appliquer les Ingress.

Depuis admin-vm (192.168.1.100) :

```bash
kubectl apply -n monitoring -f - <<EOF
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: grafana-tls
spec:
  secretName: grafana-tls-secret
  issuerRef:
    name: vault-issuer
    kind: ClusterIssuer
  commonName: grafana.shopkube.local
  dnsNames:
  - grafana.shopkube.local
  duration: 720h
  renewBefore: 240h
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: prometheus-tls
spec:
  secretName: prometheus-tls-secret
  issuerRef:
    name: vault-issuer
    kind: ClusterIssuer
  commonName: prometheus.shopkube.local
  dnsNames:
  - prometheus.shopkube.local
  duration: 720h
  renewBefore: 240h
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: alertmanager-tls
spec:
  secretName: alertmanager-tls-secret
  issuerRef:
    name: vault-issuer
    kind: ClusterIssuer
  commonName: alertmanager.shopkube.local
  dnsNames:
  - alertmanager.shopkube.local
  duration: 720h
  renewBefore: 240h
EOF
```

## Exposer les métriques du control plane

Par défaut, kubeadm configure les composants du control plane pour exposer leurs métriques sur `127.0.0.1` uniquement. Prometheus ne peut pas les atteindre depuis un pod sur un autre nœud. Les modifications suivantes sont nécessaires.

Sur controlplane-c1 (192.168.1.69) :

```bash
# controller-manager et scheduler
sudo sed -i 's/--bind-address=127.0.0.1/--bind-address=0.0.0.0/' \
  /etc/kubernetes/manifests/kube-controller-manager.yaml

sudo sed -i 's/--bind-address=127.0.0.1/--bind-address=0.0.0.0/' \
  /etc/kubernetes/manifests/kube-scheduler.yaml

# etcd
sudo sed -i 's|--listen-metrics-urls=http://127.0.0.1:2381|--listen-metrics-urls=http://0.0.0.0:2381|' \
  /etc/kubernetes/manifests/etcd.yaml
```

Ces fichiers sont des static pods — kubelet détecte les modifications et recrée les pods automatiquement.

Depuis admin-vm (192.168.1.100), pour kube-proxy :

```bash
kubectl edit configmap kube-proxy -n kube-system
# Modifier : metricsBindAddress: "0.0.0.0:10249"

kubectl rollout restart daemonset kube-proxy -n kube-system
```

## Ajouter les FQDN sur les clients

Sur chaque machine qui doit accéder aux interfaces de monitoring, ajouter cette ligne dans le fichier hosts :

Linux (`/etc/hosts`) et Windows (`C:\Windows\System32\drivers\etc\hosts`) :

```
192.168.1.202  grafana.shopkube.local prometheus.shopkube.local alertmanager.shopkube.local
```

## Identifiants Grafana par défaut

| Champ | Valeur |
|---|---|
| Utilisateur | admin |
| Mot de passe | shopkube-admin |
