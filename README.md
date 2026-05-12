# Complete Deployment Guide for MERN Apps on K3s with GitHub Actions

This guide will help you deploy any MERN (or similar full-stack) application to a K3s cluster on a VPS using automated CI/CD with GitHub Actions.

## 📋 Prerequisites

- VPS with K3s cluster installed
- Docker Hub account
- MongoDB Atlas (or other hosted database)
- Domain name with DNS access
- GitHub repository for your project

---

## 🗂️ Project Structure

Your project should have this structure:
```
your-app/
├── backend/
│   ├── Dockerfile
│   ├── package.json
│   ├── tsconfig.json
│   └── src/
├── frontend/
│   ├── Dockerfile
│   ├── nginx.conf
│   ├── package.json
│   └── src/
├── kubernetes/
│   ├── backend-deployment.yml
│   ├── backend-service.yml
│   ├── frontend-deployment.yml
│   ├── frontend-service.yml
│   ├── cluster-issuer-prod.yml
│   ├── cluster-issuer-staging.yml
│   └── ingress.yml
└── .github/
    └── workflows/
        └── deploy.yml
```

---

## 🐳 Step 1: Create Docker Files

### Backend Dockerfile

Create `backend/Dockerfile`:

```dockerfile
# Build stage
FROM node:20-alpine AS builder

# Set working directory
WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies (including dev dependencies for build)
RUN npm ci

# Copy source code
COPY . .

# Build TypeScript
RUN npm run build

# Production stage
FROM node:20-alpine

# Create app directory
WORKDIR /app

# Install dumb-init to handle PID 1 properly
RUN apk add --no-cache dumb-init

# Create non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

# Copy package files
COPY package*.json ./

# Install only production dependencies
RUN npm ci --only=production && \
    npm cache clean --force

# Copy built application from builder stage
COPY --from=builder --chown=nodejs:nodejs /app/dist ./dist

# Switch to non-root user
USER nodejs

# Expose port (use your backend port)
EXPOSE 4001

# Set environment to production
ENV NODE_ENV=production

# Health check (adjust path and port as needed)
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD node -e "require('http').get('http://localhost:4001/api/health', (r) => {process.exit(r.statusCode === 200 ? 0 : 1)})"

# Use dumb-init to handle signals properly
ENTRYPOINT ["dumb-init", "--"]

# Start the application
CMD ["node", "dist/index.js"]
```

**Adjust:**
- Port number (4001 → your port)
- Health check path (`/api/health`)
- Entry file (`dist/index.js`)

### Frontend Dockerfile

Create `frontend/Dockerfile`:

```dockerfile
# Build stage
FROM node:20-alpine AS builder

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm ci

# Copy source code
COPY . .

# Build the application
RUN npm run build

# Production stage
FROM nginx:alpine

# Install dumb-init for proper signal handling
RUN apk add --no-cache dumb-init

# Create non-root user
RUN addgroup -g 1001 -S nginx-app && \
    adduser -S nginx-app -u 1001

# Copy custom nginx configuration
COPY --chown=nginx-app:nginx-app nginx.conf /etc/nginx/nginx.conf

# Copy built application
COPY --from=builder --chown=nginx-app:nginx-app /app/dist /usr/share/nginx/html

# Create necessary directories and set permissions
RUN mkdir -p /var/cache/nginx /var/log/nginx /tmp && \
    chown -R nginx-app:nginx-app /var/cache/nginx /var/log/nginx /tmp /usr/share/nginx/html && \
    chmod -R 755 /var/cache/nginx /var/log/nginx /tmp /usr/share/nginx/html

# Switch to non-root user
USER nginx-app

# Expose port
EXPOSE 8080

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD wget --quiet --tries=1 --spider http://localhost:8080/ || exit 1

# Use dumb-init to handle signals properly
ENTRYPOINT ["dumb-init", "--"]

# Start nginx
CMD ["nginx", "-g", "daemon off;"]
```

**Adjust:**
- Build output directory (`/app/dist` → your build folder: might be `/app/build`)

### Frontend nginx.conf

Create `frontend/nginx.conf`:

