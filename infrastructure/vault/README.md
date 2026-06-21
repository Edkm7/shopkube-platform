# Vault PKI

Vault est installé sur une VM dédiée hors cluster (192.168.1.121), sur la même machine que GitLab. Il sert d'autorité de certification interne pour signer les certificats TLS des services exposés sur le cluster.

## Installation

Sur la VM Vault (192.168.1.121) :

```bash
wget -O - https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list

sudo apt update && sudo apt install -y vault
```

Ne pas utiliser snap, le service systemd ne sera pas créé.

## Configuration

La configuration se trouve dans `/etc/vault.d/vault.hcl`. Vault utilise le stockage fichier pour persister les données entre les redémarrages.

```hcl
ui = true

storage "file" {
  path = "/opt/vault/data"
}

listener "tcp" {
  address     = "0.0.0.0:8200"
  tls_disable = true
}

api_addr     = "http://192.168.1.121:8200"
cluster_addr = "http://192.168.1.121:8201"
```

TLS est désactivé sur Vault lui-même car il tourne sur un réseau interne. En production, Vault devrait être derrière un reverse proxy avec TLS.

Sur la VM Vault (192.168.1.121) :

```bash
# Créer l'utilisateur système et les répertoires
sudo useradd --system --home /etc/vault.d --shell /bin/false vault
sudo mkdir -p /opt/vault/data
sudo chown -R vault:vault /opt/vault
sudo chown -R vault:vault /etc/vault.d

# Démarrer Vault
sudo systemctl enable --now vault
```

## Initialisation

Sur la VM Vault (192.168.1.121) :

```bash
export VAULT_ADDR='http://127.0.0.1:8200'

# Initialiser Vault — à exécuter une seule fois
vault operator init -key-shares=5 -key-threshold=3

# Déverrouiller Vault après chaque redémarrage — répéter 3 fois avec 3 clés différentes
vault operator unseal

# Vérifier que Vault est déverrouillé
vault status
```

L'initialisation génère 5 clés de déchiffrement (threshold 3) et un root token. Ces éléments ne sont générés qu'une seule fois et ne peuvent pas être récupérés ensuite.

En environnement de production, les clés de déchiffrement Vault sont stockées dans un boîtier HSM (Hardware Security Module) qui effectue les opérations cryptographiques sans jamais exposer les clés en clair. Des recovery keys sont générées séparément et conservées hors ligne en cas de perte ou de défaillance du HSM. Dans ce lab, les clés sont stockées manuellement à titre d'apprentissage uniquement.

## Configuration du moteur PKI

Sur la VM Vault (192.168.1.121) :

```bash
# Se connecter avec le root token
vault login

# Activer le moteur PKI
vault secrets enable pki
vault secrets tune -max-lease-ttl=87600h pki

# Générer la CA root avec une durée de 10 ans
vault write pki/root/generate/internal \
  common_name="ShopKube Internal CA" \
  issuer_name="shopkube-root" \
  ttl=87600h

# Créer le role pour le domaine shopkube.local
vault write pki/roles/shopkube-local \
  allowed_domains="shopkube.local" \
  allow_subdomains=true \
  allow_bare_domains=true \
  max_ttl=8760h
```

Vérifier la durée de la CA après création. Si le TTL n'a pas été pris en compte, la CA peut avoir une durée par défaut bien inférieure à 10 ans.

Sur la VM Vault (192.168.1.121) :

```bash
vault read pki/cert/ca -format=json | jq -r '.data.certificate' | openssl x509 -noout -dates
```

## Intégration avec cert-manager

Vault valide les ServiceAccounts Kubernetes pour authentifier cert-manager sans token statique.

Créer les ressources nécessaires sur le cluster. Depuis admin-vm :

```yaml
# vault-auth.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: vault-auth
  namespace: default
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: vault-auth-delegator
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:auth-delegator
subjects:
- kind: ServiceAccount
  name: vault-auth
  namespace: default
---
apiVersion: v1
kind: Secret
metadata:
  name: vault-auth-token
  namespace: default
  annotations:
    kubernetes.io/service-account.name: vault-auth
type: kubernetes.io/service-account-token
```

Depuis admin-vm :

```bash
kubectl apply -f vault-auth.yaml
```

Récupérer les informations du cluster. Depuis admin-vm :

```bash
export VAULT_SA_TOKEN=$(kubectl get secret vault-auth-token -n default -o jsonpath='{.data.token}' | base64 --decode)
export KUBE_CA_CERT=$(kubectl get secret vault-auth-token -n default -o jsonpath='{.data.ca\.crt}' | base64 --decode)
export KUBE_HOST=$(kubectl config view --raw --minify --flatten -o jsonpath='{.clusters[].cluster.server}')
```

Configurer le Kubernetes auth method. Sur la VM Vault (192.168.1.121) :

```bash
vault auth enable kubernetes

vault write auth/kubernetes/config \
  token_reviewer_jwt="$VAULT_SA_TOKEN" \
  kubernetes_host="$KUBE_HOST" \
  kubernetes_ca_cert="$KUBE_CA_CERT"

# Créer la policy
vault policy write cert-manager - <<EOF
path "pki/sign/shopkube-local" {
  capabilities = ["create", "update"]
}
EOF

# Créer le role
vault write auth/kubernetes/role/cert-manager \
  bound_service_account_names=cert-manager \
  bound_service_account_namespaces=cert-manager \
  policies=cert-manager \
  ttl=1h
```

## Installer la CA sur les clients

Depuis admin-vm :

```bash
# Récupérer le certificat CA
curl -s http://192.168.1.121:8200/v1/pki/ca/pem -o shopkube-ca.crt
```

Sur les nœuds Linux (controlplane-c1, worker1-c1, worker2-c1) :

```bash
sudo cp shopkube-ca.crt /usr/local/share/ca-certificates/shopkube-ca.crt
sudo update-ca-certificates
```

Sur Windows, importer `shopkube-ca.crt` dans le magasin "Autorités de certification racines de confiance" via `certmgr.msc` ou en PowerShell administrateur :

```powershell
Import-Certificate -FilePath "shopkube-ca.crt" -CertStoreLocation Cert:\LocalMachine\Root
```
