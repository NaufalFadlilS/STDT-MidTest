mkdir -p uts-distributed-systems
cd uts-distributed-systems

mkdir -p primary replica
mkdir -p primary/data
mkdir -p replica/data

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

cat > primary/postgresql.conf <<'EOF'
listen_addresses = '*'
port = 5432
max_wal_senders = 10
wal_level = replica
synchronous_commit = off
hot_standby = on
archive_mode = off
EOF

cat > primary/pg_hba.conf <<'EOF'
local   all             all                                     trust
host    all             all             0.0.0.0/0               trust
host    replication     all             0.0.0.0/0               trust
EOF

cat > primary/init-replication.sh <<'EOF'
#!/bin/bash
set -e

psql -v ON_ERROR_STOP=1 --username "$POSTGRES_USER" <<-EOSQL
  CREATE ROLE repuser WITH REPLICATION LOGIN PASSWORD 'reppass';
  CREATE DATABASE demo;
  \connect demo
  CREATE TABLE messages (id serial PRIMARY KEY, body text, created_at timestamptz DEFAULT now());
  INSERT INTO messages (body) VALUES ('Hello from primary');
EOSQL
EOF

cat > replica/Dockerfile <<'EOF'
FROM postgres:14
COPY start-replica.sh /usr/local/bin/start-replica.sh
RUN chmod +x /usr/local/bin/start-replica.sh
ENTRYPOINT ["/usr/local/bin/start-replica.sh"]
EOF

cat > replica/start-replica.sh <<'EOF'
#!/bin/bash
set -e

PRIMARY_HOST="postgres-primary"
PRIMARY_PORT=5432
REPL_USER="repuser"
REPL_PASSWORD="reppass"
PGDATA=${PGDATA:-/var/lib/postgresql/data}

echo "Waiting for primary ${PRIMARY_HOST}:${PRIMARY_PORT}..."
until pg_isready -h "${PRIMARY_HOST}" -p "${PRIMARY_PORT}" -U postgres >/dev/null 2>&1; do
  sleep 1
done
echo "Primary is ready."

if [ ! -s "${PGDATA}/PG_VERSION" ]; then
  echo "Initializing replica data directory with pg_basebackup..."

  rm -rf "${PGDATA:?}/"*

  export PGPASSWORD=${REPL_PASSWORD}
  pg_basebackup -h "${PRIMARY_HOST}" -p "${PRIMARY_PORT}" -D "${PGDATA}" -U "${REPL_USER}" -Fp -Xs -P -R

  echo "Base backup complete."
else
  echo "PGDATA exists, starting postgres normally."
fi

exec docker-entrypoint.sh postgres
EOF

cat > README.md <<'EOF'
# uts-distributed-systems

Streaming replication PostgreSQL menggunakan Docker Compose.

## Cara pakai:
1. chmod +x primary/init-replication.sh replica/start-replica.sh
2. docker compose up --build

## Testing:
Primary:
  psql -h localhost -p 5432 -U postgres -d demo -c "select * from messages;"

Replica:
  psql -h localhost -p 5433 -U postgres -d demo -c "select * from messages;"
EOF

cat > .gitignore <<'EOF'
primary/data/
replica/data/
*.log
EOF

chmod +x primary/init-replication.sh
chmod +x replica/start-replica.sh