```nginx
worker_processes auto;
error_log /var/log/nginx/error.log warn;
pid /tmp/nginx.pid;

events {
    worker_connections 1024;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';

    access_log /var/log/nginx/access.log main;

    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;
    client_max_body_size 20M;

    # Gzip compression
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types text/plain text/css text/xml text/javascript 
               application/json application/javascript application/xml+rss 
               application/rss+xml font/truetype font/opentype 
               application/vnd.ms-fontobject image/svg+xml;
    gzip_disable "msie6";

    server {
        listen 8080;
        server_name _;
        root /usr/share/nginx/html;
        index index.html;

        # Security headers
        add_header X-Frame-Options "SAMEORIGIN" always;
        add_header X-Content-Type-Options "nosniff" always;
        add_header X-XSS-Protection "1; mode=block" always;
        add_header Referrer-Policy "no-referrer-when-downgrade" always;

        # Handle React Router (SPA)
        location / {
            try_files $uri $uri/ /index.html;
        }

        # Cache static assets
        location ~* \.(jpg|jpeg|png|gif|ico|css|js|svg|woff|woff2|ttf|eot)$ {
            expires 1y;
            add_header Cache-Control "public, immutable";
        }

        # Disable cache for index.html
        location = /index.html {
            add_header Cache-Control "no-cache, no-store, must-revalidate";
            expires 0;
        }
    }
}
```

---

## ☸️ Step 2: Create Kubernetes Manifests

### Backend Deployment

Create `kubernetes/backend-deployment.yml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: APP_NAME-backend
  labels:
    app: APP_NAME-backend
spec:
  replicas: 1
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 0
      maxUnavailable: 1
  selector:
    matchLabels:
      app: APP_NAME-backend
  template:
    metadata:
      labels:
        app: APP_NAME-backend
    spec:
      containers:
      - name: backend
        image: DOCKERHUB_USERNAME/APP_NAME-backend:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 4001  # YOUR_BACKEND_PORT
        env:
        - name: NODE_ENV
          value: "production"
        - name: PORT
          value: "4001"  # YOUR_BACKEND_PORT
        - name: FRONTEND_URL
          value: "https://FRONTEND_DOMAIN"
        # Add all your environment variables from secrets
        - name: MONGO_URI
          valueFrom:
            secretKeyRef:
              name: APP_NAME-secrets
              key: MONGO_URI
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: APP_NAME-secrets
              key: JWT_SECRET
        # Add other secrets as needed
        # Resources for backend: --requests=cpu=75m,memory=128Mi  --limits=cpu=500m,memory=512Mi
        # Repources for frontend: --requests=cpu=25m,memory=64Mi   --limits=cpu=250m,memory=256Mi
        resources:
          requests:
            memory: "128Mi"
            cpu: "75m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /api/health  # YOUR_HEALTH_ENDPOINT
            port: 4001  # YOUR_BACKEND_PORT
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /api/health  # YOUR_HEALTH_ENDPOINT
            port: 4001  # YOUR_BACKEND_PORT
          initialDelaySeconds: 5
          periodSeconds: 5
```

**Replace:**
- `APP_NAME` → your app name (e.g., `myapp`, `fitverse`)
- `DOCKERHUB_USERNAME` → your Docker Hub username
- `4001` → your backend port
- `FRONTEND_DOMAIN` → your frontend domain
- Add/remove environment variables as needed

### Backend Service

Create `kubernetes/backend-service.yml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: APP_NAME-backend
spec:
  selector:
    app: APP_NAME-backend
  ports:
  - protocol: TCP
    port: 4001  # YOUR_BACKEND_PORT
    targetPort: 4001  # YOUR_BACKEND_PORT
  type: ClusterIP
```

### Frontend Deployment

Create `kubernetes/frontend-deployment.yml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: APP_NAME-frontend
  labels:
    app: APP_NAME-frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: APP_NAME-frontend
  template:
    metadata:
      labels:
        app: APP_NAME-frontend
    spec:
      containers:
      - name: frontend
        image: DOCKERHUB_USERNAME/APP_NAME-frontend:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "100m"
        livenessProbe:
          httpGet:
            path: /
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
```

### Frontend Service

