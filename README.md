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

Construction de l'image Docker et push vers un dépôt privé AWS ECR.

**Concepts démontrés :**
- `docker build` et tag de l'image avec l'URI complète du registre ECR (`<account>.dkr.ecr.<region>.amazonaws.com/<repo>:<tag>`)
- Authentification ECR via `aws ecr get-login-password | docker login --username AWS --password-stdin <uri>` — utilisation de `--password-stdin` pour éviter que le token n'apparaisse dans l'historique shell
- Distinction entre stockage inline des credentials (`auths` dans `config.json`) et délégation à un credential store externe (`credsStore`) — le premier étant requis pour la création d'un Secret Kubernetes

### Étape 2 — Daemon Docker isolé de Minikube

Minikube exécute son propre daemon Docker dans une VM isolée et ne partage pas le `~/.docker/config.json` de l'hôte.

**Concepts démontrés :**
- SSH dans Minikube via `minikube ssh` pour accéder au daemon Docker interne
- Login manuel depuis l'intérieur de Minikube avec le token ECR
- Copie du `config.json` généré vers l'hôte WSL avec `minikube cp` pour créer le Secret Kubernetes

### Étape 3 — Création du Secret Kubernetes

Le composant `Secret` de type `kubernetes.io/dockerconfigjson` fournit à Kubernetes les credentials nécessaires au pull de l'image privée.

**Deux méthodes :**

**Méthode 1 — depuis `config.json`** (recommandée pour plusieurs registres) : encodage base64 du fichier de config et injection dans un manifest YAML, ou passage direct via `kubectl create secret generic --from-file=.dockerconfigjson`.

**Méthode 2 — credentials directs** (plus simple pour un seul registre) : `kubectl create secret docker-registry` avec `--docker-server`, `--docker-username`, `--docker-password`.

**Concepts démontrés :**
- Les Secrets sont scoped au namespace — ils doivent être recréés dans chaque namespace qui en a besoin
- Les tokens ECR expirent après **12 heures** — automatisation via CronJob recommandée en production
- Observation de l'erreur `ErrImagePull` sans secret, puis résolution après création

### Étape 4 — Déploiement avec `imagePullSecrets`

Référencement du Secret dans le manifest de Deployment via le champ `imagePullSecrets`.

**Concepts démontrés :**
- `imagePullPolicy: Always` — force un pull à chaque démarrage de pod, même si l'image existe localement
- Validation du déploiement réussi via `kubectl describe pod` (événements `Pulling` → `Pulled` → `Started`)
- Test des deux secrets en parallèle avec deux Deployments distincts

---

## EN — Description

### Step 1 — Build and Push to AWS ECR

Build the Docker image and push it to a private AWS ECR repository.

**Concepts demonstrated:**
- `docker build` and tagging the image with the full ECR registry URI (`<account>.dkr.ecr.<region>.amazonaws.com/<repo>:<tag>`)
- ECR authentication via `aws ecr get-login-password | docker login --username AWS --password-stdin <uri>` — using `--password-stdin` to avoid the token appearing in shell history
- Distinction between inline credential storage (`auths` in `config.json`) and delegation to an external credential store (`credsStore`) — the former is required for Kubernetes Secret creation

### Step 2 — Minikube's Isolated Docker Daemon

Minikube runs its own Docker daemon inside an isolated VM and does not share the host's `~/.docker/config.json`.

**Concepts demonstrated:**
- SSH into Minikube via `minikube ssh` to access the internal Docker daemon
- Manual login from inside Minikube using the ECR token
- Copying the generated `config.json` to the WSL host via `minikube cp` to create the Kubernetes Secret

### Step 3 — Creating the Kubernetes Secret

The `Secret` component of type `kubernetes.io/dockerconfigjson` provides Kubernetes with the credentials needed to pull the private image.

**Two methods:**

**Method 1 — from `config.json`** (recommended for multiple registries): base64-encode the config file and inject it into a YAML manifest, or pass it directly via `kubectl create secret generic --from-file=.dockerconfigjson`.

**Method 2 — direct credentials** (simpler for a single registry): `kubectl create secret docker-registry` with `--docker-server`, `--docker-username`, `--docker-password`.

**Concepts demonstrated:**
- Secrets are namespace-scoped — they must be recreated in each namespace that needs them
- ECR tokens expire after **12 hours** — automation via CronJob is recommended in production
- Observing the `ErrImagePull` error without a secret, then resolving it after creation

### Step 4 — Deployment with `imagePullSecrets`

Reference the Secret in the Deployment manifest via the `imagePullSecrets` field.

**Concepts demonstrated:**
- `imagePullPolicy: Always` — forces a fresh pull on every pod start, even if the image exists locally
- Validating successful deployment via `kubectl describe pod` (events: `Pulling` → `Pulled` → `Started`)
- Testing both secrets in parallel with two distinct Deployments
