# Setup Guide

Step-by-step instructions to stand up a new MERN project on this pipeline, from Docker files through first deploy. See [../README.md](../README.md) for prerequisites and project structure, [pipeline-design.md](pipeline-design.md) for the reasoning behind how the pipeline is built, and [operations.md](operations.md) for day-2 troubleshooting/monitoring/common commands.

---

## 🐳 Step 1: Create Docker Files

> **Node version — don't just trust the default.** The Dockerfiles and CI workflow below default to Node 22. Treat that as a starting point, not a given: run `npm ci` for each service locally (backend and frontend, separately — they frequently need different Node versions from each other) and read its output for `EBADENGINE` warnings. Those tell you exactly what version that service's own dependencies expect. Match the Dockerfile's `FROM node:X-alpine` **and** the CI workflow's `setup-node` `node-version` to that, per service — copying the same number into both places without checking is how you end up with a container that builds fine locally (different Node on your machine) but breaks in CI or at runtime. Also worth knowing: GitHub-hosted runners are in the process of deprecating Node 20 — workflows that request `node-version: '20'` already print a deprecation warning, and runner images have started silently substituting Node 24 instead. Node 20 is not a safe default for new projects for that reason alone; this guide uses 22 as a template default, but bump it further if `EBADENGINE` says to.
>
> **Static SPA vs. SSR frontend — pick your path now.** Everything in this Step 1 (and the frontend K8s manifests in Step 2) assumes a static single-page app: Vite builds plain HTML/JS/CSS, nginx serves the files, client-side routing handles navigation. If your frontend does server-side rendering instead (React Router v8 SSR, Remix, Next.js, etc.), that model doesn't apply — loaders/actions need a live Node process per request, not a directory of static files. Follow the static-SPA Dockerfile/nginx.conf below if that's what you have; if you're on SSR, skip ahead to the "Fork point: SSR frontend" callout after the nginx.conf section for the Node-based alternative.

### Backend Dockerfile

Create `backend/Dockerfile`:

```dockerfile
# Build stage
FROM node:22-alpine AS builder

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
FROM node:22-alpine

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

# node:*-alpine ships a global npm install, and that npm bundles its own
# internal copy of the `tar` package — separate from anything in your
# package-lock.json. When that bundled tar has a CRITICAL CVE (npm ships new
# versions on its own schedule, independent of the app), it trips the
# scan-images Trivy gate in CI even though it has nothing to do with your
# app's own dependencies — `npm ls` won't show it, which is what makes it
# confusing to debug the first time you hit it. The container only ever runs
# `node dist/index.js` (see CMD below), never `npm` itself, so the fix is to
# delete npm/npx/corepack from the final image entirely rather than try to
# patch or ignore the CVE.
RUN rm -rf /usr/local/lib/node_modules/npm \
           /usr/local/lib/node_modules/corepack \
           /usr/local/bin/npm \
           /usr/local/bin/npx \
           /usr/local/bin/corepack

# Copy built application from builder stage
COPY --from=builder --chown=nodejs:nodejs /app/dist ./dist

# Switch to non-root user
USER nodejs

# Expose port (use your backend port)
EXPOSE 5000

# Set environment to production
ENV NODE_ENV=production

# Health check (adjust path and port as needed)
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD node -e "require('http').get('http://localhost:5000/api/health', (r) => {process.exit(r.statusCode === 200 ? 0 : 1)})"

# Use dumb-init to handle signals properly
ENTRYPOINT ["dumb-init", "--"]

# Start the application
CMD ["node", "dist/index.js"]
```

**Adjust:**
- Port number (5000 → your port)
- Health check path (`/api/health`)
- Entry file (`dist/index.js`)

### Frontend Dockerfile

Create `frontend/Dockerfile`:

```dockerfile
# Build stage
FROM node:22-alpine AS builder

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

# Expose port (matches Vite's default dev server port for consistency)
EXPOSE 5173

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD wget --quiet --tries=1 --spider http://localhost:5173/ || exit 1

# Use dumb-init to handle signals properly
ENTRYPOINT ["dumb-init", "--"]

# Start nginx
CMD ["nginx", "-g", "daemon off;"]
```

**Adjust:**
- Build output directory (`/app/dist` → your build folder: might be `/app/build`)

Note this static-SPA production stage is `nginx:alpine`, not a `node:*-alpine` — so it never has npm in it to begin with, and doesn't need the npm/tar removal step used in the backend Dockerfile above. That fix only applies to production stages that are themselves `node:*-alpine` (the backend, and the SSR frontend variant further down).

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
        listen 5173;
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

### ⚠️ Fork point: SSR frontend (React Router v8 SSR, Remix, Next.js, etc.)

Everything above — the Dockerfile, `nginx.conf`, and the frontend K8s manifests in Step 2 — assumes a **static SPA**: Vite produces plain HTML/JS/CSS, nginx serves those files, and `try_files ... /index.html` hands unknown paths to client-side routing. That model has no server-side code running per request, which is exactly what breaks for a **server-rendering frontend**: loaders/actions execute on the server for every request, so you need a live Node process, not a static file server. If that's your project, use this Dockerfile instead of the one above — and skip `nginx.conf` entirely, it doesn't apply.

```dockerfile
# Build stage
FROM node:22-alpine AS builder

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .

# Build the framework's server + client bundles (path depends on the
# framework — this is react-router build's default output layout).
RUN npm run build

# Production stage — node:*-alpine, not nginx:alpine, because we need a
# live Node process to run the SSR server, not just serve static files.
FROM node:22-alpine

WORKDIR /app

RUN apk add --no-cache dumb-init

RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

COPY package*.json ./

# Only production deps — the running server doesn't need the bundler, type
# checker, etc. that devDependencies bring in for the build step above.
RUN npm ci --omit=dev && \
    npm cache clean --force

# Same bundled-tar CVE issue as the backend Dockerfile (see its comment
# above the equivalent step) — this production stage is node:*-alpine too,
# so it ships the same vulnerable npm-bundled tar unless removed. Safe here
# because CMD below invokes node directly rather than going through npm.
RUN rm -rf /usr/local/lib/node_modules/npm \
           /usr/local/lib/node_modules/corepack \
           /usr/local/bin/npm \
           /usr/local/bin/npx \
           /usr/local/bin/corepack

# Copy the built server/client output and the production node_modules the
# server needs to actually run (unlike the static SPA, there's no "just the
# dist folder" here — the SSR server is a real Node app).
COPY --from=builder --chown=nodejs:nodejs /app/build ./build
COPY --from=builder --chown=nodejs:nodejs /app/node_modules ./node_modules
COPY --from=builder --chown=nodejs:nodejs /app/package.json ./package.json

USER nodejs

# SSR frontends listen on whatever port the framework's own PORT env var
# controls — this is NOT the static-SPA template's assumed nginx port 5173.
# react-router-serve defaults to 3000; check your framework's docs and match
# the K8s Service/Deployment ports in Step 2 to whatever you set here.
EXPOSE 3000
ENV NODE_ENV=production
ENV PORT=3000

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD node -e "require('http').get('http://localhost:3000/', (r) => {process.exit(r.statusCode === 200 ? 0 : 1)})"

ENTRYPOINT ["dumb-init", "--"]

# Do NOT use ["npm", "run", "start"] — npm was just deleted from this image
# (see the rm -rf step above), so `npm run start` would fail outright.
# Invoke the framework's own entry point with node directly instead. For
# React Router v8 SSR that's @react-router/serve's CLI; confirm the exact
# path in your own build (`ls node_modules/@react-router/serve/dist`) rather
# than trusting this literally — it will differ by framework and version.
CMD ["node", "node_modules/@react-router/serve/dist/cli.js", "./build/server/index.js"]
```

