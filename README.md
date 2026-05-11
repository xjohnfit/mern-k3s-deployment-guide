# Complete Deployment Guide for Next.js Apps on K3s with GitHub Actions

This guide will help you deploy any Next.js application to a K3s cluster on a VPS using automated CI/CD with GitHub Actions.

## 📋 Prerequisites

- VPS with K3s cluster installed
- Docker Hub account
- A database (e.g., PostgreSQL, MySQL, MongoDB Atlas, or any hosted DB)
- Domain name with DNS access
- GitHub repository for your project

---

## 🗂️ Project Structure

Your project should have this structure:
```
your-app/
├── Dockerfile
├── next.config.js (or next.config.ts)
├── package.json
├── public/
├── src/ (or app/ at root)
├── kubernetes/
│   ├── deployment.yml
│   ├── service.yml
│   ├── cluster-issuer-prod.yml
│   ├── cluster-issuer-staging.yml
│   └── ingress.yml
└── .github/
    └── workflows/
        └── deploy.yml
```

> **Note:** Next.js handles both frontend rendering and backend API routes in a single application, so you only need **one** container and one set of Kubernetes manifests.

---

## ⚙️ Step 0: Configure Next.js for Standalone Output

Next.js has a built-in **standalone** output mode that produces a minimal, self-contained bundle — ideal for Docker.

In your `next.config.js` (or `next.config.ts`):

For **CommonJS** config (`next.config.js`):
```js
/** @type {import('next').NextConfig} */
const nextConfig = {
  output: 'standalone',
};

module.exports = nextConfig;
```

For **ES module** or **TypeScript** config (`next.config.mjs` / `next.config.ts`):
```ts
import type { NextConfig } from 'next';

const nextConfig: NextConfig = {
  output: 'standalone',
};

export default nextConfig;
```

This tells Next.js to bundle only the files needed to run the app, resulting in a much smaller Docker image.

---

## 🐳 Step 1: Create the Dockerfile

Create `Dockerfile` at the root of your project:

```dockerfile
# Build stage
FROM node:20-alpine AS builder

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install all dependencies (including dev for build)
RUN npm ci

# Copy source code
COPY . .

# Build the Next.js application
RUN npm run build

# Production stage
FROM node:20-alpine

WORKDIR /app

# Install dumb-init to handle PID 1 properly
RUN apk add --no-cache dumb-init

# Create non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

# Copy standalone output (includes server.js and required node_modules)
COPY --from=builder --chown=nodejs:nodejs /app/.next/standalone ./

# Copy static assets
COPY --from=builder --chown=nodejs:nodejs /app/.next/static ./.next/static

# Copy public folder
COPY --from=builder --chown=nodejs:nodejs /app/public ./public

# Switch to non-root user
USER nodejs

# Expose port
EXPOSE 3000

# Set environment to production
ENV NODE_ENV=production
ENV PORT=3000
ENV HOSTNAME="0.0.0.0"

# Health check — adjust path to a real route in your app
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
    CMD node -e "require('http').get('http://localhost:3000/', (r) => {process.exit(r.statusCode < 500 ? 0 : 1)})"

# Use dumb-init to handle signals properly
ENTRYPOINT ["dumb-init", "--"]

# Start the Next.js standalone server
CMD ["node", "server.js"]
```

**Notes:**
- The `output: 'standalone'` config in `next.config.js` is required for `server.js` to be generated
- `HOSTNAME="0.0.0.0"` is required so the server binds to all interfaces inside the container
- The health check probes `/` by default. If you want a dedicated health endpoint, create `app/api/health/route.ts` returning `{ status: 'ok' }` and update the path to `/api/health`

---

## ☸️ Step 2: Create Kubernetes Manifests

### App Deployment

Create `kubernetes/deployment.yml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: APP_NAME
  labels:
    app: APP_NAME
spec:
  replicas: 2
  selector:
    matchLabels:
      app: APP_NAME
  template:
    metadata:
      labels:
        app: APP_NAME
    spec:
      containers:
      - name: app
        image: DOCKERHUB_USERNAME/APP_NAME:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 3000
        env:
        - name: NODE_ENV
          value: "production"
        - name: PORT
          value: "3000"
        - name: HOSTNAME
          value: "0.0.0.0"
        - name: NEXTAUTH_URL
          value: "https://yourdomain.com"
        # Add all your environment variables from secrets
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: APP_NAME-secrets
              key: DATABASE_URL
        - name: NEXTAUTH_SECRET
          valueFrom:
            secretKeyRef:
              name: APP_NAME-secrets
              key: NEXTAUTH_SECRET
        # Add other secrets as needed (API keys, OAuth credentials, etc.)
        resources:
          requests:
            memory: "256Mi"
            cpu: "200m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
```

