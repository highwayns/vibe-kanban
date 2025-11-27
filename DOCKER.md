# Docker Setup Guide for Vibe Kanban

This guide covers how to run Vibe Kanban using Docker and Docker Compose.

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/) (version 20.10 or later)
- [Docker Compose](https://docs.docker.com/compose/install/) (version 2.0 or later)

## Quick Start

### Development Mode (with hot-reload)

1. **Clone the repository**
   ```bash
   git clone <repository-url>
   cd vibe-kanban
   ```

2. **Create environment file**
   ```bash
   cp .env.example .env
   # Edit .env if you need to change default ports
   ```

3. **Start development servers**
   ```bash
   docker-compose up
   ```

4. **Access the application**
   - Frontend: http://localhost:3000
   - Backend API: http://localhost:8080

5. **Stop the servers**
   ```bash
   docker-compose down
   ```

### Production Mode

1. **Build and start production container**
   ```bash
   docker-compose -f docker-compose.prod.yml up -d
   ```

2. **Access the application**
   - Application: http://localhost:3000

3. **View logs**
   ```bash
   docker-compose -f docker-compose.prod.yml logs -f
   ```

4. **Stop the container**
   ```bash
   docker-compose -f docker-compose.prod.yml down
   ```

## Configuration

### Environment Variables

Create a `.env` file in the project root (use `.env.example` as template):

#### Development
```bash
# Frontend port
FRONTEND_PORT=3000

# Backend port
BACKEND_PORT=8080

# Rust logging level
RUST_LOG=debug
```

#### Production
```bash
# Application port
PORT=3000

# PostHog analytics (optional)
POSTHOG_API_KEY=your_key_here
```

### Ports

Default ports can be changed in `.env`:

- **Development:**
  - Frontend: `3000` (configurable via `FRONTEND_PORT`)
  - Backend: `8080` (configurable via `BACKEND_PORT`)

- **Production:**
  - Application: `3000` (configurable via `PORT`)

## Architecture

### Development Setup

The development compose file (`docker-compose.yml`) provides:

- **Backend Container**:
  - Rust server with `cargo-watch` for hot-reload
  - SQLite database
  - Volume mounts for source code
  - Cargo cache for faster rebuilds

- **Frontend Container**:
  - Vite dev server with Hot Module Replacement (HMR)
  - Volume mounts for source code
  - Node modules cache

### Production Setup

The production compose file (`docker-compose.prod.yml`) provides:

- **Single Optimized Container**:
  - Pre-built Rust backend
  - Pre-built React frontend (served by backend)
  - SQLite database
  - Minimal Alpine-based image (~100MB)

## Common Tasks

### Viewing Logs

```bash
# Development - all services
docker-compose logs -f

# Development - specific service
docker-compose logs -f backend
docker-compose logs -f frontend

# Production
docker-compose -f docker-compose.prod.yml logs -f
```

### Rebuilding Containers

```bash
# Development
docker-compose up --build

# Production
docker-compose -f docker-compose.prod.yml up --build
```

### Accessing Container Shell

```bash
# Development - backend
docker-compose exec backend bash

# Development - frontend
docker-compose exec frontend bash

# Production
docker-compose -f docker-compose.prod.yml exec vibe-kanban sh
```

### Database Management

The SQLite database is stored in volumes:

```bash
# Development: List database files
docker-compose exec backend ls -la /app/dev_assets/

# Production: List database files
docker-compose -f docker-compose.prod.yml exec vibe-kanban ls -la /repos/

# Backup production database
docker-compose -f docker-compose.prod.yml exec vibe-kanban \
  cat /repos/db.sqlite > backup.sqlite
```

### Cleaning Up

```bash
# Stop and remove containers (preserves volumes)
docker-compose down

# Stop, remove containers and volumes (deletes data)
docker-compose down -v

# Remove all unused Docker resources
docker system prune -a
```

## Volume Management

### Development Volumes

- `vibe-kanban-repos`: Git repositories and worktrees
- `vibe-kanban-cargo-cache`: Cargo registry cache
- `vibe-kanban-cargo-git`: Cargo git dependencies
- `vibe-kanban-target-cache`: Rust build artifacts
- `vibe-kanban-frontend-node-modules`: Node.js dependencies

### Production Volumes

- `vibe-kanban-prod-repos`: Application data including database

### Backing Up Data

```bash
# Create backup directory
mkdir -p backups

# Backup development repositories
docker run --rm -v vibe-kanban-repos:/data -v $(pwd)/backups:/backup \
  alpine tar czf /backup/repos-backup-$(date +%Y%m%d).tar.gz /data

# Backup production data
docker run --rm -v vibe-kanban-prod-repos:/data -v $(pwd)/backups:/backup \
  alpine tar czf /backup/prod-backup-$(date +%Y%m%d).tar.gz /data
```

### Restoring Data

```bash
# Restore to development volume
docker run --rm -v vibe-kanban-repos:/data -v $(pwd)/backups:/backup \
  alpine tar xzf /backup/repos-backup-YYYYMMDD.tar.gz -C /

# Restore to production volume
docker run --rm -v vibe-kanban-prod-repos:/data -v $(pwd)/backups:/backup \
  alpine tar xzf /backup/prod-backup-YYYYMMDD.tar.gz -C /
```

## Troubleshooting

### Port Already in Use

If you get a "port already in use" error:

```bash
# Find process using the port
lsof -i :3000  # or :8080

# Kill the process
kill -9 <PID>

# Or change the port in .env
echo "FRONTEND_PORT=3001" >> .env
```

### Build Failures

```bash
# Clean build cache
docker-compose build --no-cache

# Remove all containers and rebuild
docker-compose down -v
docker-compose up --build
```

### Hot Reload Not Working

Development mode uses volume mounts for hot-reload. If changes aren't detected:

1. Check volume mounts are working:
   ```bash
   docker-compose exec backend ls -la /app/crates
   ```

2. Restart the containers:
   ```bash
   docker-compose restart
   ```

3. For macOS users, ensure Docker Desktop has file sharing enabled for the project directory.

### Permission Issues

If you encounter permission errors:

```bash
# Fix ownership of volumes (Linux)
docker-compose exec backend chown -R $(id -u):$(id -g) /app

# Or run containers with current user
docker-compose exec --user $(id -u):$(id -g) backend bash
```

### Database Locked Errors

SQLite can have locking issues with multiple processes:

```bash
# Restart the backend service
docker-compose restart backend

# Check for multiple processes accessing the database
docker-compose exec backend ps aux | grep server
```

## Performance Optimization

### Development

For faster development builds:

1. **Use cache mounts** (already configured in `docker-compose.yml`)
2. **Limit resources** in Docker Desktop settings
3. **Use BuildKit**:
   ```bash
   export DOCKER_BUILDKIT=1
   export COMPOSE_DOCKER_CLI_BUILD=1
   ```

### Production

For optimal production performance:

1. **Resource limits** are configured in `docker-compose.prod.yml`
2. **Health checks** ensure service availability
3. **Multi-stage builds** reduce image size

## Remote/Cloud Deployment

For deploying to remote servers with PostgreSQL, see the separate configuration in `crates/remote/`:

```bash
cd crates/remote
docker-compose --env-file .env.remote up --build
```

See `crates/remote/README.md` for more details.

## Integration with Native Development

You can mix Docker and native development:

```bash
# Run only backend in Docker
docker-compose up backend

# Run frontend natively
cd frontend && npm run dev
```

Or vice versa:

```bash
# Run only frontend in Docker
docker-compose up frontend

# Run backend natively
cargo run --bin server
```

## CI/CD Integration

### GitHub Actions Example

```yaml
name: Docker Build

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Build Docker image
        run: docker-compose -f docker-compose.prod.yml build
      - name: Run tests
        run: docker-compose -f docker-compose.prod.yml up -d
```

## Additional Resources

- [Docker Documentation](https://docs.docker.com/)
- [Docker Compose Documentation](https://docs.docker.com/compose/)
- [Vibe Kanban Main README](README.md)
- [Development Guide](CLAUDE.md)
