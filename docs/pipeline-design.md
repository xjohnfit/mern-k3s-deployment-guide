# Pipeline Design & Best Practices

Why this CI/CD pipeline is built the way it is, and the practices worth carrying into any future project. For the step-by-step of actually using it, see [setup-guide.md](setup-guide.md); for day-2 operations see [operations.md](operations.md).

---

## Design Notes

This pipeline was hardened after comparing several near-identical variants of it that had drifted apart across different projects. If you're copying an older version of this workflow, watch for these problems it fixes:

- **Secret name typos** — some variants read `secrets.KUBE_CONFIG` while the docs said to create `KUBECONFIG`. Pick one name (`KUBECONFIG`) and use it everywhere, including in Step 5 of the [setup guide](setup-guide.md).
- **`paths-ignore: kubernetes/**`** — excluding manifest changes from the trigger means a change to a Deployment/Ingress YAML never gets applied, because the whole workflow (including the `deploy` job) is skipped. Only docs/ignore files should be excluded.
- **Delete-then-recreate secrets** — `kubectl delete secret` followed by `kubectl create` leaves a window where the Secret doesn't exist. Use `--dry-run=client -o yaml | kubectl apply -f -` instead: it updates keys in place with no gap.
- **Floating action tags** (`@master`, `version: latest`) — pin to a specific release so a third-party update can't silently break your pipeline.
- **No test gate** — scanning and deploying without ever running the app's own build/lint/test step means a broken app can still ship.
- **Scanning only source, not the image you actually ship** — a filesystem scan misses vulnerabilities baked into the final container (base image OS packages, etc.).
- **No deploy approval gate** — anyone who can push to `main` gets an unreviewed production deploy using a kubeconfig with full cluster-admin. A GitHub `environment` with required reviewers fixes this.
- **Non-secret values stored as secrets** — `VITE_API_URL` ends up in your public frontend bundle anyway, so it belongs in repository **Variables**, not **Secrets**.

Beyond fixing those, the pipeline is written to be **identical across every project**: `APP_NAME`, ports, domains, and the health-check path are never hardcoded in the workflow or the manifests — they're read from GitHub Actions Variables at deploy time, and the Kubernetes manifests are rendered with `envsubst` right before `kubectl apply`. See [setup-guide.md](setup-guide.md) for how that works in practice.

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
