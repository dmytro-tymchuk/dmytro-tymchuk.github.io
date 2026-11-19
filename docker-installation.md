# Deploy Uptime Kuma with Docker Compose

## 1. Open the Proxmox VM

Opened the course Proxmox dashboard and selected the existing VM
`33534477 (test)`. Opened **Console** and logged in with my Linux account.

## 2. Check the operating system

```bash
cat /etc/os-release
```

Checks the Linux distribution, version, and release codename.
Returned **Ubuntu 20.04.6 LTS (Focal Fossa)** with the codename `focal`.

Used `focal` to identify the Docker package repository for this VM.

## 3. Install Docker Engine and Docker Compose

### Prepare the VM

```bash
sudo apt update
sudo apt install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
```

Refreshed Ubuntu's package list. APT reported that `ca-certificates`
and `curl` were already installed. Created `/etc/apt/keyrings` to
store Docker's repository signing key.

### Download Docker's signing key

```bash
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
```

The first download failed with `curl: (6) Could not resolve host`.
A retry timed out during hostname resolution, so I checked which
addresses the VM could resolve.

```bash
getent hosts download.docker.com
getent ahostsv4 archive.ubuntu.com
getent ahostsv4 download.docker.com
```

The first lookup returned IPv6 addresses for Docker's download server.
The next two lookups returned IPv4 addresses for both servers. A retry
with `curl -4` still failed to resolve Docker's hostname.

```bash
sudo curl -4 -fsSL --connect-timeout 10 --max-time 20 --resolve download.docker.com:443:13.227.47.120 https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
ls -lh /etc/apt/keyrings/docker.asc
```

Used `--resolve` to supply an IPv4 address from the earlier lookup
for this download. Curl still verified the HTTPS certificate against
`download.docker.com`. The download succeeded, and `ls` showed a `3.8K`
file with permissions `-rw-r--r--`, which made it readable by APT.
The IP address was specific to this troubleshooting session.

### Add the Docker repository

```bash
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu focal stable" | sudo tee /etc/apt/sources.list.d/docker.list
```

Created `/etc/apt/sources.list.d/docker.list` for Ubuntu 20.04 (`focal`).
The VM reported `amd64` as its architecture. The `signed-by` option
tells APT which key to use to verify the repository.

### Install the packages

```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

Refreshed the package list and installed Docker Engine `28.1.1`,
the Docker CLI, containerd `1.7.27`, and the Docker Compose plugin
`2.35.1`. The installation output showed that the packages finished
setting up and the shell prompt returned.

## 4. Verify Docker and Docker Compose

```bash
sudo docker version
sudo docker compose version
sudo docker run --rm hello-world
```

Verified that the Docker client could communicate with the Docker
Engine. Both reported version `28.1.1`, and Docker Compose reported
`v2.35.1`.

The first `hello-world` attempt failed before the container started.
Docker tried to pull the image from Docker Hub, but the DNS lookup
for `registry-1.docker.io` timed out.

## 5. Deploy Uptime Kuma with Docker Compose

Connected to my Ubuntu VM on Proxmox over SSH from macOS Terminal.
Created a project directory and moved into it.

```bash
mkdir -p ~/uptime-kuma
cd ~/uptime-kuma
```

Created `docker-compose.yml` with the following configuration.

```yaml
services:
  uptime-kuma:
    image: louislam/uptime-kuma:2
    restart: unless-stopped
    ports:
      - "127.0.0.1:3001:3001"
    volumes:
      - uptime-kuma-data:/app/data

volumes:
  uptime-kuma-data:
```

Used the prebuilt Uptime Kuma version 2 image from Docker Hub. The
`unless-stopped` policy lets Docker restart the container automatically
unless it was manually stopped. The named volume stores application
data at `/app/data` so it persists if the container is replaced.
Published port `3001` on the VM's loopback address for access through
an SSH tunnel.

```bash
sudo docker compose config
```

Validated the Compose file and checked the image, port mapping, and
volume mount in the output.

```bash
sudo docker compose up -d
sudo docker compose ps
```

Pulled the image and started Uptime Kuma in the background. Compose
created the project network and named volume. The status output showed
the container was **Up (healthy)** and confirmed the port mapping
`127.0.0.1:3001->3001/tcp`.

## 6. Open the dashboard and verify a monitor

Opened a second Terminal window on my Mac and started an SSH tunnel.
Replace `YOUR_USER` and `VM_IP` with your Linux username and the IP
address of your Proxmox VM.

```bash
ssh -N -L 127.0.0.1:3001:127.0.0.1:3001 YOUR_USER@VM_IP
```

Forwarded port `3001` on my Mac to port `3001` on the VM. Left the
tunnel open and visited `http://127.0.0.1:3001` in my browser.
Created an Uptime Kuma administrator account during setup.

Clicked **Add New Monitor** and used these settings:

- **Monitor Type** — HTTP(s)
- **Friendly Name** — Example Website
- **URL** — `https://example.com`
- **Check interval** — 60 seconds

Saved the monitor and verified that the dashboard showed **Up** with
an HTTP `200 OK` response.
