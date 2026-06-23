# Monitoring

Ce dossier contient la configuration de la stack d'observabilité basée sur kube-prometheus-stack. Elle déploie Prometheus, Grafana et Alertmanager dans le namespace `monitoring`, sur un nœud dédié isolé des workloads applicatifs.

## Pourquoi un nœud dédié

La stack de monitoring tourne exclusivement sur worker3-c1 (192.168.1.72). Cette décision a été prise suite à un incident révélateur : lors de l'arrêt de worker1-c1 pour tester une alerte `NodeUnreachable`, on a réalisé qu'une panne de worker2-c1 aurait pu emporter Grafana et Prometheus en même temps que les pods ShopKube. L'outil qui surveille ne doit pas dépendre de ce qu'il surveille.

Worker3-c1 est protégé par une taint `dedicated=monitoring:NoSchedule` qui empêche les workloads applicatifs de s'y installer. Tous les composants monitoring ont la toleration correspondante et un nodeSelector `role=monitoring` pour y être forcés.

## Architecture d'accès

| Service | URL | IP |
|---|---|---|
| Grafana | https://grafana.shopkube.local | 192.168.1.202 |
| Prometheus | https://prometheus.shopkube.local | 192.168.1.202 |
| Alertmanager | https://alertmanager.shopkube.local | 192.168.1.202 |

Les certificats TLS sont signés par Vault et gérés automatiquement par cert-manager. L'instance Traefik dédiée au monitoring (192.168.1.202) est isolée du Traefik de production (192.168.1.200).

## Préparer worker3-c1

Depuis admin-vm (192.168.1.100) :

```bash
kubectl taint node worker3-c1 dedicated=monitoring:NoSchedule
kubectl label node worker3-c1 role=monitoring
```

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
  --set providers.kubernetesIngress.ingressClass=traefik-monitoring \
  --set tolerations[0].key=dedicated \
  --set tolerations[0].operator=Equal \
  --set tolerations[0].value=monitoring \
  --set tolerations[0].effect=NoSchedule \
  --set nodeSelector.role=monitoring

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

Par défaut, kubeadm configure les composants du control plane pour exposer leurs métriques sur `127.0.0.1` uniquement. Prometheus ne peut pas les atteindre depuis un pod sur un autre nœud.

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

## Configuration Alertmanager

Alertmanager est configuré pour envoyer les notifications vers Slack. La configuration contient le webhook Slack et est stockée dans un Secret Kubernetes (non versionné).

```bash
kubectl create secret generic alertmanager-monitoring-kube-prometheus-alertmanager \
  --from-literal=alertmanager.yaml='
global:
  slack_api_url: "<WEBHOOK_URL>"
route:
  receiver: slack-alertes
  group_by: [alertname, severity]
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
receivers:
- name: slack-alertes
  slack_configs:
  - channel: "#alertes"
    title: "{{ .GroupLabels.alertname }}"
    text: "{{ range .Alerts }}{{ .Annotations.description }}\n{{ end }}"
    send_resolved: true
' -n monitoring --dry-run=client -o yaml | kubectl apply -f -
```

## Ajouter les FQDN sur les clients

Sur chaque machine qui doit accéder aux interfaces de monitoring, ajouter cette ligne dans le fichier hosts.

Linux (`/etc/hosts`) et Windows (`C:\Windows\System32\drivers\etc\hosts`) :

```
192.168.1.202  grafana.shopkube.local prometheus.shopkube.local alertmanager.shopkube.local
```

## Identifiants Grafana par défaut

| Champ | Valeur |
|---|---|
| Utilisateur | admin |
| Mot de passe | shopkube-admin |
