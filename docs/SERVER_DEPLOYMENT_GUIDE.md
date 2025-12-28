# Server Deployment & Security Guide

This guide covers setting up a fresh Linux server (Ubuntu/Debian), creating a secure non-root user, and deploying KitchenOwl.

## Part 1: Create a Non-Root User

Running applications as `root` is a security risk. Follow these steps to create a dedicated user.

### 1. Create the User
Login as root, then create a new user (replace `kitchenuser` with your desired username):

```bash
adduser kitchenuser
```
Follow the prompts to set a password.

### 2. Grant Sudo Privileges
Add the user to the `sudo` group so they can run administrative commands:

```bash
usermod -aG sudo kitchenuser
```

### 3. Switch to the New User
Log out of root and log in as the new user:

```bash
su - kitchenuser
```

### 4. (Optional) Setup SSH Key Authentication
For better security, copy your local machine's SSH key to the server:

**On your local machine (PowerShell/Terminal):**
```bash
ssh-copy-id kitchenuser@your-server-ip
```

If you don't have an SSH key yet, generate one first with `ssh-keygen -t ed25519`.

---

## Part 2: Install Docker & Prerequisites

### 1. Install Docker Engine
Run this convenience script to install Docker and Docker Compose:

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```

### 2. Allow Non-Root Docker Access
Add your user to the `docker` group so you don't have to type `sudo` for every docker command:

```bash
sudo usermod -aG docker $USER
```

**Important:** You must log out and log back in for this to take effect!
```bash
exit
# ssh back in as kitchenuser
```

### 3. Verify Installation
```bash
docker run hello-world
```

---

## Part 3: Deploy KitchenOwl

### 1. Clone the Repository
```bash
# Install git if needed
sudo apt-get update && sudo apt-get install -y git

# Clone your fork (or the main repo)
git clone https://github.com/Edigorin/kitchenowl.git
cd kitchenowl
```

### 2. Prepare for Deployment
Since you are on LXC/Proxmox and want to include the latest frontend changes:

1.  **Uncomment uWSGI**: Edit `backend/pyproject.toml` and uncomment the uWSGI lines (Lines 43-44).
    ```bash
    nano backend/pyproject.toml
    # Remove # from uWSGI lines to enable:
    # "uWSGI>=2.0.28",
    # "uwsgi-tools>=1.1.1",
    ```

2.  **Build the Full Image**:
    I have updated the `Dockerfile` with `ENV TAR_OPTIONS="--no-same-owner"`, which allows the Flutter build to work on LXC. All you need to do is build it:

    ```bash
    # Build the full image (frontend + backend)
    # This may take 10-20 minutes and requires ~4GB RAM
    docker build -t kitchenowl:local-loyalty .
    ```

### 3. Configure `docker-compose.yml`

Create/Edit your `docker-compose-local.yml` to use your local image for BOTH services (it contains both).

```yaml
version: '3'
services:
  kitchenowl:
    image: kitchenowl:local-loyalty
    restart: unless-stopped
    ports:
      - "80:8080"
    environment:
      - JWT_SECRET_KEY=change_this_to_a_secure_random_string
    volumes:
      - ./data:/data
```

*Note: The combined image serves both frontend and backend.*

### 4. Start the Server
```bash
docker compose -f docker-compose-local.yml up -d
```

### 5. Verify
Check logs:
```bash
docker compose -f docker-compose-local.yml logs -f
```

Visit `http://your-server-ip` in your browser.