**Replace:**
- `APP_NAME` → your app name (e.g., `myapp`, `my-nextjs-app`)
- `DOCKERHUB_USERNAME` → your Docker Hub username
- `yourdomain.com` → your actual domain
- Add/remove environment variables as needed

### App Service

Create `kubernetes/service.yml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: APP_NAME
spec:
  selector:
    app: APP_NAME
  ports:
  - protocol: TCP
    port: 3000
    targetPort: 3000
  type: ClusterIP
```

### Cluster Issuers (SSL Certificates)

Create `kubernetes/cluster-issuer-prod.yml`:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: YOUR_EMAIL@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: traefik
```

Create `kubernetes/cluster-issuer-staging.yml`:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: YOUR_EMAIL@example.com
    privateKeySecretRef:
      name: letsencrypt-staging
    solvers:
    - http01:
        ingress:
          class: traefik
```

### Ingress (Routing & SSL)

Create `kubernetes/ingress.yml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: APP_NAME-ingress
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: web,websecure
    traefik.ingress.kubernetes.io/router.tls: "true"
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: traefik
  rules:
  - host: yourdomain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: APP_NAME
            port:
              number: 3000
  - host: www.yourdomain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: APP_NAME
            port:
              number: 3000
  tls:
  - hosts:
    - yourdomain.com
    - www.yourdomain.com
    secretName: APP_NAME-tls
```

**Replace:**
- `APP_NAME` → your app name
- `yourdomain.com` → your actual domain
- `YOUR_EMAIL@example.com` → your email for SSL certificates

---

## 🚀 Step 3: Create GitHub Actions Workflow

Create `.github/workflows/deploy.yml`:

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [ main ]
    paths-ignore:
      - 'kubernetes/**'
      - 'README.md'
      - '.gitignore'
  pull_request:
    branches: [ main ]

permissions:
  contents: read

jobs:
  trivy-scan:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    
    - name: Run Trivy vulnerability scanner
      uses: aquasecurity/trivy-action@master
      with:
        scan-type: 'fs'
        scan-ref: '.'
        format: 'table'
        severity: 'CRITICAL,HIGH,MEDIUM'
        scanners: 'vuln,secret,misconfig'
        exit-code: '0'  # Don't fail the build, just show results

  build-and-push:
    runs-on: ubuntu-latest
    needs: [trivy-scan]
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    steps:
    - uses: actions/checkout@v4
    
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3
    
    - name: Log in to Docker Hub
      uses: docker/login-action@v3
      with:
        username: ${{ secrets.DOCKERHUB_USERNAME }}
        password: ${{ secrets.DOCKERHUB_TOKEN }}
    
    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v5
      with:
        images: ${{ secrets.DOCKERHUB_USERNAME }}/APP_NAME
        tags: |
          type=sha,prefix={{branch}}-
          type=raw,value=latest,enable={{is_default_branch}}
    
    - name: Build and push
      uses: docker/build-push-action@v6
      with:
        context: .
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        cache-from: type=registry,ref=${{ secrets.DOCKERHUB_USERNAME }}/APP_NAME:buildcache
        cache-to: type=registry,ref=${{ secrets.DOCKERHUB_USERNAME }}/APP_NAME:buildcache,mode=max

  deploy:
    runs-on: ubuntu-latest
    needs: [build-and-push]
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    steps:
    - uses: actions/checkout@v4
    
    - name: Install kubectl
      uses: azure/setup-kubectl@v4
      with:
        version: 'latest'
    
    - name: Set up kubeconfig
      run: |
        mkdir -p $HOME/.kube
        echo "${{ secrets.KUBECONFIG }}" | base64 -d > $HOME/.kube/config
        chmod 600 $HOME/.kube/config
    
    - name: Set image tag
      id: image-tag
      run: |
        echo "IMAGE_TAG=main-$(git rev-parse --short HEAD)" >> $GITHUB_OUTPUT
    
    - name: Create or update Kubernetes secrets
      run: |
        # Delete existing secret if it exists (ignore errors if it doesn't)
        kubectl delete secret APP_NAME-secrets --ignore-not-found=true
        
        # Create new secret from GitHub Actions secrets
        kubectl create secret generic APP_NAME-secrets \
          --from-literal=DATABASE_URL="${{ secrets.DATABASE_URL }}" \
          --from-literal=NEXTAUTH_SECRET="${{ secrets.NEXTAUTH_SECRET }}"
          # Add all your other secrets here with --from-literal=KEY="${{ secrets.KEY }}"
    
    - name: Deploy to Kubernetes
      run: |
        kubectl apply -f kubernetes/cluster-issuer-staging.yml
        kubectl apply -f kubernetes/cluster-issuer-prod.yml
        kubectl apply -f kubernetes/deployment.yml
        kubectl apply -f kubernetes/service.yml
        kubectl apply -f kubernetes/ingress.yml
    
    - name: Update deployment with new image tag
      run: |
        # The container name 'app' must match the name in kubernetes/deployment.yml
        kubectl set image deployment/APP_NAME app=${{ secrets.DOCKERHUB_USERNAME }}/APP_NAME:${{ steps.image-tag.outputs.IMAGE_TAG }}
    
    - name: Wait for rollout to complete
      run: |
        kubectl rollout status deployment/APP_NAME --timeout=5m
    
    - name: Verify deployment
      run: |
        kubectl get pods -l app=APP_NAME
        kubectl get ingress APP_NAME-ingress
