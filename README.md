# Secure Event Ticketing Platform – DevOps projekt

**Studentica:** Gabriela Gavrić  
**Kolegij:** Uvod u DevOps – DevSecOps  
**Akademska godina:** 2025./2026.

## Arhitektura

| Servis | Tehnologija | Port |
|--------|-------------|------|
| frontend | nginx:alpine (staticki HTML) | 3000 |
| api | Node.js + Express | 8080 |
| worker | Node.js (queue consumer) | – |
| postgres | PostgreSQL 16 | 5432 |
| redis | Redis 7 | 6379 |

## 1. dio – Lokalni razvoj (Docker/Podman Compose)

### Preduvjeti
- Docker ili Podman + podman-compose

### Pokretanje

```bash
cp .env.example .env
# Uredi .env i postavi POSTGRES_PASSWORD

podman compose up --build -d

# Provjera statusa
podman compose ps

# Validacija
curl http://localhost:8080/healthz
curl http://localhost:8080/readyz
curl http://localhost:8080/events

# Otvori browser: http://localhost:3000
```

### Zaustavljanje

```bash
podman compose down          # cuva podatke
podman compose down -v       # brise volumen (cistи restart)
```

## 2. dio – Produkcija (Kubernetes)

```bash
# Kreiranje namespacea
kubectl apply -f k8s/namespace.yaml

# ConfigMap i Secret
kubectl apply -f k8s/configmap.yaml
kubectl create secret generic db-secret \
  --from-literal=POSTGRES_PASSWORD=<lozinka> \
  -n ticketing

# RBAC
kubectl apply -f k8s/rbac.yaml

# Deployment svih servisa
kubectl apply -f k8s/postgres-deployment.yaml
kubectl apply -f k8s/redis-deployment.yaml
kubectl apply -f k8s/api-deployment.yaml
kubectl apply -f k8s/worker-deployment.yaml
kubectl apply -f k8s/frontend-deployment.yaml

# Mreza i pristup
kubectl apply -f k8s/network-policy.yaml
kubectl apply -f k8s/ingress.yaml

# Provjera
kubectl get all -n ticketing
```

### Rolling update i rollback

```bash
# Update na novu verziju
kubectl set image deployment/api api=ghcr.io/<user>/event-ticketing/api:<sha> -n ticketing
kubectl rollout status deployment/api -n ticketing

# Rollback
kubectl rollout undo deployment/api -n ticketing
```

## Sigurnosni elementi

- Multi-stage Containerfile build (minimalna runtime slika)
- Non-root korisnik u svim kontejnerima (UID 1001)
- Trivy skeniranje slika u CI/CD pipelineu
- CodeQL staticka analiza koda
- Kubernetes Secrets za osjetljive podatke
- NetworkPolicy segmentacija prometa
- RBAC sa minimalnim privilegijama
- Liveness i readiness health probe

## Trivy skeniranje (lokalno)

```bash
# Instalacija
sudo apt-get install -y wget apt-transport-https gnupg
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo "deb https://aquasecurity.github.io/trivy-repo/deb generic main" | sudo tee /etc/apt/sources.list.d/trivy.list
sudo apt-get update && sudo apt-get install -y trivy

# Skeniranje izgradene slike
trivy image --severity HIGH,CRITICAL event-ticketing_api:latest
```
