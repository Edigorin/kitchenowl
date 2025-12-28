# Deployment on LXC Containers (Proxmox, etc.)

If you are deploying KitchenOwl on an LXC container (e.g., Proxmox) and encounter the following error during the Docker build:

```
Cannot change ownership to uid 397546, gid 5000: Invalid argument
```

This is because the standard `Dockerfile` attempts to change ownership of files for the Flutter user, which is restricted in unprivileged LXC containers due to UID mapping.

## Workaround: Backend-Only Build

Since the frontend image is available publicly and rarely needs custom modifications (unless you changed Flutter code), you can build **only the backend** locally on the server and use the official frontend image.

### 1. Build the Backend Image

Navigate to the `backend` directory and build the image:

```bash
cd backend
docker build -t kitchenowl-backend:local-loyalty .
```

This uses the `backend/Dockerfile`, which is a standard Python build and avoids the Flutter permissions issue.

### 2. Configure `docker-compose.yml`

Create or update your `docker-compose.yml` to use your local backend image and the public frontend image:

```yaml
services:
  kitchenowl-backend:
    image: kitchenowl-backend:local-loyalty
    restart: unless-stopped
    depends_on:
      - kitchenowl-db
    environment:
      - JWT_SECRET_KEY=your_secret_key_here
    volumes:
      - kitchenowl-data:/app/data
    
  kitchenowl-frontend:
    image: tombursch/kitchenowl:latest
    restart: unless-stopped
    depends_on:
      - kitchenowl-backend
    ports:
      - "80:80"
    environment:
      - BACKEND_URL=http://kitchenowl-backend:5000

  # ... database and other services ...
```

### 3. Deploy

```bash
docker-compose up -d
```

## Option B: Fixing the Main Dockerfile (Advanced)

If you must build the full image (including frontend) on an LXC container, you need to modify the `Dockerfile` to avoid `chown` errors.

Add this flag before any `COPY` or `RUN` commands that might trigger tar extraction:

```dockerfile
ENV TAR_OPTIONS="--no-same-owner"
```

Or configure your LXC container to be **privileged** (security risk) or enable `nesting`.
