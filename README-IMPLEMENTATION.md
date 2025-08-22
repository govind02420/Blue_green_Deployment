# Blue-Green Deployment: Node.js + MongoDB + Docker + Minikube

This repo contains:
- Backend Express API with MongoDB (`/backend`)
- Two frontends: **Basic** (`/frontend-blue`) and **Enhanced** (`/frontend-green`)
- Dockerfiles for each service
- `docker-compose.yml` for local multi-container dev
- Kubernetes manifests for Minikube (with blue-green switch via Service selector)

---

## Part 1: Run Locally (no Docker)

**Prereqs:** Node 18+, npm, MongoDB

1. Start MongoDB locally:
   ```bash
   mongod --dbpath /your/db/path
   ```

2. Backend
   ```bash
   cd backend
   npm install
   export MONGO_URI="mongodb://localhost:27017/bluegreen"
   export PORT=5000
   node server.js
   # Health check
   curl http://localhost:5000/health
   ```

3. Frontend (Basic)
   ```bash
   cd ../frontend-blue
   npm install
   export PORT=3100
   # (Optional) If you changed backend port/host, edit public/index.html to point to your backend API
   node server.js
   # Health:
   curl http://localhost:3100/health
   ```

4. Frontend (Enhanced)
   ```bash
   cd ../frontend-green
   npm install
   export PORT=3200
   node server.js
   curl http://localhost:3200/health
   ```

5. Test in browser
   - Basic UI: http://localhost:3100
   - Enhanced UI: http://localhost:3200
   - Register users in either UI. Data stored in MongoDB.
   - API: http://localhost:5000/api/users

---

## Part 2: Containerization

Dockerfiles are in each service folder. A small entrypoint script replaces
the hardcoded backend URL in the frontends with the `BACKEND_URL` env var
at container start.

Build & run with Compose:
```bash
docker compose build
docker compose up -d
```

Services:
- MongoDB: `localhost:27017`
- Backend: `http://localhost:5000`
- Basic UI: `http://localhost:3100`
- Enhanced UI: `http://localhost:3200`

The Compose file sets `BACKEND_URL=http://localhost:5000/api/users` for both frontends.

Verify:
```bash
curl http://localhost:5000/health
curl http://localhost:3100/health
curl http://localhost:3200/health
```

---

## Part 3: Kubernetes (Minikube)

**Prereqs:** Minikube + kubectl. Start Minikube and set Docker env so images are locally available to cluster:
```bash
minikube start
eval $(minikube docker-env)   # PowerShell: & minikube -p minikube docker-env | Invoke-Expression
```

Build images inside Minikube's Docker:
```bash
docker build -t govind2024/backend:v1 ./backend
docker build -t govind2024/frontend-blue:v1 ./frontend-blue
docker build -t govind2024/frontend-green:v1 ./frontend-green
```

Apply manifests:
```bash
kubectl apply -f k8s/mongo.yaml
kubectl apply -f k8s/backend.yaml
kubectl apply -f k8s/frontend-blue.yaml
kubectl apply -f k8s/frontend-green.yaml
kubectl apply -f k8s/frontend-service.yaml
# (optional) kubectl apply -f k8s/ingress.yaml && minikube addons enable ingress
```

Check:
```bash
kubectl get pods,svc
kubectl logs deploy/backend
kubectl get svc frontend
```

Access frontend Service (NodePort):
```bash
minikube service frontend --url
# Open the printed URL in your browser
```

Health checks/readiness probes are configured on all pods.

The frontends use `BACKEND_URL=http://backend:5000/api/users` inside the cluster.

---

## Part 4: Blue-Green Switch

The **Deployments**:
- `frontend-blue` labeled `app=frontend, color=blue`
- `frontend-green` labeled `app=frontend, color=green`

The **Service** `frontend` initially selects:
```yaml
selector:
  app: frontend
  color: blue
```

### Switch traffic to green
```bash
kubectl patch service frontend -p '{"spec":{"selector":{"app":"frontend","color":"green"}}}'
```

Verify:
```bash
minikube service frontend --url
curl $(minikube service frontend --url)/health
# The JSON will show "version": "green"
```

### Rollback to blue
```bash
kubectl patch service frontend -p '{"spec":{"selector":{"app":"frontend","color":"blue"}}}'
```

### Strategy Notes
- Both versions run side-by-side.
- Traffic is switched instantly via Service label selector update.
- Rollback is just switching the selector back.
- Backend and MongoDB are shared.

---

## Screenshots to Include (for submission)
1. Local (non-Docker) servers running and UIs accessible
2. `docker compose ps` showing all containers healthy
3. `kubectl get pods,svc -n default` showing pods/services in Minikube
4. Blue-green switch in action:
   - Output of `/health` before and after `kubectl patch`
   - Browser screenshots showing basic vs enhanced UI

---

## Troubleshooting

- **Pods not ready**: `kubectl describe pod <name>` and `kubectl logs`
- **Images not found**: Ensure you built images after `eval $(minikube docker-env)`
- **Ingress 404**: Enable addon: `minikube addons enable ingress`
- **CORS**: Backend enables CORS globally. If needed, restrict origins.
- **Mongo connection**: Confirm `mongo` service DNS resolves in pod: `ping mongo`
