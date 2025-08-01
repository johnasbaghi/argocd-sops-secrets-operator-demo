### Generate a key pair, extract the public key and create a config file for sops

```
brew install age

age-keygen -o key.txt

public_key=$(age-keygen -y key.txt)

cat > ./.sops.yaml << EOF
creation_rules:
  - encrypted_regex: ^(data|stringData)$
    age: $public_key
EOF
```

### This key will have to be provided to the operator in one of the following ways

1. Store the key as a Secret and configure the operator to read it
2. Use a KMS (e.g. Vault) to manage the key and configure the operator
   to integrate with the KMS
3. Mount the key file into the operator's pod from a volume
    
### Create namespace

```
kubectl create namespace sops-operator
```

### Create a Secret using key.txt

kubectl create secret generic -n sops-operator age-key --from-file=./key.txt

### Install the operator

```
helm repo add sops https://isindir.github.io/sops-secrets-operator/

helm repo update

helm upgrade --install sops sops/sops-secrets-operator --namespace sops-operator -f values.yaml
```

### Use sops to encrypt secrets.yaml

```
sops -e secrets.yaml > app/sops-encrypted-secrets.yaml
```

### Create the encrypted secrets in the cluster

```
kubectl apply -f app/sops-encrypted-secrets.yaml
```

### Or push this repo to a Git forge and create an Application using app as your path

```
kubectl apply -n argocd -f application.yaml
```

### Verify that the secrets are created by the operator

```
$ kubectl get secret
NAME
some-secret
some-other-secret

$ kubectl get secret some-secret -o yaml | yq .data.password | base64 -d
somepassword

$ kubectl get secret some-other-secret -o yaml | yq .data.token | base64 -d
sometoken
```
