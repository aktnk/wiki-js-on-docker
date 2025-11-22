# Wiki.js on Docker

Production-ready Wiki.js deployment using Docker Compose with enhanced security features.

## Features

- PostgreSQL 15 Alpine as database backend
- Wiki.js 2 latest version
- Environment variable-based credential management
- JSON logging with automatic rotation
- Health checks for both services
- Isolated Docker network
- Automatic restart on failure

## Prerequisites

- Docker Engine 20.10+
- Docker Compose V2
- Minimum 2GB RAM
- Minimum 10GB disk space

## Quick Start

### 1. Initial Setup

```bash
# Clone or navigate to the project directory
cd /path/to/wikijs

# Copy the environment template
cp .env.example .env

# Generate a strong password
openssl rand -base64 32

# Edit .env and update POSTGRES_PASSWORD and DB_PASS with the generated password
nano .env
```

### 2. Start the Stack

```bash
# Start services in detached mode
docker compose up -d

# Check service status
docker compose ps

# Follow the logs
docker compose logs -f
```

### 3. Access Wiki.js

Open your browser and navigate to:
```
http://localhost
```

Follow the on-screen setup wizard to configure your Wiki.js instance.

## Configuration

### Environment Variables

All sensitive configuration is stored in `.env` file (not committed to git):

| Variable | Description | Example |
|----------|-------------|---------|
| `POSTGRES_DB` | Database name | `wiki` |
| `POSTGRES_USER` | Database user | `wikijs` |
| `POSTGRES_PASSWORD` | Database password | `<strong-random-password>` |
| `DB_TYPE` | Database type | `postgres` |
| `DB_HOST` | Database hostname | `db` |
| `DB_PORT` | Database port | `5432` |
| `DB_USER` | Wiki.js DB user | `wikijs` |
| `DB_PASS` | Wiki.js DB password | `<strong-random-password>` |
| `DB_NAME` | Wiki.js DB name | `wiki` |

**Important:** `POSTGRES_PASSWORD` and `DB_PASS` must be identical.

## Common Operations

### View Logs

```bash
# All services
docker compose logs -f

# Specific service
docker compose logs -f wiki
docker compose logs -f db
```

### Restart Services

```bash
# Restart all services
docker compose restart

# Restart specific service
docker compose restart wiki
docker compose restart db
```

### Stop Services

```bash
# Stop and remove containers (data persists)
docker compose down

# Stop and remove containers and volumes (removes all data)
docker compose down -v
```

### Update Images

```bash
# Pull latest images
docker compose pull

# Recreate containers with new images
docker compose up -d
```

### Backup Database

```bash
# Create a backup
docker compose exec db pg_dump -U wikijs wiki > backup_$(date +%Y%m%d_%H%M%S).sql

# Restore from backup
docker compose exec -T db psql -U wikijs wiki < backup_20231122_120000.sql
```

## Health Checks

The stack includes automatic health monitoring:

- **Database**: Checked every 10 seconds using `pg_isready`
- **Wiki.js**: Checked every 30 seconds via `/healthz` endpoint

View health status:
```bash
docker compose ps
```

## Network Architecture

```
┌─────────────────────────────────────────┐
│         Host Machine (Port 80)          │
└──────────────────┬──────────────────────┘
                   │
         ┌─────────▼─────────┐
         │   wiki-network    │
         │   (Bridge Mode)   │
         └─────────┬─────────┘
                   │
         ┌─────────┴─────────┐
         │                   │
    ┌────▼────┐         ┌───▼───┐
    │  Wiki.js│◄────────┤   DB  │
    │  :3000  │         │ :5432 │
    └─────────┘         └───────┘
```

- PostgreSQL is only accessible to Wiki.js container
- Wiki.js is exposed on host port 80
- Internal communication via `wiki-network`

## Logging

Logs are stored in JSON format with automatic rotation:

- **Max size per file**: 10 MB
- **Max files retained**: 3
- **Total max log size**: ~30 MB

Access logs:
```bash
# Via docker compose
docker compose logs

# Via docker inspect (find log location)
docker inspect <container-id> | grep LogPath
```

## Security Considerations

### Implemented

✅ Environment variable-based secrets
✅ Isolated Docker network
✅ Health monitoring
✅ Log rotation
✅ Strong random passwords
✅ `.env` excluded from git

### Recommended Enhancements

- [ ] Use Docker secrets for swarm mode
- [ ] Implement reverse proxy (Nginx/Traefik) with HTTPS
- [ ] Enable PostgreSQL SSL/TLS connections
- [ ] Set resource limits (CPU/memory)
- [ ] Implement automated backup strategy
- [ ] Configure firewall rules on host
- [ ] Use read-only root filesystem where possible

## Troubleshooting

### Services won't start

```bash
# Check logs for errors
docker compose logs

# Verify .env file exists and contains all variables
cat .env

# Check if ports are already in use
sudo netstat -tulpn | grep :80
```

### Database connection errors

```bash
# Verify database is healthy
docker compose ps

# Check database logs
docker compose logs db

# Ensure credentials match in .env
grep PASS .env
```

### Wiki.js health check failing

```bash
# Check Wiki.js logs
docker compose logs wiki

# Verify database connection
docker compose exec wiki wget -O- http://localhost:3000/healthz

# Restart Wiki.js
docker compose restart wiki
```

### Password mismatch after migration

If you have existing data with old credentials:

```bash
# Option 1: Update database password
docker compose exec db psql -U wikijs -d wiki
# ALTER USER wikijs WITH PASSWORD 'new-password-from-env';

# Option 2: Fresh start
docker compose down -v
rm -rf data/
docker compose up -d
```

## Data Persistence

PostgreSQL data is stored in `./data` directory:

```
wikijs/
├── data/              # PostgreSQL data (gitignored)
│   ├── base/
│   ├── global/
│   └── pg_wal/
├── .env              # Secrets (gitignored)
├── .env.example      # Template
├── compose.yaml      # Docker Compose config
└── README.md         # This file
```

**Important**: The `data/` directory is excluded from git. Ensure you have proper backup strategy.

## Migration from Old Setup

If migrating from the previous setup with hardcoded credentials:

```bash
# 1. Stop the old stack
docker compose down

# 2. Update .env with OLD password temporarily
echo "POSTGRES_PASSWORD=wikijsrocks" >> .env
echo "DB_PASS=wikijsrocks" >> .env

# 3. Start with new compose.yaml
docker compose up -d

# 4. Change password in database
docker compose exec db psql -U wikijs -d wiki -c \
  "ALTER USER wikijs WITH PASSWORD 'new-strong-password';"

# 5. Update .env with new password
nano .env

# 6. Restart services
docker compose restart
```

## Official Documentation

- [Wiki.js Official Docs](https://docs.requarks.io/)
- [Wiki.js Docker Installation](https://docs.requarks.io/install/docker)
- [PostgreSQL Docker Hub](https://hub.docker.com/_/postgres)

## License

This configuration is provided as-is. Wiki.js is licensed under AGPL-3.0.

## Support

For issues with:
- **This setup**: Open an issue in this repository
- **Wiki.js**: Visit [Wiki.js GitHub](https://github.com/requarks/wiki)
- **Docker**: Visit [Docker Documentation](https://docs.docker.com/)
