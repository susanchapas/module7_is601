# Module 7 — Dockerized QR Code Generator

A Python CLI that encodes a URL into a QR code PNG. It runs inside a Docker
container as a non-root user, is configured through environment variables, and
writes its output to a mounted volume so the generated images survive on the
host.

## Links

| Resource | Link |
| --- | --- |
| GitHub repository | https://github.com/susanchapas/module7_is601 |
| DockerHub image | https://hub.docker.com/r/susanchapas/module7_is601 |

## Screenshots

| Evidence | File |
| --- | --- |
| Container logs (successful run) | [screenshots/container-logs.png](screenshots/container-logs.png) |
| GitHub Actions workflow (successful run) | [screenshots/github-actions.png](screenshots/github-actions.png) |

## Project files

| File | Purpose |
| --- | --- |
| [main.py](main.py) | QR code generator CLI |
| [requirements.txt](requirements.txt) | Python dependencies |
| [Dockerfile](Dockerfile) | Image definition (Python 3.12 slim, non-root user) |
| [docker-compose.yml](docker-compose.yml) | Env vars + volume mount for local runs |
| [.github/workflows/docker-image.yml](.github/workflows/docker-image.yml) | CI: build, smoke test, push to DockerHub |
| [reflection.md](reflection.md) | Reflection document |

## Configuration

Environment variables read by the application:

| Variable | Default | Purpose |
| --- | --- | --- |
| `QR_CODE_DIR` | `qr_codes` | Directory the PNG is written to |
| `FILL_COLOR` | `red` | QR code foreground color |
| `BACK_COLOR` | `white` | QR code background color |

Command-line argument:

| Argument | Default | Purpose |
| --- | --- | --- |
| `--url` | `https://github.com/kaw393939` | URL encoded in the QR code |

## Build and run

Build the image:

```bash
docker build -t module7-qr .
```

Run it, mounting the host `qr_codes` directory so the PNG lands on your machine:

```bash
docker run --rm \
  -e FILL_COLOR=blue \
  -e BACK_COLOR=white \
  -e QR_CODE_DIR=/app/qr_codes \
  -v "$PWD/qr_codes:/app/qr_codes" \
  module7-qr --url https://www.njit.edu
```

Expected log line:

```
2026-08-10 02:56:10,857 - INFO - QR code successfully saved to /app/qr_codes/QRCode_20260810025610.png
```

Or use Compose, which supplies the same env vars and volume mount:

```bash
docker compose up --build
```

Pull the published image instead of building:

```bash
docker run --rm -v "$PWD/qr_codes:/app/qr_codes" \
  susanchapas/module7_is601:latest --url https://www.njit.edu
```

## Running without Docker

```bash
python3 -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate.bat
pip install -r requirements.txt
python main.py --url https://www.njit.edu
```

## Continuous integration

Every push to `main` triggers [.github/workflows/docker-image.yml](.github/workflows/docker-image.yml),
which builds the image, runs the container once as a smoke test, then logs in to
DockerHub and pushes `latest` plus a commit-SHA tag.

The workflow needs two repository secrets
(**Settings → Secrets and variables → Actions**):

| Secret | Value |
| --- | --- |
| `DOCKERHUB_USERNAME` | Your DockerHub username |
| `DOCKERHUB_TOKEN` | A DockerHub access token (**Account Settings → Personal access tokens**) |