Create `kubernetes/frontend-service.yml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: APP_NAME-frontend
spec:
  selector:
    app: APP_NAME-frontend
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
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

**Option A: Separate subdomains for frontend and backend**

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
  # Frontend - app.yourdomain.com
  - host: app.yourdomain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: APP_NAME-frontend
            port:
              number: 80
  # Frontend - www.app.yourdomain.com
  - host: www.app.yourdomain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: APP_NAME-frontend
            port:
              number: 80
  # Backend API - api.app.yourdomain.com
  - host: api.app.yourdomain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: APP_NAME-backend
            port:
              number: 4001  # YOUR_BACKEND_PORT
  # Backend API - www.api.app.yourdomain.com
  - host: www.api.app.yourdomain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: APP_NAME-backend
            port:
              number: 4001  # YOUR_BACKEND_PORT
  tls:
  - hosts:
    - app.yourdomain.com
    - www.app.yourdomain.com
    - api.app.yourdomain.com
    - www.api.app.yourdomain.com
    secretName: APP_NAME-tls
```

**Option B: Same domain with /api path for backend**

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
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: APP_NAME-backend
            port:
              number: 4001  # YOUR_BACKEND_PORT
      - path: /
        pathType: Prefix
        backend:
          service:
            name: APP_NAME-frontend
            port:
              number: 80
  - host: www.yourdomain.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: APP_NAME-backend
            port:
              number: 4001  # YOUR_BACKEND_PORT
      - path: /
        pathType: Prefix
        backend:
          service:
            name: APP_NAME-frontend
            port:
              number: 80
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
    - uses: actions/checkout@v6
    
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
    - uses: actions/checkout@v6
    
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3
    
    - name: Log in to Docker Hub
      uses: docker/login-action@v3
      with:
        username: ${{ secrets.DOCKERHUB_USERNAME }}
        password: ${{ secrets.DOCKERHUB_TOKEN }}
    
    - name: Extract metadata for backend
      id: meta-backend
      uses: docker/metadata-action@v5
      with:
        images: ${{ secrets.DOCKERHUB_USERNAME }}/APP_NAME-backend
        tags: |
          type=sha,prefix={{branch}}-
          type=raw,value=latest,enable={{is_default_branch}}
    
    - name: Build and push backend
      uses: docker/build-push-action@v6
      with:
        context: ./backend
        push: true
        tags: ${{ steps.meta-backend.outputs.tags }}
        labels: ${{ steps.meta-backend.outputs.labels }}
        cache-from: type=registry,ref=${{ secrets.DOCKERHUB_USERNAME }}/APP_NAME-backend:buildcache
        cache-to: type=registry,ref=${{ secrets.DOCKERHUB_USERNAME }}/APP_NAME-backend:buildcache,mode=max
    
    - name: Extract metadata for frontend
      id: meta-frontend
      uses: docker/metadata-action@v5
      with:
        images: ${{ secrets.DOCKERHUB_USERNAME }}/APP_NAME-frontend
        tags: |
          type=sha,prefix={{branch}}-
          type=raw,value=latest,enable={{is_default_branch}}
    
    - name: Build and push frontend
      uses: docker/build-push-action@v6
      with:
        context: ./frontend
        push: true
        tags: ${{ steps.meta-frontend.outputs.tags }}
        labels: ${{ steps.meta-frontend.outputs.labels }}
        cache-from: type=registry,ref=${{ secrets.DOCKERHUB_USERNAME }}/APP_NAME-frontend:buildcache
        cache-to: type=registry,ref=${{ secrets.DOCKERHUB_USERNAME }}/APP_NAME-frontend:buildcache,mode=max

  deploy:
    runs-on: ubuntu-latest
    needs: [build-and-push]
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    steps:
    - uses: actions/checkout@v6
    
    - name: Install kubectl
      uses: azure/setup-kubectl@v4
      with:
        version: 'latest'
    
    - name: Set up kubeconfig
      run: |
        mkdir -p $HOME/.kube
        echo "${{ secrets.KUBECONFIG }}" | base64 -d > $HOME/.kube/config
        chmod 600 $HOME/.kube/config
    
    - name: Set image tags
      id: image-tags
      run: |
        echo "IMAGE_TAG=main-$(git rev-parse --short HEAD)" >> $GITHUB_OUTPUT
    
    - name: Create or update Kubernetes secrets
      run: |
        # Delete existing secret if it exists (ignore errors if it doesn't)
        kubectl delete secret APP_NAME-secrets --ignore-not-found=true
        
        # Create new secret from GitHub Actions secrets
        kubectl create secret generic APP_NAME-secrets \
          --from-literal=MONGO_URI="${{ secrets.MONGO_URI }}" \
          --from-literal=JWT_SECRET="${{ secrets.JWT_SECRET }}"
          # Add all your other secrets here with --from-literal=KEY="${{ secrets.KEY }}"
    
    - name: Deploy to Kubernetes
      run: |
        kubectl apply -f kubernetes/cluster-issuer-staging.yml
        kubectl apply -f kubernetes/cluster-issuer-prod.yml
        kubectl apply -f kubernetes/backend-deployment.yml
        kubectl apply -f kubernetes/backend-service.yml
        kubectl apply -f kubernetes/frontend-deployment.yml
        kubectl apply -f kubernetes/frontend-service.yml
        kubectl apply -f kubernetes/ingress.yml
    
    - name: Update deployments with new image tags
      run: |
        kubectl set image deployment/APP_NAME-backend backend=${{ secrets.DOCKERHUB_USERNAME }}/APP_NAME-backend:${{ steps.image-tags.outputs.IMAGE_TAG }}
        kubectl set image deployment/APP_NAME-frontend frontend=${{ secrets.DOCKERHUB_USERNAME }}/APP_NAME-frontend:${{ steps.image-tags.outputs.IMAGE_TAG }}
    
    - name: Wait for rollout to complete
      run: |
        kubectl rollout status deployment/APP_NAME-backend --timeout=5m
        kubectl rollout status deployment/APP_NAME-frontend --timeout=5m
    
    - name: Verify deployment
      run: |
        kubectl get pods -l app=APP_NAME-backend
        kubectl get pods -l app=APP_NAME-frontend
        kubectl get ingress APP_NAME-ingress
