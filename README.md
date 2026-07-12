# Complete Deployment Guide for MERN Apps on K3s with GitHub Actions

This guide helps you deploy any MERN (or similar full-stack) application to a K3s cluster on a VPS using automated CI/CD with GitHub Actions. It's built as a reusable template: the same Docker files, Kubernetes manifests, and GitHub Actions workflow are copied unchanged into every project — only that project's GitHub Secrets/Variables differ.

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

## 📖 Documentation

This guide is split into three focused docs:

| Doc | Use it for |
| --- | --- |
| [docs/setup-guide.md](docs/setup-guide.md) | Building out a new project end-to-end: Docker files, Kubernetes manifests, the GitHub Actions workflow, VPS/K3s setup, secrets & variables, DNS, database, and first deploy (Steps 1-8) |
| [docs/pipeline-design.md](docs/pipeline-design.md) | Why the pipeline is built the way it is, and the best practices worth carrying into future projects |
| [docs/operations.md](docs/operations.md) | Day-2 reference once a project is live: troubleshooting, making updates, monitoring, common commands |

Start with the setup guide for a new project; come back to the other two once it's running.

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
