# RedLime DevOps Technical Assessment

## Project Overview

This project deploys two independent static websites — a company landing page (**Site 1**, for a company named NexoStudio) and a portfolio/about page (**Site 2**) — on a single Linux host using Docker, with Nginx acting as a reverse proxy in front of both. Deployment to the host is fully automated via GitHub Actions: every push to `main` triggers a build and redeploy with no manual file copying or manual server login required.

- **Site 1** — `/site1` — company landing page
- **Site 2** — `/site2` — portfolio / about page
- Both are served through a single Nginx entry point using path-based routing, so no DNS or `/etc/hosts` configuration is required to access either site.

---

## Folder Structure

```
redlime_assessment/
├── site1/
│   ├── Dockerfile
│   ├── .dockerignore
│   ├── index.html
│   ├── index.css
│   └── index.js
├── site2/
│   ├── Dockerfile
│   ├── .dockerignore
│   ├── index.html
│   ├── about.css
│   └── about.js
├── nginx/
│   └── nginx.conf
├── .github/
│   └── workflows/
│       └── deploy.yml
├── docker-compose.yml
├── .env.example
├── .gitignore
└── README.md
```

---

## Server Requirements

To run or redeploy this project, the host machine needs:

- A Linux environment (built and tested on Arch Linux; any modern Linux distribution with Docker support will work)
- [Docker Engine](https://docs.docker.com/engine/install/) (v24+)
- [Docker Compose plugin](https://docs.docker.com/compose/install/) (v2 — invoked as `docker compose`, space-separated, **not** the older standalone `docker-compose` hyphenated tool)
- Git
- Port `8080` (or whatever `HTTP_PORT` is set to in `.env`) free on the host for Nginx to bind to

---

## Docker Setup

Each site runs in its **own container**, built from its own `Dockerfile`. Both use `nginx:alpine` as a minimal static file server:

```dockerfile
FROM nginx:alpine
COPY . /usr/share/nginx/html
EXPOSE 80
```

`nginx:alpine` was chosen over a heavier base image or a Node/Express server because these sites are pure static HTML/CSS/JS with no build step or server-side logic — it keeps the image small and keeps the tooling consistent with the reverse proxy layer.

Each site folder also has a `.dockerignore` to keep the build context (and resulting image) clean:

```
.git
*.md
Dockerfile
```

This excludes the site's own `Dockerfile` and any Markdown docs from being copied into the served content — they're needed by Docker to *build* the image, not needed inside the running container.

---

## Docker Compose Usage

`docker-compose.yml` at the project root defines three services: `site1`, `site2`, and `nginx`.

```yaml
services:
  site1:
    build: ./site1
    expose:
      - "80"

  site2:
    build: ./site2
    expose:
      - "80"

  nginx:
    image: nginx:alpine
    ports:
      - "${HTTP_PORT}:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      - site1
      - site2
```

Key decisions:

- `site1` and `site2` use `expose` (internal-only, reachable by other containers on the Compose network) rather than `ports` (host-mapped). Only `nginx` needs to be reachable from outside the Docker network, since it's the single entry point for both sites.
- `HTTP_PORT` is read from a `.env` file (see [Environment Variables](#environment-variables) below) rather than hardcoded, so the exposed port can be changed without editing `docker-compose.yml`.

**Common commands:**

```bash
# Build and start all services
docker compose up -d --build

# View running containers
docker ps

# Stop and remove all containers
docker compose down

# View logs for a specific service
docker compose logs nginx
```

---

## Nginx Configuration

`nginx/nginx.conf` is mounted into the Nginx container and handles path-based routing to each site:

```nginx
server {
    listen 80;

    location /site1/ {
        proxy_pass http://site1/;
        proxy_set_header Host $host;
    }

    location /site2/ {
        proxy_pass http://site2/;
        proxy_set_header Host $host;
    }
}
```

The trailing slash on both the `location` block and the `proxy_pass` target is intentional — it tells Nginx to **strip the `/site1` or `/site2` prefix** before forwarding the request to the corresponding container. This means each site container sees a normal request for `/`, `/index.css`, `/index.js`, etc., exactly as if it were being accessed at its own root — so the sites' existing relative asset references work unmodified.

**Design decision:** path-based routing was chosen over subdomain-based routing (`site1.local`, `site2.local`) because it requires no DNS setup and works immediately against a bare IP or `localhost`. It also keeps the project simple, given the minimal scope of two static sites. Because each site lives in a separate container with no shared filesystem, internal navigation links between them (Home ↔ About) use **absolute paths** (`/site1/index.html`, `/site2/index.html`) rather than relative filesystem paths — the shared Nginx layer, not the filesystem, is what unifies them into one browsable experience.

---

## GitHub Actions Workflow

`.github/workflows/deploy.yml` automates deployment on every push to `main`:

```yaml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: self-hosted
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Write environment file
        run: |
          cat > .env << EOF
          NODE_ENV=${{ secrets.NODE_ENV }}
          HTTP_PORT=${{ secrets.HTTP_PORT }}
          EOF

      - name: Deploy with Docker Compose
        run: |
          docker compose down
          docker compose up -d --build
```

**How it works:**

1. A push to `main` is detected by GitHub.
2. The self-hosted runner (registered on the deployment host) picks up the job.
3. The workflow checks out the latest code into a fresh workspace.
4. It writes a `.env` file from **GitHub Actions Secrets** — real values are never stored in the repository, only referenced by name from an encrypted secrets store.
5. It tears down old containers and rebuilds/restarts everything with `docker compose up -d --build`, so the running deployment always matches the latest pushed commit.

No manual SSH login or file copying is involved at any point after the initial one-time runner registration.

**Security note:** this repository is public, and GitHub explicitly warns that self-hosted runners on public repos can be abused if a workflow triggers on `pull_request` or `pull_request_target` — a stranger could fork the repo and get arbitrary code executed on the host via a malicious PR. This workflow deliberately triggers only on `push: branches: [main]`, which requires write access to this repository, closing off that specific attack path. No untrusted collaborators have been added to the repo.

Below: two consecutive pushes, each automatically triggering and completing a full redeploy with no manual intervention.

*(insert workflow-runs screenshot here)*

---

## Environment Variables

Real secret/config values are never committed to git. `.env.example` documents what's expected; a real `.env` (gitignored) supplies actual values locally, and GitHub Actions Secrets supply them during automated deployment.

`.env.example`:
```
NODE_ENV=production
HTTP_PORT=8080
```

---

## Deployment Steps

**One-time setup on the host machine:**

- Install Docker and the Docker Compose plugin
- Clone the repository
- Register a GitHub Actions self-hosted runner:
  - In the repo: **Settings → Actions → Runners → New self-hosted runner**
  - Follow the generated commands to download, configure, and install as a service (`./svc.sh install && ./svc.sh start`)
- Add required secrets in **Settings → Secrets and variables → Actions**:
  - `NODE_ENV`
  - `HTTP_PORT`

**Ongoing deployment (fully automated):**

- `git add . && git commit -m "..." && git push origin main`
- The push automatically triggers `.github/workflows/deploy.yml` on the self-hosted runner
- The workflow writes `.env` from GitHub Secrets, tears down old containers, and rebuilds/redeploys with `docker compose up -d --build`
- Progress and pass/fail status are visible live in the repository's **Actions** tab

---

## Running the Project Locally

- Clone the repository onto your machine.
- Open the terminal and head to the folder: `cd redlime_assessment`
- Copy the `.env.example` file to a newly created `.env` file
  - For this project, the same values are kept for `.env.example` and `.env` for reproducing the setup.
- Run:
  ```bash
  docker compose up -d --build
  ```
- Confirm all three containers are running:
  ```bash
  docker ps
  ```
  You should see `site1`, `site2`, and `nginx` containers listed as `Up`.
- Visit the websites to check:
  - `http://localhost:8080/site1/index.html`
  - `http://localhost:8080/site2/index.html`
- To stop everything:
  ```bash
  docker compose down
  ```

---

## Assumptions & Design Decisions

- **Path-based routing** was chosen over subdomain-based routing since it requires no DNS setup and keeps the project simple given its minimal scope. Both sites run in separate containers with no shared filesystem — the shared Nginx layer is what resolves navigation between them.
- **Self-hosted GitHub Actions runner instead of a cloud VPS:** major cloud providers (Google Cloud, AWS, Oracle Cloud) all require a credit/debit card for free-tier signup verification, which wasn't available at the time of this assessment. A self-hosted runner on the developer's own Arch Linux machine — already running Docker, Docker Compose, and Nginx — was chosen as the next-best option, since it still satisfies every core requirement: containerized sites, Compose orchestration, an Nginx reverse proxy, and genuinely automated push-to-deploy CI/CD (see the Actions tab history above). `ubuntu-latest` was deliberately not used as the runner target, since GitHub's disposable cloud VMs would still need to reach out to a separate real, persistent server to deploy anything meaningful — which was exactly the constraint this approach was chosen to avoid. Because the runner and the deployment target are the same local machine, no SSH step is required at all.
- **SSL / Let's Encrypt was not implemented**, as a direct consequence of the above — Let's Encrypt requires a real, publicly resolvable domain pointing at a reachable server to issue a certificate, which isn't available for a `localhost`-only self-hosted deployment.
- **`nginx:alpine`** was used for both site containers rather than a Node/Express server, since the sites are static HTML/CSS/JS with no server-side logic, keeping images small and the tooling consistent with the reverse proxy layer.