```

**Replace:**
- `APP_NAME` → your app name (in all places)
- Add your specific secrets in the "Create or update Kubernetes secrets" step

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
   - MONGO_URI
   - JWT_SECRET
   - CLOUDINARY_CLOUD_NAME
   - CLOUDINARY_API_KEY
   - CLOUDINARY_API_SECRET
   - Any API keys, database URLs, etc.

---

## 🌐 Step 6: Configure DNS

Add DNS A records pointing to your VPS IP:

For separate subdomains:
- `app.yourdomain.com` → `YOUR_VPS_IP`
- `www.app.yourdomain.com` → `YOUR_VPS_IP`
- `api.app.yourdomain.com` → `YOUR_VPS_IP`
- `www.api.app.yourdomain.com` → `YOUR_VPS_IP`

Or for same domain:
- `yourdomain.com` → `YOUR_VPS_IP`
- `www.yourdomain.com` → `YOUR_VPS_IP`

Wait for DNS propagation (can take 5 mins to 24 hours).

---

## 🗄️ Step 7: Database Configuration

### MongoDB Atlas

1. Go to MongoDB Atlas → Network Access
2. Add your VPS IP address:
   ```bash
   # Get VPS IP
   curl -4 ifconfig.me
   ```
3. Or allow access from anywhere: `0.0.0.0/0` (less secure)
4. Save your MongoDB connection string as `MONGO_URI` GitHub secret

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
kubectl logs -l app=APP_NAME-backend
kubectl logs -l app=APP_NAME-frontend

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
# - Wrong environment variables
# - Database connection failed
# - Port mismatch
# - Missing secrets
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

# Check if images exist
docker pull DOCKERHUB_USERNAME/APP_NAME-backend:latest

# Verify DOCKERHUB_USERNAME and DOCKERHUB_TOKEN secrets
```

---

## 📝 Customization Checklist

When adapting this for a new app:

- [ ] Replace `APP_NAME` everywhere
- [ ] Update Docker Hub username
- [ ] Change backend port (4001 → your port)
- [ ] Update health check endpoints
- [ ] Update domain names
- [ ] Adjust environment variables
- [ ] Update resource limits based on app needs
- [ ] Add/remove secrets as needed
- [ ] Update build output directory for frontend
- [ ] Verify npm scripts (`npm run build`, etc.)
- [ ] Update email for SSL certificates

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
