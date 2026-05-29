# OXmaint Database Export

This directory contains a complete database export for the OXmaint Spanish Portal.

## Files

| File | Description |
|------|-------------|
| `oxmaint-database-export.tar.gz.part-aa` through `part-aj` | Split archive (10 parts, 50MB each) |
| `OXMAINT-DATABASE-SETUP-GUIDE.md` | Full setup & deployment guide |

## How to Reassemble

```bash
# Concatenate all parts back into the full archive
cat oxmaint-database-export.tar.gz.part-* > oxmaint-database-export.tar.gz

# Verify size (~475 MB)
ls -lh oxmaint-database-export.tar.gz

# Extract
tar xzf oxmaint-database-export.tar.gz
cd db-export/
```

Then follow the **OXMAINT-DATABASE-SETUP-GUIDE.md** for deployment instructions.

## Contents of the Export

| Component | Description |
|-----------|-------------|
| `mysql-8-image.tar.gz` | MySQL 8.0 Docker image |
| `mongo-7-image.tar.gz` | MongoDB 7 Docker image |
| `mysql_all_databases.sql` | Full MySQL dump (all databases) |
| `mongodump/` | MongoDB dump directory |
| `docker-compose.yml` | One-command database setup |
| `SETUP-GUIDE.md` | Step-by-step deployment guide |

## Quick Start (after reassembly)

```bash
cd db-export/

# Load Docker images
gunzip -c mysql-8-image.tar.gz | docker load
gunzip -c mongo-7-image.tar.gz | docker load

# Start databases (MySQL auto-restores)
docker compose up -d

# Restore MongoDB (one manual step)
docker cp mongodump/ oxmaint-mongodb:/tmp/mongodump
docker exec oxmaint-mongodb mongorestore \
    --username oxmaint --password 'OxMongo2024!Secure' \
    --authenticationDatabase admin --drop /tmp/mongodump/
```

## Source

Exported from the OXmaint Spanish Portal development server on 2026-05-29.
Includes MySQL 8.4 (`oxmaint_db` + related databases) and MongoDB 7 data.
