# Reflection — Dockerizing the QR Code Generator

## What I set out to do

Take a small Python CLI that writes a QR code PNG and make it run the same way
on any machine: build it into a Docker image, configure it through environment
variables, persist its output through a volume mount, and publish the image to
DockerHub from a GitHub Actions workflow.

## What I learned

**Images are built from a context, not from my folder.** The `COPY . .`
instruction copies whatever the build context contains, so stale QR codes and
the `.git` directory were being baked into the image. Adding `qr_codes/`,
`logs/`, and `.github` to `.dockerignore` kept the image limited to the source
code and dependencies, which also made rebuilds faster.

**Layer order controls rebuild cost.** Copying `requirements.txt` and running
`pip install` *before* copying the application source means editing `main.py`
does not invalidate the dependency layer. Docker reuses the cached install, and
rebuilds drop from tens of seconds to under a second.

**Containers are ephemeral; volumes are not.** My first runs looked successful —
the log said the PNG was saved — but nothing appeared on my machine, because the
file was written inside the container's filesystem and disappeared with the
container. Mounting `./qr_codes:/app/qr_codes` was what actually made the output
survive.

**Configuration belongs outside the image.** `main.py` reads `QR_CODE_DIR`,
`FILL_COLOR`, and `BACK_COLOR` with `os.getenv` and sensible defaults, so the
same image produces a red or a blue QR code depending on `-e FILL_COLOR=...`.
The `ENTRYPOINT` / `CMD` split does the same thing for the URL: `ENTRYPOINT`
fixes the program, `CMD` supplies a default `--url` that any argument after the
image name overrides.

## Challenges I hit

**Non-root user versus directory permissions.** The Dockerfile switches to
`myuser` for security, but a process that cannot write to its own output
directory just fails. The fix was ordering: create `logs` and `qr_codes` and
`chown` them to `myuser` in the same `RUN` layer *before* the `USER` switch, and
copy the source with `COPY --chown=myuser:myuser`.

**Volume mounts are the fragile part.** A mounted host directory replaces
whatever the image had at that path, including its ownership, so the permissions
set at build time do not necessarily apply at run time. Mount paths also have to
be absolute — `-v "$PWD/qr_codes:/app/qr_codes"` — and Docker Desktop has to have
file-sharing access to the parent folder, which produced a confusing
"operation not permitted" error until I granted it.

**Secrets in CI.** The workflow cannot hold my DockerHub password, so it
authenticates with `DOCKERHUB_USERNAME` and a `DOCKERHUB_TOKEN` access token
stored as repository secrets. Referencing them as `${{ secrets.* }}` keeps the
credentials out of the repository, and using a revocable token instead of my
account password limits the damage if it ever leaks.

**Verifying rather than assuming.** I added a step to the workflow that runs the
container once before pushing. A build that succeeds only proves the image
assembles; running it proves the application actually starts and writes a file.
That step is what turns a green check mark into real evidence.

## What I would do next

Publish multi-architecture images (`linux/amd64` and `linux/arm64`) so the image
runs on both Apple Silicon and CI runners, add a healthcheck or exit-code
assertion to the smoke test, and pin the base image by digest so builds are
fully reproducible.
