# Operations

Day-2 reference for a project that's already deployed: troubleshooting, routine updates, monitoring, and common commands. For first-time setup, see [setup-guide.md](setup-guide.md).

Every snippet below uses `$APP_NAME` instead of a hardcoded name. Set it once per terminal session and every command works as-is, regardless of which project you're operating on:

```bash
export APP_NAME=your-app-name
```

---

## 🔍 Troubleshooting

### Pods in CrashLoopBackOff

```bash
# Check logs
kubectl logs POD_NAME

# Check pod details
kubectl describe pod POD_NAME

# Common causes:
# - Wrong environment variables
# - Database connection failed
# - Port mismatch
# - Missing secrets
```

### SSL Certificate Not Issuing

```bash
# Check certificate status
kubectl describe certificate $APP_NAME-tls

# Check cert-manager logs
kubectl logs -n cert-manager deploy/cert-manager

# Common causes:
# - DNS not propagated yet
# - Firewall blocking port 80/443
# - Wrong email in cluster-issuer
```

### Cannot Connect to Cluster

```bash
# Verify kubeconfig on VPS
kubectl get nodes

# If it works on VPS but not GitHub Actions:
# - Regenerate kubeconfig with correct server IP
# - Update KUBECONFIG secret in GitHub
```

### Database Connection Failed

```bash
# For MongoDB Atlas:
# - Add VPS IP to Network Access whitelist
# - Verify connection string in secrets

# Test connection from VPS:
mongosh "YOUR_MONGO_URI"
```

### Image Pull Errors

```bash
# Verify Docker Hub credentials
docker login

# Check if images exist (replace with your Docker Hub username)
docker pull DOCKERHUB_USERNAME/$APP_NAME-backend:latest

# Verify DOCKERHUB_USERNAME and DOCKERHUB_TOKEN secrets
```

---

## 🔄 Making Updates

After initial deployment, just push to main:

```bash
git add .
git commit -m "Your changes"
git push origin main
```

The pipeline will automatically:
1. Build new Docker images
2. Push to Docker Hub
3. Deploy to K3s with zero downtime rolling updates

---

## 📊 Monitoring

### View Application

- Frontend: `https://app.yourdomain.com`
- Backend: `https://api.app.yourdomain.com`

### Check Status

```bash
# Get all resources
kubectl get all

# Watch pods
kubectl get pods -w

# View logs (live)
kubectl logs -f deployment/$APP_NAME-backend
kubectl logs -f deployment/$APP_NAME-frontend

# Check resource usage
kubectl top pods
kubectl top nodes
```

---

## 🆘 Common Commands Reference

```bash
# Restart deployment
kubectl rollout restart deployment/$APP_NAME-backend
kubectl rollout restart deployment/$APP_NAME-frontend

# Scale replicas
kubectl scale deployment/$APP_NAME-backend --replicas=3

# Update a single key in an existing secret (no delete/recreate gap)
kubectl create secret generic $APP_NAME-secrets --from-literal=KEY=VALUE \
  --dry-run=client -o yaml | kubectl apply -f -

# View ingress details
kubectl describe ingress $APP_NAME-ingress

# Execute command in pod
kubectl exec -it POD_NAME -- /bin/sh

# Port forward for debugging (replace 5000 with your BACKEND_PORT)
kubectl port-forward service/$APP_NAME-backend 5000:5000

# Delete this app's resources only
# (raw files under kubernetes/ contain unexpanded ${VAR} placeholders and
# aren't meant to be applied/deleted directly — target objects by name instead.
# This intentionally skips the ClusterIssuers: they're cluster-scoped and
# shared by every app on this K3s cluster, so another project may depend on them.)
kubectl delete deployment $APP_NAME-backend $APP_NAME-frontend
kubectl delete service $APP_NAME-backend $APP_NAME-frontend
kubectl delete ingress $APP_NAME-ingress
kubectl delete secret $APP_NAME-secrets
```