**The other half of this fork — how the frontend reaches the backend:** the static-SPA template passes `VITE_API_URL` as a **build-time** arg (Step 3) because the browser calls the API directly, and Vite bakes that URL into the JS bundle at build time. An SSR frontend doesn't need that at all — its loaders/actions call the backend from **inside the server**, during rendering, not from the user's browser. So instead of a build-time arg, give the SSR container a **runtime** env var (e.g. `BACKEND_URL`) pointing at the backend's internal ClusterIP Service DNS name (`http://${APP_NAME}-backend:${BACKEND_PORT}`, from the Service created in Step 2) — no public URL, no build-time baking, and it can change without rebuilding the image. Document whichever pattern your project actually uses; don't carry both.

### 🧪 Test your images locally before pushing

This step catches two entire classes of failure before they ever cost you a CI run — worth doing every time you touch a Dockerfile, not just once. It's what surfaced a real env-validation gap on the frontend during this guide's first real deploy (the app started fine even with nonsense config instead of failing loudly), and it's what would have caught the bundled-tar CVE described above before ever pushing to CI.

```bash
# 1. Build both images locally, exactly as CI will.
docker build -t local/backend ./backend
docker build -t local/frontend ./frontend   # or the SSR Dockerfile if you forked above

# 2. Run each with fake-but-correctly-shaped env vars — real variable names,
#    plausible-looking fake values (e.g. a MONGO_URI that points nowhere).
#    You're not testing that the app works end-to-end here; you're testing
#    that bad/missing config fails LOUDLY with a clear log line, rather than
#    hanging silently or crash-looping with no explanation. A container that
#    just sits there with no output on bad config is a debugging trap in
#    production — better to find that out on your own machine.
docker run --rm -e NODE_ENV=production -e PORT=5000 \
  -e MONGO_URI="mongodb://fake:fake@localhost:27017/fake" \
  -e JWT_SECRET="fake-secret-just-for-this-test" \
  -p 5000:5000 local/backend

docker run --rm -e NODE_ENV=production -e PORT=3000 \
  -e BACKEND_URL="http://fake:5000" \
  -p 3000:3000 local/frontend   # env vars here depend on static vs SSR — see the fork above

# 3. Scan the images you just built — not just the source tree, and not just
#    in CI. `trivy image` runs the exact same scanner the scan-images CI job
#    uses, locally, in seconds, against the actual artifact you're about to
#    ship. This is the step that would have caught the bundled-tar CVE
#    (see the backend Dockerfile above) before it ever reached a pipeline
#    run — `docker build` succeeding tells you nothing about known CVEs in
#    the base image or its bundled tooling.
trivy image local/backend
trivy image local/frontend
```

