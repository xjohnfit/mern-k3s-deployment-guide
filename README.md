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

These manifests are **templates** — every project-specific value is an `${VAR}` placeholder, not literal text. The pipeline in Step 3 renders them with `envsubst` at deploy time, substituting values from your repo's GitHub Actions **Variables** and **Secrets** (Step 5). You copy these files byte-for-byte into every project; you never hand-edit them per project.

### Backend Deployment

Create `kubernetes/backend-deployment.yml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${APP_NAME}-backend
  labels:
    app: ${APP_NAME}-backend
spec:
  replicas: 1
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 0
      maxUnavailable: 1
  selector:
    matchLabels:
      app: ${APP_NAME}-backend
  template:
    metadata:
      labels:
        app: ${APP_NAME}-backend
    spec:
      containers:
      - name: backend
        image: ${DOCKERHUB_USERNAME}/${APP_NAME}-backend:${IMAGE_TAG}
        imagePullPolicy: Always
        ports:
        - containerPort: ${BACKEND_PORT}
        env:
        - name: NODE_ENV
          value: "production"
        - name: PORT
          value: "${BACKEND_PORT}"
        - name: FRONTEND_URL
          value: "https://${FRONTEND_DOMAIN}"
        # Add all your app-specific environment variables from secrets here
        - name: MONGO_URI
          valueFrom:
            secretKeyRef:
              name: ${APP_NAME}-secrets
              key: MONGO_URI
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: ${APP_NAME}-secrets
              key: JWT_SECRET
        # Add other secrets as needed — keep the pattern above
        resources:
          requests:
            memory: "128Mi"
            cpu: "75m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: ${HEALTH_CHECK_PATH}
            port: ${BACKEND_PORT}
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: ${HEALTH_CHECK_PATH}
            port: ${BACKEND_PORT}
          initialDelaySeconds: 5
          periodSeconds: 5
```

The only thing you might still hand-edit here is the env var list itself (`MONGO_URI`, `JWT_SECRET`, ...) since every project needs a different set of app secrets — everything else (`APP_NAME`, `DOCKERHUB_USERNAME`, `BACKEND_PORT`, `FRONTEND_DOMAIN`, `HEALTH_CHECK_PATH`, `IMAGE_TAG`) is filled in automatically at deploy time.

### Backend Service

Create `kubernetes/backend-service.yml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: ${APP_NAME}-backend
spec:
  selector:
    app: ${APP_NAME}-backend
  ports:
  - protocol: TCP
    port: ${BACKEND_PORT}
    targetPort: ${BACKEND_PORT}
  type: ClusterIP
```

### Frontend Deployment

Create `kubernetes/frontend-deployment.yml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${APP_NAME}-frontend
  labels:
    app: ${APP_NAME}-frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ${APP_NAME}-frontend
  template:
    metadata:
      labels:
        app: ${APP_NAME}-frontend
    spec:
      containers:
      - name: frontend
        image: ${DOCKERHUB_USERNAME}/${APP_NAME}-frontend:${IMAGE_TAG}
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

Frontend container port (8080) and probe path (`/`) are left as fixed values on purpose — they're a convention of this guide's nginx setup (Step 1), not something that varies per project.

### Frontend Service

Create `kubernetes/frontend-service.yml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: ${APP_NAME}-frontend
spec:
  selector:
    app: ${APP_NAME}-frontend
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
    email: ${LETSENCRYPT_EMAIL}
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
    email: ${LETSENCRYPT_EMAIL}
    privateKeySecretRef:
      name: letsencrypt-staging
    solvers:
    - http01:
        ingress:
          class: traefik
