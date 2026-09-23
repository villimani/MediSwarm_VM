# MediSwarm cross-machine setup

This is a walkthrough for deploying the MediSwarm CIFAR10 federated learning example across two separate physical machines: Machine A is the server and site-1 and Machine B is just site-2

---

## 0. What needs to be on each computer before you start


### Accounts (create once — not tied to either machine) 
- A [Tailscale](https://tailscale.com/) account (free tier is enough)  you'll sign into it on both machines 
-  A [Docker Hub](https://hub.docker.com/) account (free tier is enough) — (Machine A ,the one that builds and pushes the image)

### Software to install 
**On both machines:** 
- [Docker Desktop](https://www.docker.com/products/docker-desktop/), installed and running
- [Git](https://git-scm.com/downloads) 
- The Tailscale app, signed into the account above 

**On Windows, additionally:** 
- Long file path support, enabled once in an **administrator** PowerShell: 

```powershell
git config --global core.longpaths true

New-ItemProperty `
    -Path "HKLM:\SYSTEM\CurrentControlSet\Control\FileSystem" `
    -Name "LongPathsEnabled" `
    -Value 1 `
    -PropertyType DWORD `
    -Force
```

**macOS:** 
- Nothing extra 

**A note on GPUs**: The Docker image is built on `nvcr.io/nvidia/pytorch`, so it only supports NVIDIA CUDA GPUs  an AMD GPU or Apple Silicon Mac cannot use it regardless of settings, and you'll see `WARNING: The NVIDIA Driver was not detected` in the container logs on those machines, which is expected. On a machine with an actual NVIDIA GPU, no image rebuild is needed — just install the NVIDIA Container Toolkit on that machine and add `--gpus=all` to its `docker run` command (see Section 9). 

No local NVFlare install, no local Python environment setup, and no manual dataset download are needed on either machine everything runs inside Docker containers.

---
## 1. Set up Tailscale (both machines)

Tailscale creates a private VPN mesh between your machines so they can reach each other by a stable hostname, regardless of home network setup, firewalls, or NAT.

1. Install Tailscale on **both** machines from [tailscale.com/download](https://tailscale.com/download) and sign in with the same account on both.
2. Rename both devices to something memorable in the admin console (e.g. `mediswarm-server`, `mediswarm-site2`), and note their full hostnames.
3. Decide which machine will be Machine A (server + site-1) and which will be Machine B (site-2), you'll need Machine A's full hostname again in Section 2 as $SERVER_FQDN.
4. Verify it's running and can reach the other machine:

#### Windows (PowerShell)
```powershell
$PEER_DEVICE_NAME = "<other-machine-tailscale-name>"

& "C:\Program Files\Tailscale\tailscale.exe" status
& "C:\Program Files\Tailscale\tailscale.exe" ping $PEER_DEVICE_NAME
```

#### macOS (bash)
```bash
PEER_DEVICE_NAME="<other-machine-tailscale-name>"

tailscale status
tailscale ping "$PEER_DEVICE_NAME"
```

> On whichever machine ends up as Machine A, keep that terminal open through Sections 2–10 or re-set `$FL_ROOT`, `$SERVER_FQDN` (from Section 2) and `$IMAGE_TAG` (from Section 4)  in a new window before continuing. `$SERVER_FQDN` in particular is needed again in Section 9.


---
## 2. Clone the repository (Machine A)

#### Windows (PowerShell)

```powershell
$FL_ROOT      = "C:\workspace\mediswarm_cifar10"
$REPO_URL     = "<your-fork-url>"
$SERVER_FQDN  = "<this-machine's-tailscale-name-e.g.-mediswarm-server.tailXXXXXX.ts.net>"

git clone --recurse-submodules $REPO_URL $FL_ROOT
cd $FL_ROOT
```
If you hit a `Filename too long` error during the clone, re-run:

```powershell
git submodule update --init --recursive
```

#### macOS (bash)
```bash
FL_ROOT="$HOME/mediswarm_cifar10"
REPO_URL="<your-fork-url>"
SERVER_FQDN="<this-machine's-tailscale-name-e.g.-mediswarm-server.tailXXXXXX.ts.net>"

git clone --recurse-submodules "$REPO_URL" "$FL_ROOT"
cd "$FL_ROOT"
```
`--recurse-submodules` is required up front — this repo pulls in a customized NVFlare fork as a git submodule, and a further nested submodule inside that. Without this flag, the Docker build will fail later.


---


## 3. Fix the Dockerfile (Machine A)

Open `docker_config/Dockerfile_cifar10` and make two changes.

### 3a. Make the base image architecture-aware

Find:
```dockerfile
ARG PYTORCH_IMAGE=nvcr.io/nvidia/pytorch:23.11-py3-igpu
FROM ${PYTORCH_IMAGE}
```

Replace with:
```dockerfile
ARG PYTORCH_IMAGE_AMD64=nvcr.io/nvidia/pytorch:23.11-py3
ARG PYTORCH_IMAGE_ARM64=nvcr.io/nvidia/pytorch:23.11-py3-igpu
ARG TARGETARCH

FROM ${PYTORCH_IMAGE_AMD64} AS base-amd64
FROM ${PYTORCH_IMAGE_ARM64} AS base-arm64

FROM base-${TARGETARCH}
```
This lets one Dockerfile build correctly for both amd64 (Windows, and Intel Macs) and arm64 (Apple Silicon Macs). Docker picks the right base automatically via TARGETARCH during the multi-platform build in Section 4.

### 3b. Pin a working aiohttp version

Add this as a new block, right before the final `LABEL name="nvflare-pt-dev:cifar10"` line:

```dockerfile
RUN python3 -m pip install aiohttp==3.14.3
```

The base image ships an old aiohttp that NVFlare's networking code isn't compatible with. Skip this and the server will crash the moment a client connects, with `AttributeError: 'WebSocketResponse' object has no attribute 'get_extra_info'`.

---

## 4. Build and publish the image to Docker Hub (Machine A)

Run from your repo root (`$FL_ROOT`). Define your variables, then build and push a single image covering both linux/amd64 (Windows, and Intel Macs) and linux/arm64 (Apple Silicon Macs).

#### Windows (PowerShell)

```powershell
$DOCKER_HUB_USER = "<your-dockerhub-username>"
$IMAGE_TAG = "${DOCKER_HUB_USER}/nvflare-pt-dev:cifar10"

# Authenticate to Docker Hub (use an Access Token when prompted, not your password)
docker login -u $DOCKER_HUB_USER

# Set up a multi-arch builder
docker buildx create --name multiarch --use
docker buildx inspect --bootstrap

# Build and push the multi-arch image
docker buildx build `
  --platform linux/amd64,linux/arm64 `
  -f docker_config/Dockerfile_cifar10 `
  -t $IMAGE_TAG `
  --push .
```
#### macOS (bash)

```bash
DOCKER_HUB_USER="<your-dockerhub-username>"
IMAGE_TAG="${DOCKER_HUB_USER}/nvflare-pt-dev:cifar10"

# Authenticate to Docker Hub (use an Access Token when prompted, not your password)
docker login -u "$DOCKER_HUB_USER"

# Set up a multi-arch builder
docker buildx create --name multiarch --use
docker buildx inspect --bootstrap

# Build and push the multi-arch image
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -f docker_config/Dockerfile_cifar10 \
  -t "$IMAGE_TAG" \
  --push .
```
---

## 5. Prepare the training data (Machine A)

This does two things in one, downloads the CIFAR-10 dataset  then runs one full local *simulated* training job. That simulation isn't just a test it's the way this repo generates the per-client data split. Since it simulates 2 clients (`-n 2`), the split comes out already named `site-1`/`site-2` so this step is a prerequisite, not a run you can skip.

#### Windows (PowerShell)

```powershell
cd $FL_ROOT

# Create directories
New-Item -ItemType Directory -Force -Path "cifar10_data", "cifar10_splits", "nvflare_workspace"

# Run dataset download and initial split generation
docker run --rm `
  -v "${PWD}:/workspace" `
  -v "${PWD}\cifar10_data:/tmp/cifar10" `
  -v "${PWD}\cifar10_splits:/tmp/cifar10_splits" `
  -v "${PWD}\nvflare_workspace:/tmp/nvflare" `
  -w /workspace `
  $IMAGE_TAG `
  /bin/bash -c "sed -i 's/\r$//' ./application/jobs/cifar10/prepare_data.sh && ./application/jobs/cifar10/prepare_data.sh && nvflare simulator ./application/jobs/cifar10 -w /tmp/nvflare/cifar10 -n 2 -t 2"
```

#### macOS (bash)

```bash
cd "$FL_ROOT"

# Create directories
mkdir -p cifar10_data cifar10_splits nvflare_workspace

# Run dataset download and initial split generation
docker run --rm \
  -v "${PWD}:/workspace" \
  -v "${PWD}/cifar10_data:/tmp/cifar10" \
  -v "${PWD}/cifar10_splits:/tmp/cifar10_splits" \
  -v "${PWD}/nvflare_workspace:/tmp/nvflare" \
  -w /workspace \
  "$IMAGE_TAG" \
  /bin/bash -c "sed -i 's/\r$//' ./application/jobs/cifar10/prepare_data.sh && ./application/jobs/cifar10/prepare_data.sh && nvflare simulator ./application/jobs/cifar10 -w /tmp/nvflare/cifar10 -n 2 -t 2"
```


---
## 6. Provision the deployment (Machine A)

### 6a. Create deployment configuration file
Create `application/provision/project_mediswarm_2site.yml`(or something similar):

```yaml
api_version: 3
name: mediswarm_cifar10_test
description: MediSwarm CIFAR10 2-client deploy across Tailscale — server + site-1 on Machine A, site-2 on Machine B

participants:
  - name: <server-fqdn-e.g.-mediswarm-server.tailXXXXXX.ts.net>
    type: server
    org: mediswarm
    fed_learn_port: 8002
    admin_port: 8003
  - name: site-1
    type: client
    org: mediswarm
  - name: site-2
    type: client
    org: mediswarm
  - name: admin@mediswarm.local
    type: admin
    org: mediswarm
    role: project_admin

builders:
  - path: nvflare.lighter.impl.workspace.WorkspaceBuilder
    args:
      template_file: master_template.yml
  - path: nvflare.lighter.impl.template.TemplateBuilder
  - path: nvflare.lighter.impl.static_file.StaticFileBuilder
    args:
      config_folder: config
      docker_image: <your-dockerhub-username>/nvflare-pt-dev:cifar10
      overseer_agent:
        path: nvflare.ha.dummy_overseer_agent.DummyOverseerAgent
        overseer_exists: false
        args:
          sp_end_point: <server-fqdn-e.g.-mediswarm-server.tailXXXXXX.ts.net>:8002:8003
  - path: nvflare.lighter.impl.cert.CertBuilder
  - path: nvflare.lighter.impl.signature.SignatureBuilder
```
Adapted from the repo's own `application/provision/project_deploy_test_2site.yml`, swapping local simulation for two real network locations.

### 6b. Run provisioning
#### Windows (PowerShell)
```powershell
cd $FL_ROOT

docker run --rm `
  -v "${PWD}\application\provision:/prov" `
  -w /prov `
  $IMAGE_TAG `
  nvflare provision -p project_mediswarm_2site.yml -w /prov/provision_output
```


#### macOS (bash)
```bash
cd "$FL_ROOT"

docker run --rm \
  -v "${PWD}/application/provision:/prov" \
  -w /prov \
  "$IMAGE_TAG" \
  nvflare provision -p project_mediswarm_2site.yml -w /prov/provision_output
```

This generates four folders under `application/provision/provision_output/mediswarm_cifar10_test/prod_00/`: one named after `$SERVER_FQDN`, plus `site-1`, `site-2`, and `admin@mediswarm.local` — each holding that participant's certs and connection config. You'll need these in later steps.

---
## 7. Send the site-2 kit and data to Machine B (Machine A)

The "kit" is the `site-2` folder Section 6 generated, site-2's certificate and connection config, not just data. Without it Machine B has no identity to join the federation with.

#### Windows (PowerShell)
```powershell
cd $FL_ROOT

# Package kit and datasets
tar -czf site2_kit.tar.gz -C application\provision\provision_output\mediswarm_cifar10_test\prod_00 site-2
tar -czf cifar10_data_splits.tar.gz cifar10_data cifar10_splits

# Send via Taildrop
& "C:\Program Files\Tailscale\tailscale.exe" file cp site2_kit.tar.gz "${PEER_DEVICE_NAME}:"
& "C:\Program Files\Tailscale\tailscale.exe" file cp cifar10_data_splits.tar.gz "${PEER_DEVICE_NAME}:"
```
#### macOS (bash)
```bash
cd "$FL_ROOT"

# Package kit and datasets
tar -czf site2_kit.tar.gz -C application/provision/provision_output/mediswarm_cifar10_test/prod_00 site-2
tar -czf cifar10_data_splits.tar.gz cifar10_data cifar10_splits

# Send via Taildrop
tailscale file cp site2_kit.tar.gz "${PEER_DEVICE_NAME}:"
tailscale file cp cifar10_data_splits.tar.gz "${PEER_DEVICE_NAME}:"
```
---
## 8. Set up Machine B (site-2)

No build needed here — just pull the image Section 4 already published.

#### Windows (PowerShell)
```powershell
$FL_ROOT = "C:\workspace\mediswarm_cifar10"
$DOCKER_HUB_USER = "<your-dockerhub-username>"
$IMAGE_TAG = "${DOCKER_HUB_USER}/nvflare-pt-dev:cifar10"

# Set up project directories
New-Item -ItemType Directory -Force -Path "$FL_ROOT\startup_kits"
cd $FL_ROOT

# Extract archives from wherever Tailscale saved the incoming files
# (check the filenames match what Step 7 sent, and grab the newest copies if you've re-sent them)
tar -xzf "$env:USERPROFILE\Downloads\site2_kit.tar.gz" -C startup_kits
tar -xzf "$env:USERPROFILE\Downloads\cifar10_data_splits.tar.gz" -C .

# Pull the image — resolves the right architecture for this machine automatically
docker pull $IMAGE_TAG
```

#### macOS (bash)

```bash
export FL_ROOT="$HOME/mediswarm_cifar10"
export DOCKER_HUB_USER="<your-dockerhub-username>"
export IMAGE_TAG="${DOCKER_HUB_USER}/nvflare-pt-dev:cifar10"

# Set up project directories
mkdir -p "${FL_ROOT}/startup_kits"
cd "${FL_ROOT}"

# Extract archives from the Taildrop download folder
# (check the filenames match what Step 7 sent, and grab the newest copies if you've re-sent them)
tar -xzf ~/Downloads/site2_kit.tar.gz -C startup_kits
tar -xzf ~/Downloads/cifar10_data_splits.tar.gz -C .

# Pull the image — resolves the right architecture for this machine automatically
docker pull $IMAGE_TAG
```


---

## 9. Start everything

### Machine A — server

Still in the same window as Sections 2, 4, and 6
If not, re-set `$FL_ROOT`, `$IMAGE_TAG`, and `$SERVER_FQDN` first.

#### Windows (PowerShell)
```powershell
cd $FL_ROOT

docker rm -f flserver 2>$null
docker run -d --name=flserver `
  -v "${PWD}\application\provision\provision_output\mediswarm_cifar10_test\prod_00\${SERVER_FQDN}:/startupkit/" `
  -w /startupkit/startup/ `
  --ipc=host `
  -p 8002:8002 `
  -p 8003:8003 `
  $IMAGE_TAG `
  /bin/bash -c "export PYTHONPATH=/workspace/controller && python -u -m nvflare.private.fed.app.server.server_train -m /startupkit -s fed_server.json --set secure_train=true config_folder=config org=mediswarm"
```
#### macOS (bash)
```bash
cd "$FL_ROOT"

docker rm -f flserver 2>/dev/null
docker run -d --name=flserver \
  -v "${PWD}/application/provision/provision_output/mediswarm_cifar10_test/prod_00/${SERVER_FQDN}:/startupkit/" \
  -w /startupkit/startup/ \
  --ipc=host \
  -p 8002:8002 \
  -p 8003:8003 \
  "$IMAGE_TAG" \
  /bin/bash -c "export PYTHONPATH=/workspace/controller && python -u -m nvflare.private.fed.app.server.server_train -m /startupkit -s fed_server.json --set secure_train=true config_folder=config org=mediswarm"
```

### Machine A — site-1

Same window as above.

#### Windows (PowerShell)
```powershell
docker rm -f site-1 2>$null
docker run -d --name=site-1 `
  -v "${PWD}\application\provision\provision_output\mediswarm_cifar10_test\prod_00\site-1:/startupkit/" `
  -v "${PWD}\cifar10_data:/tmp/cifar10" `
  -v "${PWD}\cifar10_splits:/tmp/cifar10_splits" `
  -w /startupkit/startup/ `
  --ipc=host `
  $IMAGE_TAG `
  /bin/bash -c "export PYTHONPATH=/workspace/controller && python -u -m nvflare.private.fed.app.client.client_train -m /startupkit -s fed_client.json --set uid=site-1 secure_train=true config_folder=config org=mediswarm"
```

#### macOS (bash)
```bash
docker rm -f site-1 2>/dev/null
docker run -d --name=site-1 \
  -v "${PWD}/application/provision/provision_output/mediswarm_cifar10_test/prod_00/site-1:/startupkit/" \
  -v "${PWD}/cifar10_data:/tmp/cifar10" \
  -v "${PWD}/cifar10_splits:/tmp/cifar10_splits" \
  -w /startupkit/startup/ \
  --ipc=host \
  "$IMAGE_TAG" \
  /bin/bash -c "export PYTHONPATH=/workspace/controller && python -u -m nvflare.private.fed.app.client.client_train -m /startupkit -s fed_client.json --set uid=site-1 secure_train=true config_folder=config org=mediswarm"
```

### Machine B — site-2

Still in the same terminal as Section 8.
If not, re-set `$FL_ROOT`, `$DOCKER_HUB_USER`, and `$IMAGE_TAG` first.


#### Windows (PowerShell)
```powershell
cd $FL_ROOT

docker rm -f site-2 2>$null
docker run -d --name=site-2 `
  -v "${PWD}\startup_kits\site-2:/startupkit/" `
  -v "${PWD}\cifar10_data:/tmp/cifar10" `
  -v "${PWD}\cifar10_splits:/tmp/cifar10_splits" `
  -w /startupkit/startup/ `
  --ipc=host `
  $IMAGE_TAG `
  /bin/bash -c "export PYTHONPATH=/workspace/controller && python -u -m nvflare.private.fed.app.client.client_train -m /startupkit -s fed_client.json --set uid=site-2 secure_train=true config_folder=config org=mediswarm"
```

#### macOS (bash)
```bash
cd "$FL_ROOT"

docker rm -f site-2 2>/dev/null
docker run -d --name=site-2 \
  -v "${PWD}/startup_kits/site-2:/startupkit/" \
  -v "${PWD}/cifar10_data:/tmp/cifar10" \
  -v "${PWD}/cifar10_splits:/tmp/cifar10_splits" \
  -w /startupkit/startup/ \
  --ipc=host \
  "$IMAGE_TAG" \
  /bin/bash -c "export PYTHONPATH=/workspace/controller && python -u -m nvflare.private.fed.app.client.client_train -m /startupkit -s fed_client.json --set uid=site-2 secure_train=true config_folder=config org=mediswarm"
```

Check each container's logs (`docker logs -f <name>`) — look for `"Successfully registered client"` on the client side, and `"Total clients: 2"` in the server's log once both are up.

---

## 10. Submit the job (Machine A)

### 10a. Launch the admin console

Still in the same window as Sections 2 and 4
If not, re-set `$FL_ROOT` and `$IMAGE_TAG` first.

#### Windows (PowerShell)
```powershell
cd $FL_ROOT

docker rm -f fladmin 2>$null
docker run -it --name=fladmin `
  -v "${PWD}\application\provision\provision_output\mediswarm_cifar10_test\prod_00\admin@mediswarm.local:/startupkit/" `
  -w /startupkit/startup/ `
  --ipc=host `
  $IMAGE_TAG `
  /bin/bash -c "mkdir -p /startupkit/transfer && python3 -m nvflare.fuel.hci.tools.admin -m /startupkit -s fed_admin.json"
```

#### macOS (bash)
```bash
cd "$FL_ROOT"

docker rm -f fladmin 2>/dev/null
docker run -it --name=fladmin \
  -v "${PWD}/application/provision/provision_output/mediswarm_cifar10_test/prod_00/admin@mediswarm.local:/startupkit/" \
  -w /startupkit/startup/ \
  --ipc=host \
  "$IMAGE_TAG" \
  /bin/bash -c "mkdir -p /startupkit/transfer && python3 -m nvflare.fuel.hci.tools.admin -m /startupkit -s fed_admin.json"
```

No `-d` here, this stays in the foreground since it's the interactive console you'll type into. Log in with username `admin@mediswarm.local`.

### 10b. Copy the job directory (new terminal window)

#### Windows (PowerShell)
```powershell
$FL_ROOT = "C:\workspace\mediswarm_cifar10"
cd $FL_ROOT

Copy-Item -Recurse `
  "application\jobs\cifar10" `
  "application\provision\provision_output\mediswarm_cifar10_test\prod_00\admin@mediswarm.local\transfer\cifar10"
```
#### macOS (bash)
```bash
FL_ROOT="$HOME/mediswarm_cifar10"
cd "$FL_ROOT"

cp -R application/jobs/cifar10 \
  application/provision/provision_output/mediswarm_cifar10_test/prod_00/admin@mediswarm.local/transfer/cifar10
```

### 10c. Run in the admin console
`submit_job cifar10`


The console logs itself out after 15 minutes idle — the job keeps running server-side regardless. Reconnect with the same `docker run -it --name=fladmin ...` command from 10a (removing the old container first: `docker rm -f fladmin`).


---

## 11. Get the results (Machine A)

Once `list_jobs` shows `FINISHED:COMPLETED`:

In the admin console:
```
download_job <job-id>
```
This pulls the *entire* job workspace (every site's logs and models)

### Where everything lands

#### Windows (PowerShell)
```powershell
cd $FL_ROOT
$JOB_ID = "<job-id>"
$JOB_DIR = "application\provision\provision_output\mediswarm_cifar10_test\prod_00\admin@mediswarm.local\transfer\$JOB_ID"

# see everything that came down
Get-ChildItem -Recurse $JOB_DIR | Select-Object FullName
```

#### macOS (bash)

```bash
cd "$FL_ROOT"
JOB_ID="<job-id>"
JOB_DIR="application/provision/provision_output/mediswarm_cifar10_test/prod_00/admin@mediswarm.local/transfer/$JOB_ID"

# see everything that came down
find "$JOB_DIR"
```


What you'll find under `$JOB_DIR/workspace/`:

- `meta.json` — job metadata: submit time, participating sites, final status.
- `log.txt` — the full run log, merged server-side.
- `app_server/models/FL_global_model.pt` — the global model after the last training round.
- `app_server/models/best_FL_global_model.pt` — the global model checkpoint from whichever round scored best on validation.
- `app_server/config_fed_server.conf` — a snapshot of the exact server config used for this specific run, useful for confirming what settings actually applied (e.g. `eval_task_timeout`, `num_rounds`).
- `app_site-1/`, `app_site-2/` — each site's own working directory, with its own `log.txt`. 
- `cross_site_val/cross_val_results.json` — the cross-site evaluation numbers 


