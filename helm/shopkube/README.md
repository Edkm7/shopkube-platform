# ShopKube Helm Chart

Ce chart déploie l'ensemble des 12 microservices de ShopKube ainsi que les ressources associées : Services, ConfigMap, Ingress, Certificate cert-manager et PersistentVolumeClaim Redis. Il a été conçu pour fonctionner sur plusieurs environnements depuis un seul jeu de templates.

## Structure du chart

| Fichier / Dossier | Contenu |
|---|---|
| `Chart.yaml` | Métadonnées du chart (nom, version, appVersion) |
| `values.yaml` | Valeurs par défaut communes à tous les environnements |
| `values-prod.yaml` | Surcharge production (replicas élevés, contraintes de scheduling activées) |
| `values-dev.yaml` | Surcharge développement (1 replica, contraintes désactivées) |
| `templates/` | Un fichier Deployment et un fichier Service par microservice |
| `templates/serviceaccounts.yaml` | Les 11 ServiceAccounts générés par une boucle range |
| `templates/configmap.yaml` | ConfigMap shopkube-config (REDIS_ADDR, DISABLE_PROFILER...) |
| `templates/ingress.yaml` | Ingress Traefik avec TLS |
| `templates/certificate.yaml` | Certificate cert-manager signé par Vault |
| `templates/redis-pvc.yaml` | PersistentVolumeClaim NFS pour Redis |

## Déploiement

Depuis admin-vm (192.168.1.100) :

```bash
# Production
helm install shopkube-prod ./helm/shopkube \
  -f helm/shopkube/values-prod.yaml \
  -n shopkube-prod --create-namespace

# Développement
helm install shopkube-dev ./helm/shopkube \
  -f helm/shopkube/values-dev.yaml \
  -n shopkube-dev --create-namespace
```

## Mise à jour et rollback

Depuis admin-vm (192.168.1.100) :

```bash
# Mettre à jour un environnement
helm upgrade shopkube-prod ./helm/shopkube \
  -f helm/shopkube/values-prod.yaml \
  -n shopkube-prod

# Voir l'historique des révisions
helm history shopkube-prod -n shopkube-prod

# Revenir à la révision précédente
helm rollback shopkube-prod -n shopkube-prod
```

## Valider le rendu sans déployer

Depuis admin-vm (192.168.1.100) :

```bash
# Rendu complet
helm template shopkube-prod ./helm/shopkube -f helm/shopkube/values-prod.yaml

# Rendu d'un seul template
helm template . -s templates/frontend-deployment.yaml

# Vérifier la syntaxe
helm lint ./helm/shopkube/
```

## Points importants

### Le tiret dans redis-cart

Le nom `redis-cart` contient un tiret que Helm interprète comme une soustraction dans la notation pointée. Toutes les références à ce microservice dans les templates utilisent la fonction `index` à la place :

```yaml
# Mauvais — Helm interprète le tiret comme une soustraction
{{ .Values.redis-cart.replicas }}

# Correct
{{ index .Values "redis-cart" "replicas" }}
```

### Le PVC Redis est protégé contre la suppression

Le PVC Redis est inclus dans le chart mais annoté pour ne pas être supprimé lors d'un `helm uninstall`. Les données persistent même si le chart est désinstallé.

```yaml
metadata:
  annotations:
    helm.sh/resource-policy: keep
```

### Les champs optionnels sont conditionnés

Les champs comme `priorityClassName` qui peuvent être vides selon l'environnement sont conditionnés pour ne pas apparaître dans le YAML généré quand leur valeur est vide. Un champ vide dans un manifeste Kubernetes provoque une erreur.

```yaml
{{- if .Values.checkoutservice.priorityClassName }}
priorityClassName: {{ .Values.checkoutservice.priorityClassName }}
{{- end }}
```

### Désactiver l'Ingress

Si cert-manager ou Vault ne sont pas disponibles sur le cluster cible, l'Ingress et le Certificate peuvent être désactivés :

```bash
helm install shopkube-dev ./helm/shopkube \
  -f helm/shopkube/values-dev.yaml \
  --set ingress.enabled=false \
  -n shopkube-dev --create-namespace
```