```

### Ingress (Routing & SSL)

Create `kubernetes/ingress.yml`. Pick **one** of the two options below — both are fully templated, so switching projects between them is just a matter of which repo Variables you set (Step 5).

**Option A: Separate subdomains for frontend and backend**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ${APP_NAME}-ingress
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: web,websecure
    traefik.ingress.kubernetes.io/router.tls: "true"
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: traefik
  rules:
  # Frontend - e.g. app.yourdomain.com
  - host: ${FRONTEND_DOMAIN}
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: ${APP_NAME}-frontend
            port:
              number: 80
  # Frontend - e.g. www.app.yourdomain.com
  - host: www.${FRONTEND_DOMAIN}
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: ${APP_NAME}-frontend
            port:
              number: 80
  # Backend API - e.g. api.app.yourdomain.com
  - host: ${BACKEND_DOMAIN}
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: ${APP_NAME}-backend
            port:
              number: ${BACKEND_PORT}
  # Backend API - e.g. www.api.app.yourdomain.com
  - host: www.${BACKEND_DOMAIN}
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: ${APP_NAME}-backend
            port:
              number: ${BACKEND_PORT}
  tls:
  - hosts:
    - ${FRONTEND_DOMAIN}
    - www.${FRONTEND_DOMAIN}
    - ${BACKEND_DOMAIN}
    - www.${BACKEND_DOMAIN}
    secretName: ${APP_NAME}-tls
```

**Option B: Same domain with /api path for backend**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ${APP_NAME}-ingress
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: web,websecure
    traefik.ingress.kubernetes.io/router.tls: "true"
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: traefik
  rules:
  - host: ${DOMAIN}
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: ${APP_NAME}-backend
            port:
              number: ${BACKEND_PORT}
      - path: /
        pathType: Prefix
        backend:
          service:
            name: ${APP_NAME}-frontend
            port:
              number: 80
  - host: www.${DOMAIN}
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: ${APP_NAME}-backend
            port:
              number: ${BACKEND_PORT}
      - path: /
        pathType: Prefix
        backend:
          service:
            name: ${APP_NAME}-frontend
            port:
              number: 80
  tls:
  - hosts:
    - ${DOMAIN}
    - www.${DOMAIN}
    secretName: ${APP_NAME}-tls
