# 1. Buat direktori proyek dan masuk
mkdir -p uts-distributed-systems
cd uts-distributed-systems

# 2. Buat struktur folder
mkdir -p primary replica
mkdir -p primary/data
mkdir -p replica/data

# 3. docker-compose.yml
cat > docker-compose.yml <<'EOF'
version: "3.8"

services:
  postgres-primary:
    image: postgres:14
    container_name: postgres-primary
    environment:
      - POSTGRES_PASSWORD=primarypass
      - POSTGRES_USER=postgres
      - PGDATA=/var/lib/postgresql/data
    volumes:
      - ./primary/data:/var/lib/postgresql/data
      - ./primary/postgresql.conf:/var/lib/postgresql/data/postgresql.conf
      - ./primary/pg_hba.conf:/var/lib/postgresql/data/pg_hba.conf
      - ./primary/init-replication.sh:/docker-entrypoint-initdb.d/init-replication.sh:ro
    ports:
      - "5432:5432"
    command: ["-c", "config_file=/var/lib/postgresql/data/postgresql.conf"]

  postgres-replica:
    build: ./replica
    container_name: postgres-replica
    environment:
      - POSTGRES_PASSWORD=primarypass
      - PGDATA=/var/lib/postgresql/data
    volumes:
      - ./replica/data:/var/lib/postgresql/data
      - ./replica/start-replica.sh:/usr/local/bin/start-replica.sh:ro
    depends_on:
      - postgres-primary
    ports:
      - "5433:5432"
EOF

# 4. primary/postgresql.conf
cat > primary/postgresql.conf <<'EOF'
# Primary server config (partial)
listen_addresses = '*'
port = 5432
max_wal_senders = 10
wal_level = replica
synchronous_commit = off
hot_standby = on
archive_mode = off
# Use this file as main config
EOF

# 5. primary/pg_hba.conf
cat > primary/pg_hba.conf <<'EOF'
# Allow local connections
local   all             all                                     trust
host    all             all             0.0.0.0/0               trust

# Allow replication user from replica container (for learning/demo)
host    replication     all             0.0.0.0/0               trust
EOF

# 6. primary/init-replication.sh
cat > primary/init-replication.sh <<'EOF'
#!/bin/bash
set -e

# This script runs only on first container init
psql -v ON_ERROR_STOP=1 --username "$POSTGRES_USER" <<-EOSQL
  -- create replication role
  CREATE ROLE repuser WITH REPLICATION LOGIN PASSWORD 'reppass';
  -- create example db and table
  CREATE DATABASE demo;
  \connect demo
  CREATE TABLE messages (id serial PRIMARY KEY, body text, created_at timestamptz DEFAULT now());
  INSERT INTO messages (body) VALUES ('Hello from primary');
EOSQL
EOF

# 7. replica/Dockerfile
cat > replica/Dockerfile <<'EOF'
FROM postgres:14
COPY start-replica.sh /usr/local/bin/start-replica.sh
RUN chmod +x /usr/local/bin/start-replica.sh
ENTRYPOINT ["/usr/local/bin/start-replica.sh"]
EOF

# 8. replica/start-replica.sh
cat > replica/start-replica.sh <<'EOF'
#!/bin/bash
set -e

PRIMARY_HOST="postgres-primary"
PRIMARY_PORT=5432
REPL_USER="repuser"
REPL_PASSWORD="reppass"
PGDATA=${PGDATA:-/var/lib/postgresql/data}

# Wait primary to be reachable
echo "Waiting for primary ${PRIMARY_HOST}:${PRIMARY_PORT}..."
until pg_isready -h "${PRIMARY_HOST}" -p "${PRIMARY_PORT}" -U postgres >/dev/null 2>&1; do
  sleep 1
done
echo "Primary is ready."

# If PGDATA is empty (first run), perform base backup
if [ ! -s "${PGDATA}/PG_VERSION" ]; then
  echo "Initializing replica data directory with pg_basebackup..."

  # Clean any existing data if present (caution)
  rm -rf "${PGDATA:?}/"*

  export PGPASSWORD=${REPL_PASSWORD}
  pg_basebackup -h "${PRIMARY_HOST}" -p "${PRIMARY_PORT}" -D "${PGDATA}" -U "${REPL_USER}" -Fp -Xs -P -R

  echo "Base backup complete."
else
  echo "PGDATA exists, starting postgres normally."
fi

# Start postgres (use the main postgres entrypoint)
exec docker-entrypoint.sh postgres
EOF

# 9. README.md (opsional, bisa diedit)
cat > README.md <<'EOF'
# uts-distributed-systems

Contoh streaming replication PostgreSQL menggunakan Docker Compose.

Services:
- postgres-primary (host port 5432)
- postgres-replica (host port 5433)

Cara pakai:
1. chmod +x primary/init-replication.sh replica/start-replica.sh
2. docker compose up --build
3. Tes:
   - psql -h localhost -p 5432 -U postgres -d demo -c "select * from messages;"
   - psql -h localhost -p 5433 -U postgres -d demo -c "select * from messages;"
EOF

# 10. .gitignore
cat > .gitignore <<'EOF'
primary/data/
replica/data/
*.log
EOF

# 11. Set executable
chmod +x primary/init-replication.sh
chmod +x replica/start-replica.sh

echo "All files created. Run 'docker compose up --build' to start the demo."