If step 3 requires installing Trivy locally, see the [Trivy installation docs](https://trivy.dev/latest/getting-started/installation/) — it's the same binary the `trivy-action` GitHub Action wraps, so what fails here is what fails in CI, just faster to iterate on.

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
      # This pod never talks to the Kubernetes API (it's a REST/GraphQL
      # backend, not a controller or operator), so the token K8s would
      # otherwise auto-mount into every pod by default is unused attack
      # surface: if the container is ever compromised, there's simply no
      # credential sitting in the filesystem for an attacker to steal and
      # use against the cluster API.
      automountServiceAccountToken: false
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
        # Defense in depth on top of the Dockerfile's own USER nodejs: even
        # if an image somehow got built or overridden to run as root, K8s
        # itself refuses to start the container as root (runAsNonRoot),
        # pins it to the exact UID the Dockerfile's `adduser -u 1001`
        # created (so a "run as root" image fails loudly instead of quietly
        # succeeding as UID 0), blocks the container from ever gaining more
        # privileges than it started with (allowPrivilegeEscalation), and
        # strips every Linux capability it wasn't explicitly granted — a
        # plain Node HTTP server needs none of them (e.g. no NET_BIND_SERVICE,
        # since we listen on an unprivileged port above 1024).
        securityContext:
          runAsNonRoot: true
          runAsUser: 1001
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
        # Tuned for low-traffic personal projects sharing one small VPS — the
        # CPU *request* is what's reserved against the node's schedulable
        # capacity, so keeping it low lets more projects coexist on one box.
        # Bump these for a specific project only if it actually sees real traffic.
        resources:
          requests:
            memory: "64Mi"
            cpu: "15m"
          limits:
            memory: "256Mi"
            cpu: "150m"
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
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 0
      maxUnavailable: 1
  selector:
    matchLabels:
      app: ${APP_NAME}-frontend
  template:
    metadata:
      labels:
        app: ${APP_NAME}-frontend
    spec:
      # Same reasoning as the backend Deployment above: nginx (or the SSR
      # Node server, if you forked in Step 1) never calls the Kubernetes
      # API, so there's no reason to hand it a token that could talk to it.
      automountServiceAccountToken: false
      containers:
      - name: frontend
        image: ${DOCKERHUB_USERNAME}/${APP_NAME}-frontend:${IMAGE_TAG}
        imagePullPolicy: Always
        ports:
        - containerPort: 5173
        # Same rationale as the backend's securityContext above — pinned to
        # UID 1001 to match the nginx-app user the frontend Dockerfile
        # creates (Step 1). If you forked to the SSR Dockerfile, this is
        # still 1001 since that template creates its non-root user the same
        # way — check your Dockerfile's `adduser -u` value if you changed it.
        securityContext:
          runAsNonRoot: true
          runAsUser: 1001
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
        # Static nginx serving pre-built assets — needs almost nothing at rest.
        # This profile assumes the static-SPA path from Step 1. If you forked
        # to the SSR Dockerfile instead, delete this block and use a profile
        # closer to the backend's above — an SSR Node process does real
        # per-request work (rendering, loader calls to the backend), unlike
        # nginx handing out files from disk, so 32Mi/5m will throttle it hard.
        resources:
          requests:
            memory: "32Mi"
            cpu: "5m"
          limits:
            memory: "64Mi"
            cpu: "50m"
        livenessProbe:
          httpGet:
            path: /
            port: 5173
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /
            port: 5173
          initialDelaySeconds: 5
          periodSeconds: 5
```

Frontend container port (5173, matching Vite's default dev server port for consistency) and probe path (`/`) are left as fixed values on purpose — they're a convention of this guide's nginx setup (Step 1), not something that varies per project. This port is purely internal to the K3s pod network (nginx ↔ Service ↔ Ingress) — it's never exposed directly on the VPS, so it has no effect on your firewall/ufw rules.

**If you forked to the SSR Dockerfile in Step 1:** the fixed `5173`/nginx assumptions above don't apply. Set `containerPort`, the liveness/readiness probe `port`, and `frontend-service.yml`'s `targetPort` (below) to whatever port your framework's `PORT` env var is actually set to (`3000` in the Step 1 SSR Dockerfile example) — and add a `PORT`/`BACKEND_URL` env block to this container spec the same way the backend Deployment above reads its secrets, rather than a `VITE_API_URL` build arg.

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
    targetPort: 5173
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
  # No www.${BACKEND_DOMAIN} rule here on purpose: nobody browses to a "www."
  # prefix on an API subdomain, so a default rule for it is just an unused
  # DNS record and an extra name on the TLS cert for no benefit. If some
  # specific project actually needs it, copy this block, change the host to
  # `www.${BACKEND_DOMAIN}`, and add that host to the `tls.hosts` list below.
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
  tls:
  - hosts:
    - ${FRONTEND_DOMAIN}
    - www.${FRONTEND_DOMAIN}
    - ${BACKEND_DOMAIN}
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

This pipeline was hardened by comparing several near-identical variants of it that had drifted apart across different projects — inconsistent secret names, floating action tags, no test gate, and more. For the full list of what changed and why, see [pipeline-design.md](pipeline-design.md).

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
  # NOTE on `--if-present`: it makes `npm run lint`/`npm test` silently no-op
  # if the project has no `lint`/`test` script — deliberately, so this
  # workflow doesn't force every project to define scripts it may not need
  # yet. But that means an absent script is a SILENT gap, not a failure —
  # this job passing doesn't mean lint/tests actually ran, only that they
  # didn't fail. Check your own package.json has real `lint`/`test` scripts
  # if you want this gate to mean anything for those two.
  #
  # Also: `npm run build --if-present` is NOT a type-check for a Vite
  # frontend. Vite's production build strips TypeScript types via esbuild
  # without ever checking them, so a build can succeed with type errors
  # still in the code. Only `tsc` (or `tsc --noEmit`) actually type-checks.
  # If your frontend has a separate `typecheck` script, add an explicit
  # `npm run typecheck` step below rather than relying on `build` to catch
  # type errors — it won't.
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
          # 22 is this template's default — check EBADENGINE warnings from
          # `npm ci` and set this to whatever the backend's own dependencies
          # actually need (see the Node version note in Step 1). Keep this
          # in sync with the backend Dockerfile's `FROM node:X-alpine`.
          node-version: '22'
          cache: 'npm'
          cache-dependency-path: backend/package-lock.json
      - run: npm ci
      - run: npm run lint --if-present
      - run: npm test --if-present
      - run: npm run build --if-present
      # If backend/package.json has a `typecheck` script, add it explicitly:
      # - run: npm run typecheck

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
          # Same caveat as the backend job above — frontend and backend can
          # legitimately need different Node versions; don't assume they match.
          node-version: '22'
          cache: 'npm'
          cache-dependency-path: frontend/package-lock.json
      - run: npm ci
      - run: npm run lint --if-present
      - run: npm test --if-present
      - run: npm run build --if-present
      # `build` alone does NOT type-check a Vite project (see the note
      # above) — if frontend/package.json has a `typecheck` script
      # (e.g. `tsc --noEmit`), run it explicitly so real type errors fail
      # this job instead of shipping silently:
      # - run: npm run typecheck

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
        # Pinned actions can still break without you touching anything: this
        # action itself used to be pinned here at v0.28.0, which resolved
        # fine for a while, then started failing outright — v0.28.0 pins its
        # own internal dependency (aquasecurity/setup-trivy) to a tag
        # (v0.2.1) that the aquasecurity org later deleted upstream. A pin
        # only freezes *this* action's version string; it doesn't freeze
        # what that version's own dependencies point at, so if a pinned
        # action's own transitive references get cleaned up upstream, your
        # "unchanged" workflow starts failing to resolve anyway. That's not
        # something you did wrong — when a pin suddenly breaks, check what
        # the pinned version's own dependencies reference (its release notes
        # or action.yml) rather than assuming your workflow regressed.
        # v0.36.0 fixes this specific failure mode by pinning setup-trivy by
        # commit SHA instead of a mutable tag. Check
        # https://github.com/aquasecurity/trivy-action/tags for whatever the
        # current latest stable release is before reusing this template.
        uses: aquasecurity/trivy-action@v0.36.0
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
    # Must include workflow_dispatch, not just push — the `on:` trigger list
    # above includes workflow_dispatch specifically so you can manually kick
    # off a real deploy with no new commit (e.g. redeploying to pick up a
    # secrets-only change). If this condition only checked `push`, a manual
    # run would execute the test/scan jobs, then silently skip build-and-push,
    # scan-images, and deploy — the whole run shows green with those three
    # marked "skipped," which reads as success and is easy to miss.
    # workflow_dispatch defaults to running against whatever branch/ref you
    # pick in the Actions tab, so the `github.ref` check still matters here:
    # it's what stops someone accidentally deploying from a non-main ref.
    if: (github.event_name == 'push' || github.event_name == 'workflow_dispatch') && github.ref == 'refs/heads/main'
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
        # See the comment on the trivy-fs-scan job's equivalent step for why
        # this pin matters and what to do if it ever breaks again.
        uses: aquasecurity/trivy-action@v0.36.0
        with:
          image-ref: ${{ env.BACKEND_IMAGE }}:${{ needs.build-and-push.outputs.image-tag }}
          format: 'table'
          severity: 'CRITICAL'
          exit-code: '1'
          ignore-unfixed: true

      - name: Scan frontend image
        uses: aquasecurity/trivy-action@v0.36.0
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
    # Same reasoning as build-and-push's condition above — keep both in sync.
    if: (github.event_name == 'push' || github.event_name == 'workflow_dispatch') && github.ref == 'refs/heads/main'
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

- Create the `production` Environment with required reviewers, **and verify the protection rule actually saved** — see "Set up the `production` Environment with required reviewers" in Step 5. This has a real, silent failure mode: the UI can appear to succeed while the rule never actually took, and `deploy` then runs unblocked with no error or warning either way.
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
2. **BACKEND_PORT** - The port your backend listens on (e.g. `5000`)
3. **HEALTH_CHECK_PATH** - Backend health endpoint used by the liveness/readiness probes (e.g. `/api/health`)
4. **LETSENCRYPT_EMAIL** - Email address used for SSL certificate registration
5. **VITE_API_URL** - Public URL the frontend calls (e.g. `https://api.app.yourdomain.com`)
6. Either:
   - **FRONTEND_DOMAIN** and **BACKEND_DOMAIN** - if you kept ingress.yml Option A (separate subdomains), or
   - **DOMAIN** - if you kept ingress.yml Option B (single domain, `/api` path)

The `deploy` job's render step fails immediately with a clear message if `APP_NAME`, `BACKEND_PORT`, `HEALTH_CHECK_PATH`, or `LETSENCRYPT_EMAIL` are missing — so a forgotten variable shows up as a red pipeline run, not a broken deploy.

### Set up the `production` Environment with required reviewers

The `deploy` job (Step 3) targets `environment: production` specifically so it can be gated behind approval. But referencing `environment: production` in a workflow does **not**, by itself, create any protection rule — it just names an Environment. If that Environment doesn't already exist with a reviewer rule actually saved to it, GitHub auto-creates it on the workflow's first run with **zero** protection rules attached, and `deploy` runs straight through unblocked. There's no error, no warning, nothing distinguishing that run from one that was properly gated — it looks exactly like success, because the deploy *does* succeed, it just silently skipped the safety gate it was supposed to have. The only way to be sure is to verify the rule actually saved, not just perform the setup steps and trust that the UI took them.

**Recommended: set it up via the API**, not just UI clicks — it's unambiguous and scriptable, where the Settings UI gives no strong confirmation that a reviewer was actually added:

```bash
# Find your GitHub user ID — the API needs the numeric ID, not your username
gh api user --jq .id

# Create the environment (or update it) and add yourself as a required
# reviewer in one call
gh api -X PUT repos/<owner>/<repo>/environments/production \
  -f "reviewers[][type]=User" \
  -F "reviewers[][id]=<your-user-id>"
```

(Equivalent UI path: Settings → Environments → New environment → `production` → Deployment protection rules → Required reviewers → add yourself. Both end up in the same place — the API path just leaves no room for a click that silently didn't register.)

**Then verify it actually took — do this before your first push to `main`, not after:**

```bash
gh api repos/<owner>/<repo>/environments/production --jq '.protection_rules'
```

This should return a non-empty array containing a `required_reviewers` rule. If it returns `[]`, the protection rule never actually saved — regardless of what Settings appeared to show you — and the next push to `main` deploys straight to production, unblocked, exactly as if `environment: production` weren't in the workflow at all. Re-run this check any time you're unsure the Environment survived a repo transfer, a re-creation, or just time — it costs one API call and catches a failure mode that otherwise goes unnoticed until someone asks why an unreviewed deploy went out.

One more thing worth knowing rather than being surprised by: `can_admins_bypass` defaults to `true` on a freshly created Environment, meaning repo admins can approve their own deploys (or push through without a second reviewer). That's intentional — it's what stops an admin from being locked out of their own production deploy during an actual emergency — not an oversight in this setup.

Once the Environment exists with its protection rule confirmed, you can optionally move `KUBECONFIG` and your database/app secrets from repository-level secrets into that environment's own secrets. That way a compromised PR workflow (which never touches the `production` environment) can't read your cluster credentials at all — only the reviewed `deploy` job can.

---

## 🌐 Step 6: Configure DNS

Add DNS A records pointing to your VPS IP:

For separate subdomains:
- `app.yourdomain.com` → `YOUR_VPS_IP`
- `www.app.yourdomain.com` → `YOUR_VPS_IP`
- `api.app.yourdomain.com` → `YOUR_VPS_IP`
- (`www.api.app.yourdomain.com` only if you added that optional ingress rule back in Step 2 — the default template doesn't route it)

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

Every manual `kubectl` snippet in this step — and throughout [operations.md](operations.md) — uses `$APP_NAME` instead of a hardcoded name. Set it once per terminal session and every command works as-is, regardless of which project you're operating on:

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
4. On a push to `main`, these run in order: `test-backend` / `test-frontend` → `trivy-fs-scan` → `build-and-push` → `scan-images` → `deploy` (paused for approval **only if** the `production` environment's protection rule actually saved — see Step 5's verification command. If `deploy` runs straight through with no pause on this first push, that's the silent-failure case, not a feature: stop and check `gh api repos/<owner>/<repo>/environments/production --jq '.protection_rules'` before letting another push go out.)

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

## 📝 Customization Checklist

The workflow and Kubernetes manifests never need editing between projects — everything project-specific lives in GitHub Secrets/Variables (Step 5). For a new app:

- [ ] Set `APP_NAME`, `BACKEND_PORT`, `HEALTH_CHECK_PATH`, `LETSENCRYPT_EMAIL`, `VITE_API_URL`, and domain variable(s) (Step 5)
- [ ] Set `DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN`, `KUBECONFIG`, and app-specific secrets (Step 5) — reuse org-level secrets if `DOCKERHUB_USERNAME`/`DOCKERHUB_TOKEN` are the same as your other projects
- [ ] Add any app-specific `--from-literal` lines to the "Create or update Kubernetes secrets" step, and matching `secretKeyRef` entries in `backend-deployment.yml`
- [ ] Adjust CPU/memory resource limits in the manifests if this app's footprint differs from the defaults
- [ ] Update the backend port and health check path in the Dockerfile (Step 1) to match the `BACKEND_PORT`/`HEALTH_CHECK_PATH` variables
- [ ] Update the frontend build output directory in the Dockerfile if it's not `dist` (e.g. Create React App uses `build`)
- [ ] Check `npm ci`'s `EBADENGINE` output for each service and set the Dockerfile base image + CI `setup-node` version accordingly, per service (Step 1)
- [ ] Decide static SPA vs. SSR frontend and use the matching Dockerfile/nginx.conf/K8s manifest pattern (Step 1 and Step 2) — don't mix the two
- [ ] Run the local Docker + `trivy image` recipe at the end of Step 1 before your first push
- [ ] Create the `production` GitHub Environment with required reviewers, **then verify** with `gh api repos/<owner>/<repo>/environments/production --jq '.protection_rules'` that the rule actually saved (Step 5) — this step has a silent failure mode where the UI appears to succeed but the gate never takes