```

Option A needs the `FRONTEND_DOMAIN` and `BACKEND_DOMAIN` repo Variables set; Option B needs just `DOMAIN` instead. Don't set both — only the variables your chosen option references need to exist.

---

## 🚀 Step 3: Create GitHub Actions Workflow

### Design notes

This pipeline was hardened after comparing several near-identical variants of it that had drifted apart across different projects. If you're copying an older version of this workflow, watch for these problems it fixes:

- **Secret name typos** — some variants read `secrets.KUBE_CONFIG` while the docs said to create `KUBECONFIG`. Pick one name (`KUBECONFIG`) and use it everywhere, including in Step 5 below.
- **`paths-ignore: kubernetes/**`** — excluding manifest changes from the trigger means a change to a Deployment/Ingress YAML never gets applied, because the whole workflow (including the `deploy` job) is skipped. Only docs/ignore files should be excluded.
- **Delete-then-recreate secrets** — `kubectl delete secret` followed by `kubectl create` leaves a window where the Secret doesn't exist. Use `--dry-run=client -o yaml | kubectl apply -f -` instead: it updates keys in place with no gap.
- **Floating action tags** (`@master`, `version: latest`) — pin to a specific release so a third-party update can't silently break your pipeline.
- **No test gate** — scanning and deploying without ever running the app's own build/lint/test step means a broken app can still ship.
- **Scanning only source, not the image you actually ship** — a filesystem scan misses vulnerabilities baked into the final container (base image OS packages, etc.).
- **No deploy approval gate** — anyone who can push to `main` gets an unreviewed production deploy using a kubeconfig with full cluster-admin. A GitHub `environment` with required reviewers fixes this.
- **Non-secret values stored as secrets** — `VITE_API_URL` ends up in your public frontend bundle anyway, so it belongs in repository **Variables**, not **Secrets**.

This version goes one step further: it's written to be **identical across every project**. `APP_NAME`, ports, domains, and the health-check path are never hardcoded in the workflow or the manifests — they're read from GitHub Actions **Variables** at deploy time, and the Kubernetes manifests are rendered with `envsubst` right before `kubectl apply`. When you start a new project, you copy this file and the manifests from Step 2 unchanged, then just fill in that project's Variables/Secrets (Step 5).

Create `.github/workflows/deploy.yml`:

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [ main ]
    paths-ignore:
      - '**.md'
      - '.gitignore'
  pull_request:
    branches: [ main ]
  workflow_dispatch:

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

permissions:
  contents: read

jobs:
  # ---- Catch known-vulnerable dependencies before they're merged ----
  dependency-review:
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    timeout-minutes: 5
    permissions:
      contents: read
      pull-requests: write
    steps:
      - uses: actions/checkout@v6
      - uses: actions/dependency-review-action@v4
        with:
          fail-on-severity: high

  # ---- Fail fast: don't scan or build an app that doesn't build/pass ----
  test-backend:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    defaults:
      run:
        working-directory: backend
    steps:
      - uses: actions/checkout@v6
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
          cache-dependency-path: backend/package-lock.json
      - run: npm ci
      - run: npm run lint --if-present
      - run: npm test --if-present
      - run: npm run build --if-present

  test-frontend:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    defaults:
      run:
        working-directory: frontend
    steps:
      - uses: actions/checkout@v6
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
          cache-dependency-path: frontend/package-lock.json
      - run: npm ci
      - run: npm run lint --if-present
      - run: npm test --if-present
      - run: npm run build --if-present

  # ---- Source-level scan: secrets, IaC misconfig, known-vulnerable deps ----
  trivy-fs-scan:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    permissions:
      contents: read
      security-events: write
    steps:
      - uses: actions/checkout@v6

      - name: Run Trivy filesystem scan
        uses: aquasecurity/trivy-action@0.28.0
        with:
          scan-type: 'fs'
          scan-ref: '.'
          format: 'sarif'
          output: 'trivy-fs-results.sarif'
          severity: 'CRITICAL,HIGH,MEDIUM'
          scanners: 'vuln,secret,misconfig'
          exit-code: '0' # advisory here; findings land in the Security tab instead of a table nobody reads

      - name: Upload results to the Security tab
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: 'trivy-fs-results.sarif'

  # ---- Build & push images (main branch only) ----
  build-and-push:
    runs-on: ubuntu-latest
    timeout-minutes: 20
    needs: [test-backend, test-frontend, trivy-fs-scan]
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    env:
      BACKEND_IMAGE: ${{ secrets.DOCKERHUB_USERNAME }}/${{ vars.APP_NAME }}-backend
      FRONTEND_IMAGE: ${{ secrets.DOCKERHUB_USERNAME }}/${{ vars.APP_NAME }}-frontend
    outputs:
      image-tag: ${{ steps.image-tag.outputs.IMAGE_TAG }}
    steps:
      - uses: actions/checkout@v6

      - uses: docker/setup-buildx-action@v3

      - uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Compute image tag
        id: image-tag
        run: echo "IMAGE_TAG=main-$(git rev-parse --short HEAD)" >> "$GITHUB_OUTPUT"

      - name: Extract metadata for backend
        id: meta-backend
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.BACKEND_IMAGE }}
          tags: |
            type=raw,value=${{ steps.image-tag.outputs.IMAGE_TAG }}
            type=raw,value=latest,enable={{is_default_branch}}

      - name: Build and push backend
        uses: docker/build-push-action@v6
        with:
          context: ./backend
          push: true
          provenance: true
          sbom: true
          tags: ${{ steps.meta-backend.outputs.tags }}
          labels: ${{ steps.meta-backend.outputs.labels }}
          cache-from: type=registry,ref=${{ env.BACKEND_IMAGE }}:buildcache
          cache-to: type=registry,ref=${{ env.BACKEND_IMAGE }}:buildcache,mode=max

      - name: Extract metadata for frontend
        id: meta-frontend
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.FRONTEND_IMAGE }}
          tags: |
            type=raw,value=${{ steps.image-tag.outputs.IMAGE_TAG }}
            type=raw,value=latest,enable={{is_default_branch}}

      - name: Build and push frontend
        uses: docker/build-push-action@v6
        with:
          context: ./frontend
          push: true
          provenance: true
          sbom: true
          build-args: |
            VITE_API_URL=${{ vars.VITE_API_URL }}
          tags: ${{ steps.meta-frontend.outputs.tags }}
          labels: ${{ steps.meta-frontend.outputs.labels }}
          cache-from: type=registry,ref=${{ env.FRONTEND_IMAGE }}:buildcache
          cache-to: type=registry,ref=${{ env.FRONTEND_IMAGE }}:buildcache,mode=max

  # ---- Gate: block deploy if the image you're about to ship has a critical, fixable CVE ----
  scan-images:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    needs: [build-and-push]
    env:
      BACKEND_IMAGE: ${{ secrets.DOCKERHUB_USERNAME }}/${{ vars.APP_NAME }}-backend
      FRONTEND_IMAGE: ${{ secrets.DOCKERHUB_USERNAME }}/${{ vars.APP_NAME }}-frontend
    steps:
      - name: Scan backend image
        uses: aquasecurity/trivy-action@0.28.0
        with:
          image-ref: ${{ env.BACKEND_IMAGE }}:${{ needs.build-and-push.outputs.image-tag }}
          format: 'table'
          severity: 'CRITICAL'
          exit-code: '1'
          ignore-unfixed: true

      - name: Scan frontend image
        uses: aquasecurity/trivy-action@0.28.0
        with:
          image-ref: ${{ env.FRONTEND_IMAGE }}:${{ needs.build-and-push.outputs.image-tag }}
          format: 'table'
          severity: 'CRITICAL'
          exit-code: '1'
          ignore-unfixed: true

  # ---- Deploy: gated behind a GitHub Environment with required reviewers ----
  deploy:
    runs-on: ubuntu-latest
    timeout-minutes: 15
    needs: [build-and-push, scan-images]
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    environment: production
    env:
      APP_NAME: ${{ vars.APP_NAME }}
    steps:
      - uses: actions/checkout@v6

      - name: Install kubectl
        uses: azure/setup-kubectl@v4
        with:
          version: 'v1.30.0' # pin to match your K3s cluster's Kubernetes version

      - name: Set up kubeconfig
        run: |
          mkdir -p "$HOME/.kube"
          echo "${{ secrets.KUBECONFIG }}" | base64 -d > "$HOME/.kube/config"
          chmod 600 "$HOME/.kube/config"

      - name: Render Kubernetes manifests
        env:
          DOCKERHUB_USERNAME: ${{ secrets.DOCKERHUB_USERNAME }}
          IMAGE_TAG: ${{ needs.build-and-push.outputs.image-tag }}
          BACKEND_PORT: ${{ vars.BACKEND_PORT }}
          FRONTEND_DOMAIN: ${{ vars.FRONTEND_DOMAIN }}
          BACKEND_DOMAIN: ${{ vars.BACKEND_DOMAIN }}
          DOMAIN: ${{ vars.DOMAIN }}
          HEALTH_CHECK_PATH: ${{ vars.HEALTH_CHECK_PATH }}
          LETSENCRYPT_EMAIL: ${{ vars.LETSENCRYPT_EMAIL }}
        run: |
          # Fail fast with a clear message instead of deploying with blank values.
          : "${APP_NAME:?Set the APP_NAME repository variable}"
          : "${BACKEND_PORT:?Set the BACKEND_PORT repository variable}"
          : "${HEALTH_CHECK_PATH:?Set the HEALTH_CHECK_PATH repository variable}"
          : "${LETSENCRYPT_EMAIL:?Set the LETSENCRYPT_EMAIL repository variable}"

          mkdir -p rendered
          for f in kubernetes/*.yml; do
            envsubst < "$f" > "rendered/$(basename "$f")"
          done

      - name: Create or update Kubernetes secrets
        run: |
          kubectl create secret generic "$APP_NAME-secrets" \
            --from-literal=MONGO_URI="${{ secrets.MONGO_URI }}" \
            --from-literal=JWT_SECRET="${{ secrets.JWT_SECRET }}" \
            --dry-run=client -o yaml | kubectl apply -f -
          # Add your other app secrets the same way. Server-side apply updates
          # keys in place instead of deleting and recreating the Secret object.

      - name: Deploy to Kubernetes
        run: |
          kubectl apply -f rendered/cluster-issuer-staging.yml
          kubectl apply -f rendered/cluster-issuer-prod.yml
          kubectl apply -f rendered/backend-deployment.yml
          kubectl apply -f rendered/backend-service.yml
          kubectl apply -f rendered/frontend-deployment.yml
          kubectl apply -f rendered/frontend-service.yml
          kubectl apply -f rendered/ingress.yml

      - name: Wait for rollout, roll back automatically on failure
        run: |
          kubectl rollout status "deployment/$APP_NAME-backend" --timeout=5m || { kubectl rollout undo "deployment/$APP_NAME-backend"; exit 1; }
          kubectl rollout status "deployment/$APP_NAME-frontend" --timeout=5m || { kubectl rollout undo "deployment/$APP_NAME-frontend"; exit 1; }

      - name: Verify deployment
        run: |
          kubectl get pods -l "app=$APP_NAME-backend"
          kubectl get pods -l "app=$APP_NAME-frontend"
          kubectl get ingress "$APP_NAME-ingress"
```

**Nothing in this file needs to change between projects.** Set the repo Variables/Secrets in Step 5 and this workflow works as-is. The one line you may still want to touch:

- `v1.30.0` (kubectl version) → whatever version your K3s cluster runs (`kubectl version` on the VPS)

Because the image tag is baked directly into the rendered Deployment manifests, `kubectl apply` on the rendered files triggers the rolling update by itself — there's no separate `kubectl set image` step to keep in sync.

**One-time repo setup this pipeline expects:**

- Settings → Environments → create an environment named `production` and add required reviewers (this is what turns `deploy` into an approval-gated step).
- Settings → Secrets and variables → Actions → **Variables** tab → set `APP_NAME`, `BACKEND_PORT`, `HEALTH_CHECK_PATH`, `LETSENCRYPT_EMAIL`, `VITE_API_URL`, and either (`FRONTEND_DOMAIN` + `BACKEND_DOMAIN`) or `DOMAIN` depending on which `ingress.yml` option you kept.

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

## 🔐 Step 5: Configure GitHub Secrets & Variables

Go to your GitHub repository → Settings → Secrets and variables → Actions.

This is the **only** thing that changes between projects — the workflow (Step 3) and manifests (Step 2) are identical every time. Set these once per new repo and you're done.

### Required Secrets (Secrets tab → New repository secret)

Use these exact names — the workflow in Step 3 reads them literally, and a typo (e.g. `KUBE_CONFIG` instead of `KUBECONFIG`) fails silently at deploy time rather than at edit time.

1. **DOCKERHUB_USERNAME** - Your Docker Hub username
2. **DOCKERHUB_TOKEN** - Docker Hub access token (create at hub.docker.com → Account Settings → Security)
3. **KUBECONFIG** - Base64 encoded kubeconfig from Step 4
4. **MONGO_URI**, **JWT_SECRET** - Read directly by the `deploy` job's "Create or update Kubernetes secrets" step
5. **Any other app-specific secrets** (examples): CLOUDINARY_CLOUD_NAME, CLOUDINARY_API_KEY, CLOUDINARY_API_SECRET, or any other API key/connection string — add a matching `--from-literal` line to that same step

For anything that would be sensitive if leaked (API keys, connection strings, tokens), use a **Secret**. If it's only ever going to end up embedded in the built frontend bundle anyway (like `VITE_API_URL`), it isn't actually secret — put it in Variables instead so it isn't masked/mistaken for something sensitive in logs.

> `DOCKERHUB_USERNAME` and `DOCKERHUB_TOKEN` are almost always the same across all your projects. If your repos live under one GitHub organization, set these once as **Organization secrets** (Organization Settings → Secrets and variables → Actions) instead of re-entering them per repo.

### Required Variables (Variables tab → New repository variable)

1. **APP_NAME** - Your app name, used to build image names and Kubernetes object names (e.g. `fitverse`)
2. **BACKEND_PORT** - The port your backend listens on (e.g. `4001`)
3. **HEALTH_CHECK_PATH** - Backend health endpoint used by the liveness/readiness probes (e.g. `/api/health`)
4. **LETSENCRYPT_EMAIL** - Email address used for SSL certificate registration
5. **VITE_API_URL** - Public URL the frontend calls (e.g. `https://api.app.yourdomain.com`)
6. Either:
   - **FRONTEND_DOMAIN** and **BACKEND_DOMAIN** - if you kept ingress.yml Option A (separate subdomains), or
   - **DOMAIN** - if you kept ingress.yml Option B (single domain, `/api` path)

The `deploy` job's render step fails immediately with a clear message if `APP_NAME`, `BACKEND_PORT`, `HEALTH_CHECK_PATH`, or `LETSENCRYPT_EMAIL` are missing — so a forgotten variable shows up as a red pipeline run, not a broken deploy.

### Recommended: scope secrets to the `production` environment

The `deploy` job in Step 3 targets `environment: production`. Once you create that environment (Settings → Environments → New environment → `production`) and add required reviewers, you can optionally move `KUBECONFIG` and your database/app secrets from repository-level secrets into that environment's own secrets. That way a compromised PR workflow (which never touches the `production` environment) can't read your cluster credentials at all — only the reviewed `deploy` job can.

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

Every manual `kubectl` snippet below (and in Troubleshooting/Monitoring/Common Commands) uses `$APP_NAME` instead of a hardcoded name. Set it once per terminal session and every command works as-is, regardless of which project you're operating on:

```bash
export APP_NAME=your-app-name
```

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
3. If this is a pull request, `dependency-review` also runs
4. On a push to `main`, these run in order: `test-backend` / `test-frontend` → `trivy-fs-scan` → `build-and-push` → `scan-images` → `deploy` (paused for approval if you set up required reviewers on the `production` environment)

### Check Deployment on VPS

```bash
# Check pods
kubectl get pods

# View logs
kubectl logs -l app=$APP_NAME-backend
kubectl logs -l app=$APP_NAME-frontend

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

## 📝 Customization Checklist

The workflow and Kubernetes manifests never need editing between projects — everything project-specific lives in GitHub Secrets/Variables (Step 5). For a new app:

- [ ] Set `APP_NAME`, `BACKEND_PORT`, `HEALTH_CHECK_PATH`, `LETSENCRYPT_EMAIL`, `VITE_API_URL`, and domain variable(s) (Step 5)
- [ ] Set `DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN`, `KUBECONFIG`, and app-specific secrets (Step 5) — reuse org-level secrets if `DOCKERHUB_USERNAME`/`DOCKERHUB_TOKEN` are the same as your other projects
- [ ] Add any app-specific `--from-literal` lines to the "Create or update Kubernetes secrets" step, and matching `secretKeyRef` entries in `backend-deployment.yml`
- [ ] Adjust CPU/memory resource limits in the manifests if this app's footprint differs from the defaults
- [ ] Update the backend port and health check path in the Dockerfile (Step 1) to match the `BACKEND_PORT`/`HEALTH_CHECK_PATH` variables
- [ ] Update the frontend build output directory in the Dockerfile if it's not `dist` (e.g. Create React App uses `build`)
- [ ] Create the `production` GitHub Environment with required reviewers (Step 5)

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

### CI/CD-specific

1. **Pin third-party actions** - Never use `@master`/`@latest` for actions like `trivy-action` or `setup-kubectl`; pin to a release tag so an upstream change can't silently break or compromise your pipeline
2. **Gate on tests, not just scans** - A pipeline that scans and deploys but never runs the app's own build/lint/test step will happily ship a broken app
3. **Scan the image you ship, not just the source tree** - a filesystem scan misses vulnerabilities baked into the final container; scan the built image too, and let it actually fail the build on CRITICAL findings
4. **Require review for production deploys** - use a GitHub `environment` with required reviewers on the deploy job, don't let a bare push to `main` trigger a deploy that skipped review, using cluster-admin credentials
5. **Never let `paths-ignore` exclude your Kubernetes manifests** - excluding `kubernetes/**` means a manifest change never triggers a deploy at all
6. **Keep secret names consistent everywhere** - a workflow that reads `secrets.KUBE_CONFIG` while your docs say to create `KUBECONFIG` fails at deploy time, not at review time

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

# Port forward for debugging (replace 4001 with your BACKEND_PORT)
kubectl port-forward service/$APP_NAME-backend 4001:4001

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
