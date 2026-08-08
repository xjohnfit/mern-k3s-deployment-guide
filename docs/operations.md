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

### `scan-images` CI job fails on a CVE not in your dependencies

If the pipeline dies in `scan-images` with a CRITICAL finding that doesn't appear anywhere in `package-lock.json` or `npm ls`, check what package the finding is actually in — `node:*-alpine` base images ship a global npm install with its own bundled internal `tar` package, and that bundled tar has had CRITICAL CVEs of its own (gzip-bomb DoS, etc.) with nothing to do with your app's code. It's confusing the first time because it genuinely isn't one of your dependencies. Fix is in the Dockerfile, not your app: remove npm/npx/corepack from the final image (see the `rm -rf` step in the backend Dockerfile and the SSR frontend Dockerfile, Step 1 of [setup-guide.md](setup-guide.md)) since the running container only needs `node`, never `npm`, at runtime. Catch this locally next time by running `trivy image <local-image>` after `docker build` — see "Test your images locally before pushing" at the end of Step 1.

### `trivy-action` fails to resolve / can't find a pinned tag

A pinned third-party GitHub Action can break even without you touching your workflow, if *its own* pinned dependency gets cleaned up upstream. This happened with `aquasecurity/trivy-action@v0.28.0`, which pinned `aquasecurity/setup-trivy@v0.2.1` — a tag the aquasecurity org later deleted, so the action stopped resolving entirely. If a pinned action suddenly fails to resolve, don't assume you broke something: check the pinned version's own `action.yml`/release notes for what tag or dependency it references, and bump to a newer release if that dependency has moved on. Current setup-guide.md pins `trivy-action@v0.36.0`, which sidesteps this specific failure mode by pinning `setup-trivy` by commit SHA instead of a mutable tag — but check [the tags page](https://github.com/aquasecurity/trivy-action/tags) for whatever's current before assuming that pin is still the latest stable one.

### A deploy went out to production without pausing for approval

`environment: production` in the `deploy` job (Step 3) only pauses for approval if that Environment already has a `required_reviewers` protection rule actually saved to it. Referencing the Environment name in the workflow does not create the rule by itself — if it was never saved (a UI click that silently didn't register, or the Environment simply not created yet), GitHub auto-creates it with zero protection rules on first use and `deploy` runs straight through, with no error or warning to flag that the gate didn't apply. This looks exactly like a successful, ungated deploy, because that's exactly what it is.

Check whether the rule is actually in place:

```bash
gh api repos/<owner>/<repo>/environments/production --jq '.protection_rules'
```

An empty array (`[]`) means there's no reviewer gate at all — fix it with the `gh api -X PUT .../environments/production` command in Step 5 of [setup-guide.md](setup-guide.md), then re-run this check to confirm it saved before trusting it again. This is also worth checking any time a repo gets transferred or an Environment gets recreated, not just once at initial setup.

### Manually running the workflow ("Run workflow" button) does nothing

If you trigger the workflow via `workflow_dispatch` (Actions tab → this workflow → Run workflow) — e.g. to redeploy after changing a secret with no new commit — and the run finishes green but nothing actually deployed, check whether `build-and-push`, `scan-images`, and `deploy` show as **Skipped** rather than having run. Their `if` conditions have to explicitly include `github.event_name == 'workflow_dispatch'` alongside `push`; if a copy of this workflow only checks for `push`, a manual run passes the test/scan jobs and then silently skips the three jobs that actually do anything, with no failure anywhere to flag it. Compare your workflow's `if:` lines on those two jobs against the current version in [setup-guide.md](setup-guide.md) Step 3.

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
