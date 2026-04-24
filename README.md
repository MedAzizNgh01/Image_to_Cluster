# ATELIER FROM IMAGE TO CLUSTER

## Description

Cet atelier consiste à construire une image Docker Nginx personnalisée avec Packer, puis à la déployer sur un cluster Kubernetes K3d dans un GitHub Codespace.

---

## Étape 1 — Création du cluster K3d

```bash
curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash

k3d cluster create lab \
  --servers 1 \
  --agents 2

kubectl get nodes
```

---

## Étape 2 — Installation de Packer et Ansible

```bash
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -
sudo apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"
sudo apt update && sudo apt install packer -y

sudo apt install ansible -y
```

---

## Étape 3 — Création du template Packer

Créez le fichier `nginx.pkr.hcl` :

```bash
nano nginx.pkr.hcl
```

Contenu du fichier :

```hcl
packer {
  required_plugins {
    docker = {
      source  = "github.com/hashicorp/docker"
      version = "~> 1"
    }
  }
}

source "docker" "nginx" {
  image  = "nginx:latest"
  commit = true
}

build {
  sources = ["source.docker.nginx"]

  provisioner "file" {
    source      = "index.html"
    destination = "/usr/share/nginx/html/index.html"
  }

  post-processors {
    post-processor "docker-tag" {
      repository = "mon-nginx-custom"
      tags       = ["latest"]
    }
  }
}
```

---

## Étape 4 — Build de l'image avec Packer

```bash
packer init .
packer build nginx.pkr.hcl
```

Vérification :

```bash
docker images | grep mon-nginx
```

---

## Étape 5 — Import de l'image dans K3d

```bash
k3d image import mon-nginx-custom:latest -c lab
```

---

## Étape 6 — Création du fichier de déploiement Kubernetes

Créez le fichier `deployment.yaml` :

```bash
nano deployment.yaml
```

Contenu du fichier :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-custom
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-custom
  template:
    metadata:
      labels:
        app: nginx-custom
    spec:
      containers:
      - name: nginx-custom
        image: mon-nginx-custom:latest
        imagePullPolicy: Never
        ports:
        - containerPort: 80
```

> `imagePullPolicy: Never` est obligatoire pour que Kubernetes utilise l'image locale importée.

---

## Étape 7 — Déploiement sur K3d

```bash
kubectl apply -f deployment.yaml
kubectl get pods
```

Résultat attendu :

```
NAME                            READY   STATUS    RESTARTS   AGE
nginx-custom-685688785c-p29n2   1/1     Running   0          ...
```

---

## Étape 8 — Exposition et accès à l'application

```bash
kubectl expose deployment nginx-custom --type=NodePort --port=80
kubectl port-forward svc/nginx-custom 8081:80 &
```

Dans l'onglet **[PORTS]** du Codespace, rendez le port **8081** public et ouvrez l'URL dans votre navigateur.
