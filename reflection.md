# Reflection

I took a small Python CLI that saves a QR code as a PNG and put it in a Docker image. The image reads its settings from environment variables, saves output to a mounted folder, and gets pushed to DockerHub by a GitHub Actions workflow.

`COPY . .` grabs everything in the build context. My old QR codes and the `.git` folder were ending up inside the image until I added them to `.dockerignore`.

Layer order matters for build speed. If I copy `requirements.txt` and run `pip install` before copying my source code, Docker reuses the cached install when I edit `main.py`. My rebuilds went from around 30 seconds to about 1 second.

Anything a container writes disappears when the container stops. My first few runs printed a success message and I still had no file on my laptop. Mounting `./qr_codes:/app/qr_codes` fixed it. Keeping config in environment variables means one image can make a red QR code or a blue one just by changing `-e FILL_COLOR=...`.

Permissions took me the longest. The Dockerfile switches to `myuser`, so I had to create `logs` and `qr_codes` and `chown` them in the same `RUN` step before that switch. Then the volume mount overrode those permissions anyway, and Docker Desktop gave me an "operation not permitted" error until I let it share the folder.

For CI, I used `DOCKERHUB_USERNAME` and a `DOCKERHUB_TOKEN` stored as repository secrets. A token is safer than my password since I can revoke it. I also added a step that runs the container before pushing. A successful build only means the image assembled, so I wanted to see it actually write a file.

If I kept working on this, I would pin the base image by digest instead of the `python:3.12-slim-bullseye` tag, so a rebuild months from now produces the same image.
