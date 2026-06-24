# Kubernetes — Deploy from a Private AWS ECR Registry

> **FR** — Déploiement d'une application Node.js conteneurisée sur Kubernetes (Minikube) en tirant l'image depuis un registre privé AWS ECR. Mise en place de l'authentification via un Secret Kubernetes de type `dockerconfigjson`, avec deux méthodes de création et analyse du fonctionnement du daemon Docker isolé de Minikube.
>
> **EN** — Deployment of a containerized Node.js application on Kubernetes (Minikube) by pulling the image from a private AWS ECR registry. Authentication is handled via a Kubernetes `dockerconfigjson` Secret, covering two creation methods and an analysis of Minikube's isolated Docker daemon.

---

## Stack

![Kubernetes](https://img.shields.io/badge/Kubernetes-Minikube-326CE5?logo=kubernetes)
![Docker](https://img.shields.io/badge/Docker-ECR%20%7C%20Registry-2496ED?logo=docker)
![AWS](https://img.shields.io/badge/AWS-ECR-orange?logo=amazonaws)
![Node.js](https://img.shields.io/badge/Node.js-20--alpine-339933?logo=node.js)

---

## FR — Description

### Étape 1 — Build et push vers AWS ECR

Construction de l'image Docker et push vers un dépôt privé AWS ECR. L'image est taguée avec l'URI complète du registre (`<account>.dkr.ecr.<region>.amazonaws.com/<repo>:<tag>`). L'authentification ECR se fait via `aws ecr get-login-password | docker login --username AWS --password-stdin <uri>` — `--password-stdin` évite que le token apparaisse dans l'historique shell. Important : Docker doit être configuré en mode credentials inline (`auths` dans `config.json`) plutôt qu'avec un credential store externe (`credsStore`) — seul le mode inline permet de créer un Secret Kubernetes valide.

### Étape 2 — Daemon Docker isolé de Minikube

Minikube exécute son propre daemon Docker dans une VM isolée et ne partage pas le `~/.docker/config.json` de l'hôte. L'accès au daemon interne se fait via `minikube ssh`. Après le login ECR depuis l'intérieur de Minikube, le `config.json` généré est copié vers l'hôte WSL avec `minikube cp` pour servir de base au Secret Kubernetes.

### Étape 3 — Création du Secret Kubernetes

Le composant `Secret` de type `kubernetes.io/dockerconfigjson` fournit à Kubernetes les credentials nécessaires au pull de l'image privée.

Deux méthodes de création :
- Depuis `config.json` (plusieurs registres) : encodage base64 du fichier et injection dans un manifest YAML, ou `kubectl create secret generic --from-file=.dockerconfigjson`
- Credentials directs (un seul registre) : `kubectl create secret docker-registry` avec `--docker-server`, `--docker-username`, `--docker-password`

Les Secrets sont scoped au namespace — ils doivent être recréés dans chaque namespace qui en a besoin. Les tokens ECR expirent après **12 heures** — automatisation via CronJob recommandée en production. L'erreur `ErrImagePull` observable sans secret disparaît après création.

### Étape 4 — Déploiement avec `imagePullSecrets`

Le Secret est référencé dans le manifest Deployment via le champ `imagePullSecrets`. `imagePullPolicy: Always` force un pull à chaque démarrage de pod, même si l'image existe localement. Validation via `kubectl describe pod` (séquence d'événements `Pulling` → `Pulled` → `Started`). Les deux méthodes de création de Secret sont testées en parallèle avec deux Deployments distincts.

---

## EN — Description

### Step 1 — Build and Push to AWS ECR

Build the Docker image and push it to a private AWS ECR repository. The image is tagged with the full registry URI (`<account>.dkr.ecr.<region>.amazonaws.com/<repo>:<tag>`). ECR authentication uses `aws ecr get-login-password | docker login --username AWS --password-stdin <uri>` — `--password-stdin` keeps the token out of shell history. One important distinction: Docker must be configured with inline credential storage (`auths` in `config.json`), not an external credential store (`credsStore`) — only inline credentials work for creating a valid Kubernetes Secret.

### Step 2 — Minikube's Isolated Docker Daemon

Minikube runs its own Docker daemon inside an isolated VM and does not share the host's `~/.docker/config.json`. The internal daemon is accessed via `minikube ssh`. After logging in to ECR from inside Minikube, the generated `config.json` is copied to the WSL host via `minikube cp` to use as the Secret source.

### Step 3 — Creating the Kubernetes Secret

The `Secret` of type `kubernetes.io/dockerconfigjson` gives Kubernetes the credentials needed to pull the private image.

Two creation methods:
- From `config.json` (multiple registries): base64-encode the file and inject into a YAML manifest, or use `kubectl create secret generic --from-file=.dockerconfigjson`
- Direct credentials (single registry): `kubectl create secret docker-registry` with `--docker-server`, `--docker-username`, `--docker-password`

Secrets are namespace-scoped — they must be recreated in each namespace that needs them. ECR tokens expire after **12 hours** — a CronJob is the recommended production solution. The `ErrImagePull` error is visible without the secret and clears up immediately after creation.

### Step 4 — Deployment with `imagePullSecrets`

The Secret is referenced in the Deployment manifest via `imagePullSecrets`. `imagePullPolicy: Always` forces a fresh pull on every pod start, even if the image exists locally. Deployment is validated via `kubectl describe pod` (event sequence: `Pulling` → `Pulled` → `Started`). Both Secret creation methods are tested in parallel with two separate Deployments.
