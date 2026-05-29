# OXmaint Database Migration & Setup Guide

This package contains everything needed to set up the OXmaint databases on a new server.

## Contents

| File | Description |
|------|-------------|
| `mysql-8-image.tar.gz` | MySQL 8.0 Docker image (compressed) |
| `mongo-7-image.tar.gz` | MongoDB 7 Docker image (compressed) |
| `mysql_all_databases.sql` | Full MySQL dump (all databases) |
| `mongodump/` | MongoDB dump directory |
| `docker-compose.yml` | Docker Compose file for database services |
| `mongo-init.sh` | MongoDB restore automation script |

## Prerequisites

- **Docker** and **Docker Compose** installed on the target server
- At least **10GB** free disk space (for images, volumes, and data growth)
- Ports **3306** (MySQL) and **27017** (MongoDB) available

### Installing Docker (if needed)

```bash
# Ubuntu/Debian
sudo apt-get update
sudo apt-get install -y docker.io docker-compose-v2

# RHEL/CentOS/Fedora
sudo yum install -y docker docker-compose

# Start Docker
sudo systemctl enable --now docker

# Add your user to docker group (optional, avoids sudo)
sudo usermod -aG docker $USER
# Log out and back in for this to take effect
```

## Quick Start (Recommended)

### Step 1: Load Docker Images

```bash
cd db-export/

# Load the compressed Docker images (one-time)
gunzip -c mysql-8-image.tar.gz | docker load
gunzip -c mongo-7-image.tar.gz | docker load
```

### Step 2: Start Databases with Auto-Restore

```bash
# Start both databases (MySQL auto-restores from the .sql dump)
docker compose up -d

# Wait for MySQL to be healthy (~30 seconds)
docker ps
```

### Step 3: Restore MongoDB (Manual)

MongoDB requires manual restore after first start:

```bash
# Wait 10 seconds for MongoDB to be ready
sleep 10

# Copy dump into container and restore
docker cp mongodump/ oxmaint-mongodb:/tmp/mongodump
docker exec oxmaint-mongodb mongorestore \
    --username oxmaint \
    --password 'OxMongo2024!Secure' \
    --authenticationDatabase admin \
    --drop \
    /tmp/mongodump/
```

### Step 4: Verify

```bash
# Verify MySQL
docker exec oxmaint-mysql mysql -uroot -prootpassword -e "SHOW DATABASES;"

# Verify MongoDB
docker exec oxmaint-mongodb mongosh -u oxmaint -p 'OxMongo2024!Secure' --authenticationDatabase admin --eval "show dbs"
```

## Database Connection Details

### MySQL
| Setting | Value |
|---------|-------|
| Host | `<server-ip>` or `localhost` |
| Port | `3306` |
| User | `root` |
| Password | `rootpassword` |
| Database | `oxmaint_db` |

### MongoDB
| Setting | Value |
|---------|-------|
| Host | `<server-ip>` or `localhost` |
| Port | `27017` |
| User | `oxmaint` |
| Password | `OxMongo2024!Secure` |
| Auth DB | `admin` |

## Connecting the Backend

Update your backend `.env` with:

```env
DB_SERVER=<your-server-ip-or-localhost>
DB_PORT=3306
DB_USER=root
DB_PASSWORD=rootpassword
DB_NAME=oxmaint_db

# MongoDB (used by frontend)
NEXT_PUBLIC_MONGO_DB_URL=mongodb://oxmaint:OxMongo2024!Secure@<your-server-ip>:27017
NEXT_PUBLIC_MONGO_DB_NAME=oxmaint_synapse
```

## Spinning Up the Full Application

### Backend (Go API)

```bash
cd backend/

# Copy and configure environment
cp .env.example .env
# Edit .env with your settings

# Build and run with Docker
docker compose -f docker-compose.local.yml up -d
```

### Frontend (Next.js)

```bash
cd frontend/

# Configure environment
# Edit .env.local with your settings

# Build and run with Docker
docker compose up -d
```

## Management Commands

```bash
# Stop databases
docker compose -f db-export/docker-compose.yml down

# Stop and DELETE all data (fresh start)
docker compose -f db-export/docker-compose.yml down -v

# View logs
docker logs oxmaint-mysql --tail 100
docker logs oxmaint-mongodb --tail 100

# Backup MySQL (regular)
docker exec oxmaint-mysql mysqldump -uroot -prootpassword --all-databases \
    --single-transaction --routines --triggers --events \
    > backup_$(date +%Y%m%d).sql

# Backup MongoDB (regular)
docker exec oxmaint-mongodb mongodump \
    --username oxmaint --password 'OxMongo2024!Secure' \
    --authenticationDatabase admin --out /tmp/backup
docker cp oxmaint-mongodb:/tmp/backup ./mongo_backup_$(date +%Y%m%d)
```

## Security Notes

> ⚠️ **Change default passwords before going to production!**

```bash
# Change MySQL root password
docker exec oxmaint-mysql mysql -uroot -prootpassword -e \
    "ALTER USER 'root'@'%' IDENTIFIED BY 'YourNewSecurePassword!';"

# Change MongoDB admin password
docker exec oxmaint-mongodb mongosh -u oxmaint -p 'OxMongo2024!Secure' \
    --authenticationDatabase admin --eval \
    "db.changeUserPassword('oxmaint', 'YourNewSecurePassword!')"
```

Then update your `.env` files accordingly.

## Troubleshooting

### MySQL won't start
```bash
# Check logs
docker logs oxmaint-mysql

# Common: port already in use
sudo lsof -i :3306
# Kill the process or change the port in docker-compose.yml
```

### MongoDB restore fails with auth error
```bash
# Make sure you're using the correct credentials
docker exec oxmaint-mongodb mongosh --eval "db.runCommand({connectionStatus: 1})"
```

### Port conflicts with existing MySQL/MongoDB
Edit `docker-compose.yml` and change the left-side port number:
```yaml
ports:
  - "3307:3306"  # Maps host 3307 to container 3306
```

## Databases Included

### MySQL (`oxmaint_db` and related)
- Analytics, Assets, Communication, Core
- Inspections, Maintenance, Reports, Scheduling
- Security, StaticScreens, Subscription

### MongoDB
- `oxmaint_synapse` — conversations, feedback, org_context
- `oxmaint_logbook` — shifts, shift_tasks, shift_messages, routeinspectionnews