```

**Replace:**
- `APP_NAME` → your app name (in all places)
- Update the secrets in the "Create or update Kubernetes secrets" step to match your app's env vars

---

## 🔧 Step 4: VPS & K3s Setup

### 1. Prepare Kubeconfig

On your VPS:

```bash
# Get your VPS public IP
MY_IP=$(curl -s ifconfig.me)
echo "Your VPS IP: $MY_IP"

# Copy k3s config
sudo cp /etc/rancher/k3s/k3s.yaml ~/k3s-config.yaml
sudo chown $USER:$USER ~/k3s-config.yaml

# Replace 127.0.0.1 with your actual IP
sed -i "s/127.0.0.1/$MY_IP/g" ~/k3s-config.yaml

# Verify the change
grep "server:" ~/k3s-config.yaml

# Generate base64 encoded kubeconfig
cat ~/k3s-config.yaml | base64 -w 0
echo ""

# Clean up
rm ~/k3s-config.yaml
```

Copy the base64 output - you'll need it for GitHub secrets.

### 2. Ensure Required K3s Components

Make sure cert-manager is installed on your K3s cluster:

```bash
# Check if cert-manager is installed
kubectl get pods -n cert-manager

# If not installed, install it:
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml

# Wait for cert-manager to be ready
kubectl wait --for=condition=ready pod -l app.kubernetes.io/instance=cert-manager -n cert-manager --timeout=300s
```

---

## 🔐 Step 5: Configure GitHub Secrets

Go to your GitHub repository → Settings → Secrets and variables → Actions → New repository secret

### Required Secrets:

1. **DOCKERHUB_USERNAME** - Your Docker Hub username
2. **DOCKERHUB_TOKEN** - Docker Hub access token (create at hub.docker.com → Account Settings → Security)
3. **KUBECONFIG** - Base64 encoded kubeconfig from Step 4
4. **Your app-specific secrets** (examples):
   - `DATABASE_URL` — your database connection string (PostgreSQL, MySQL, MongoDB, etc.)
   - `NEXTAUTH_SECRET` — a random secret for NextAuth.js session encryption
   - `NEXTAUTH_URL` — your production URL (e.g., `https://yourdomain.com`)
   - Any third-party API keys, OAuth credentials, etc.

---

## 🌐 Step 6: Configure DNS

Add DNS A records pointing to your VPS IP:

- `yourdomain.com` → `YOUR_VPS_IP`
- `www.yourdomain.com` → `YOUR_VPS_IP`

Wait for DNS propagation (can take 5 mins to 24 hours).

---

## 🗄️ Step 7: Database Configuration

Next.js works with any database. Common options:

### PostgreSQL / MySQL (via Prisma or Drizzle)

1. Set up a hosted database (e.g., Supabase, PlanetScale, Railway, Neon, or self-hosted)
2. Whitelist your VPS IP address in the database provider's network settings
3. Save your connection string as `DATABASE_URL` GitHub secret

### MongoDB Atlas

1. Go to MongoDB Atlas → Network Access
2. Add your VPS IP address:
   ```bash
   # Get VPS IP
   curl -4 ifconfig.me
   ```
3. Or allow access from anywhere: `0.0.0.0/0` (less secure)
4. Save your MongoDB connection string as `DATABASE_URL` GitHub secret

### Other Databases

Ensure your database accepts connections from your VPS IP address.

---

## 🚀 Step 8: Deploy

### Initial Deployment

```bash
# Commit all files
git add .
git commit -m "Add deployment configuration"
git push origin main
```

### Monitor Deployment

1. Go to GitHub → Actions tab
2. Watch the pipeline execute
3. All three jobs should succeed:
   - trivy-scan
   - build-and-push
   - deploy

### Check Deployment on VPS

```bash
# Check pods
kubectl get pods

# View logs
kubectl logs -l app=APP_NAME

# Check ingress
kubectl get ingress

# Check SSL certificate
kubectl get certificate
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
# - Missing or wrong environment variables (DATABASE_URL, NEXTAUTH_SECRET, etc.)
# - Database connection failed
# - Missing output: 'standalone' in next.config.js
# - HOSTNAME not set to "0.0.0.0" (app binds only to localhost inside container)
```

### App Returns 500 Errors

```bash
# Check application logs
kubectl logs -l app=APP_NAME --tail=100

# Common causes:
# - DATABASE_URL secret missing or incorrect
# - NEXTAUTH_URL not matching the actual domain
# - Missing required environment variables
```

### SSL Certificate Not Issuing

```bash
# Check certificate status
kubectl describe certificate APP_NAME-tls

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
# Test connection from VPS (PostgreSQL example)
psql "$DATABASE_URL"

# For MongoDB Atlas:
# - Add VPS IP to Network Access whitelist
# - Verify connection string in secrets
```

### Image Pull Errors

```bash
# Verify Docker Hub credentials
docker login

# Check if image exists
docker pull DOCKERHUB_USERNAME/APP_NAME:latest

# Verify DOCKERHUB_USERNAME and DOCKERHUB_TOKEN secrets
```

---

## 📝 Customization Checklist

When adapting this for a new app:

- [ ] Add `output: 'standalone'` to `next.config.js`
- [ ] Replace `APP_NAME` everywhere
- [ ] Update Docker Hub username
- [ ] Update domain names in ingress and `NEXTAUTH_URL`
- [ ] Adjust environment variables to match your app
- [ ] Update resource limits based on app needs
- [ ] Add/remove secrets as needed
- [ ] Verify npm scripts (`npm run build`, etc.)
- [ ] Update email for SSL certificates
- [ ] Whitelist VPS IP in your database provider

---

## 🔄 Making Updates

After initial deployment, just push to main:

```bash
git add .
git commit -m "Your changes"
git push origin main
```

The pipeline will automatically:
1. Build a new Docker image
2. Push to Docker Hub
3. Deploy to K3s with zero downtime rolling updates

---

## 📊 Monitoring

### View Application

- App: `https://yourdomain.com`
- API Routes: `https://yourdomain.com/api/...`

### Check Status

```bash
# Get all resources
kubectl get all

# Watch pods
kubectl get pods -w

# View logs (live)
kubectl logs -f deployment/APP_NAME-backend
kubectl logs -f deployment/APP_NAME-frontend

# Check resource usage
kubectl top pods
kubectl top nodes
```

---

## 🎯 Best Practices

1. **Always test locally first** - Build Docker images locally before pushing
2. **Use staging environment** - Test with `letsencrypt-staging` first
3. **Monitor logs** - Check logs after each deployment
4. **Resource limits** - Set appropriate CPU/memory limits
5. **Security** - Keep secrets in GitHub Actions, never commit them
6. **Health checks** - Implement proper health endpoints
7. **Graceful shutdown** - Handle SIGTERM in your app
8. **Database backups** - Regular backups of your database
9. **SSL staging first** - Use staging issuer to avoid rate limits
10. **Document changes** - Keep deployment notes for your team

---

## 🆘 Common Commands Reference

```bash
# Restart deployment
kubectl rollout restart deployment/APP_NAME-backend
kubectl rollout restart deployment/APP_NAME-frontend

# Scale replicas
kubectl scale deployment/APP_NAME-backend --replicas=3

# Update secrets
kubectl delete secret APP_NAME-secrets
kubectl create secret generic APP_NAME-secrets --from-literal=KEY=VALUE

# View ingress details
kubectl describe ingress APP_NAME-ingress

# Execute command in pod
kubectl exec -it POD_NAME -- /bin/sh

# Port forward for debugging
kubectl port-forward service/APP_NAME-backend 4001:4001

# Delete all resources
kubectl delete -f kubernetes/
```

---

## 📚 Additional Resources

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [K3s Documentation](https://docs.k3s.io/)
- [Docker Documentation](https://docs.docker.com/)
- [Cert-Manager Documentation](https://cert-manager.io/docs/)
- [Traefik Documentation](https://doc.traefik.io/traefik/)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)

---

**Good luck with your deployments! 🚀**
