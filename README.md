# Zabbix High Availability Setup with PostgreSQL Replication

## **Introduction**

This comprehensive guide provides step-by-step instructions for setting up Zabbix 7.0 in a high-availability (HA) configuration with PostgreSQL replication. The setup includes two Zabbix servers (one master and one failover) to ensure uninterrupted monitoring. Additionally, we configure database synchronization over the internet or local network and include tuning recommendations for large deployments to optimize performance and scalability.

---

## **Prerequisites**

Before starting, ensure the following requirements are met:

1. **Hardware and Network:**
   - Two dedicated servers with both internet and local network access.
   - Sufficient CPU, memory, and disk resources to support Zabbix and PostgreSQL.

2. **Software:**
   - Docker and Docker Compose installed on both servers.
   - DNS and Cloudflare configured to proxy requests to the Zabbix web interface.

3. **Skills:**
   - Basic knowledge of Docker, PostgreSQL, and system administration.

---

## **Folder Structure**

Organize your setup files as follows:

```plaintext
zabbix-setup/
├── srv01.zabb.vaheed.net/
│   ├── docker-compose.yml
│   ├── .env
│   ├── zbx.env
│   ├── pg.env
├── srv02.zabb.vaheed.net/
│   ├── docker-compose.yml
│   ├── .env
│   ├── zbx.env
│   ├── pg.env
├── prx01.zabb.vaheed.net/
│   ├── docker-compose.yml
│   ├── .env
├── prx02.zabb.vaheed.net/
│   ├── docker-compose.yml
│   ├── .env
├── prx03.zabb.vaheed.net/
    ├── docker-compose.yml
    ├── .env
```

---

## **Step 0: Requirements Check and Setup Script (Optional!)**

Create a Bash script to automate the initial setup on each server. This script will:

1. Check if the operating system meets the requirements.
2. Install the latest versions of Docker and Docker Compose.
3. Add `daemon.json` configuration for the Docker registry:
   ```json
   {
     "registry-mirrors": ["https://registry.vaheed.net:2053"]
   }
   ```
4. Create the project folder structure and prepare necessary files.
5. Prompt the user for input to configure server-specific details, such as:
   - Server role (e.g., `srv` or `prx`).
   - Server IPs.
   - Environment variables.
6. Generate secure passwords and user credentials as needed.

### Example Script

```bash
#!/bin/bash

set -e

# Check and install Docker if not present
if ! [ -x "$(command -v docker)" ]; then
  echo "Installing Docker..."
  curl -fsSL https://get.docker.com | bash
fi

# Check and install Docker Compose if not present
if ! [ -x "$(command -v docker-compose)" ]; then
  echo "Installing Docker Compose..."
  curl -L "https://github.com/docker/compose/releases/download/$(curl -s https://api.github.com/repos/docker/compose/releases/latest | grep tag_name | cut -d '"' -f 4)/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
  chmod +x /usr/local/bin/docker-compose
fi

# Configure Docker daemon.json
mkdir -p /etc/docker
cat <<EOF > /etc/docker/daemon.json
{
  "registry-mirrors": ["https://registry.vaheed.net:2053"]
}
EOF
systemctl restart docker

# Create project structure
read -p "Enter project directory name: " project_dir
mkdir -p "/$project_dir/zbx_env"

# Get user inputs
read -p "Enter server role (srv or prx): " server_role
read -p "Enter server IP address: " server_ip
read -p "Enter hostname (e.g., srv01.zbx.vaheed.net): " hostname

# Generate environment files
if [ "$server_role" == "srv" ]; then
  cat <<EOT > /$project_dir/zbx_env/.env
POSTGRES_USER=zabbix
POSTGRES_PASSWORD=$(openssl rand -base64 12)
POSTGRES_DB=zabbix_db
DB_REPLICATION_USER=replicator
DB_REPLICATION_PASSWORD=$(openssl rand -base64 12)

ZBX_SERVER_NAME=$hostname
ZBX_SERVER_PORT=10051

HA_NODE_NAME=$hostname
HA_NODE_PASS=$(openssl rand -base64 12)

ZBX_WEB_PORT=8080
EOT
else
  cat <<EOT > /$project_dir/zbx_env/.env
ZBX_PROXY_NAME=$hostname
ZBX_SERVER_HOST=srv01.zbx.vaheed.net,srv02.zbx.vaheed.net
ZBX_PROXY_PORT=10051
EOT
fi

# Completion message
echo "Setup complete. Copy this folder to the appropriate server and run the script there."
```

---

## **Step 1: Prepare Environment Files**

Create `.env` files for all servers and proxies to store environment variables securely.

### srv01.zabb.vaheed.net

```plaintext
POSTGRES_USER=zabbix
POSTGRES_PASSWORD=secure_password
POSTGRES_DB=zabbix_db
DB_REPLICATION_USER=replicator
DB_REPLICATION_PASSWORD=replication_password

ZBX_SERVER_NAME=srv01.zabb.vaheed.net
ZBX_SERVER_PORT=10051

HA_NODE_NAME=srv01.zabb.vaheed.net
HA_NODE_PASS=ha_secure_password

ZBX_WEB_PORT=8080
```

### srv02.zabb.vaheed.net

```plaintext
POSTGRES_USER=zabbix
POSTGRES_PASSWORD=secure_password
POSTGRES_DB=zabbix_db
DB_REPLICATION_USER=replicator
DB_REPLICATION_PASSWORD=replication_password

ZBX_SERVER_NAME=srv02.zabb.vaheed.net
ZBX_SERVER_PORT=10051

HA_NODE_NAME=srv02.zabb.vaheed.net
HA_NODE_PASS=ha_secure_password

ZBX_WEB_PORT=8080
```

### prx01.zabb.vaheed.net

```plaintext
ZBX_PROXY_NAME=prx01.zabb.vaheed.net
ZBX_SERVER_HOST=srv01.zabb.vaheed.net,srv02.zabb.vaheed.net
ZBX_PROXY_PORT=10051
```

### prx02.zabb.vaheed.net

```plaintext
ZBX_PROXY_NAME=prx02.zabb.vaheed.net
ZBX_SERVER_HOST=srv01.zabb.vaheed.net,srv02.zabb.vaheed.net
ZBX_PROXY_PORT=10051
```

### prx03.zabb.vaheed.net

```plaintext
ZBX_PROXY_NAME=prx03.zabb.vaheed.net
ZBX_SERVER_HOST=srv01.zabb.vaheed.net,srv02.zabb.vaheed.net
ZBX_PROXY_PORT=10051
```

---

## **Step 2: Configure Docker Compose Files**

### srv01.zabb.vaheed.net

```yaml
version: '3.5'

services:
  postgres:
    image: timescale/timescaledb:pg16-latest
    container_name: zbx_postgres
    restart: unless-stopped
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
      TIMESCALEDB_TELEMETRY: "off"
    volumes:
      - pgdata:/var/lib/postgresql/data
    networks:
      - zbx-net

  zabbix-server:
    image: zabbix/zabbix-server-pgsql:alpine-7.0-latest
    container_name: zbx_server
    restart: unless-stopped
    depends_on:
      - postgres
    environment:
      DB_SERVER_HOST: postgres
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
      HANodeName: ${HA_NODE_NAME}
      HANodePass: ${HA_NODE_PASS}
    networks:
      - zbx-net

  zabbix-web:
    image: zabbix/zabbix-web-nginx-pgsql:alpine-7.0-latest
    container_name: zbx_web
    restart: unless-stopped
    depends_on:
      - zabbix-server
    environment:
      DB_SERVER_HOST: postgres
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    ports:
      - "${ZBX_WEB_PORT}:8080"
    networks:
      - zbx-net

volumes:
  pgdata:

networks:
  zbx-net:
    driver: bridge
```

### srv02.zabb.vaheed.net

The `docker-compose.yml` file for srv02.zabb.vaheed.net is identical to srv01.zabb.vaheed.net, except:
- `HANodeName` and `HANodePass` must reflect srv02.zabb.vaheed.net's values.

### Proxy Nodes

Each proxy node uses a similar `docker-compose.yml` file:

```yaml
version: '3.5'

services:
  zabbix-proxy:
    image: zabbix/zabbix-proxy-sqlite3:alpine-7.0-latest
    container_name: zbx_proxy
    restart: unless-stopped
    environment:
      ZBX_PROXY_NAME: ${ZBX_PROXY_NAME}
      ZBX_SERVER_HOST: ${ZBX_SERVER_HOST}
      ZBX_PROXY_PORT: ${ZBX_PROXY_PORT}
    networks:
      - zbx-net

networks:
  zbx-net:
    external: true
```

---

## **Step 3: Configure PostgreSQL Replication**

### Primary Database (srv01.zabb.vaheed.net)

1. Modify `postgresql.conf` to enable replication:
   ```plaintext
   wal_level = replica
   max_wal_senders = 5
   max_replication_slots = 5
   ```

2. Add a replication user:
   ```sql
   CREATE USER replicator WITH REPLICATION ENCRYPTED PASSWORD 'replication_password';
   ```

3. Update `pg_hba.conf` to allow replication:
   ```plaintext
   host replication replicator 0.0.0.0/0 md5
   ```

### Standby Database (srv02.zabb.vaheed.net)

1. Set recovery settings in `postgresql.conf`:
   ```plaintext
   primary_conninfo = 'host=srv01.zabb.vaheed.net user=replicator password=replication_password'
   standby_mode = on
   ```

2. Use `pg_basebackup` to clone the primary database:
   ```bash
   pg_basebackup -h srv01.zabb.vaheed.net -D /var/lib/postgresql/data -U replicator -v -P
   ```

---

## **Step 4: Configure Zabbix HA**

1. Create a `zbx.env` file with the following content:
   ```plaintext
   ZBX_SERVER_HOST=srv01.zabb.vaheed.net,srv02.zabb.vaheed.net
   ZBX_SERVER_PORT=10051
   HANodeName=${HA_NODE_NAME}
   HANodePass=${HA_NODE_PASS}
   ```

2. Place the `zbx.env` file on both servers in the appropriate directory.

---

## **Step 5: Set Up Cloudflare Proxy**

1. **DNS Configuration:**
   - Create A records for `srv01.zabb.vaheed.net`, `srv02.zabb.vaheed.net`, `prx01.zabb.vaheed.net`, `prx02.zabb.vaheed.net`, and `prx03.zabb.vaheed.net`.

2. **Failover Configuration:**
   - Use Cloudflare's load balancing feature or configure a failover script to handle server outages.

3. **SSL Configuration:**
   - Use Cloudflare's SSL/TLS encryption to secure connections. Configure Zabbix to serve over HTTP since Cloudflare manages the SSL termination.

---

## **Tuning for Large Deployments**

### PostgreSQL Optimization

1. **Memory Settings:**
   ```plaintext
   shared_buffers = 4GB
   work_mem = 128MB
   maintenance_work_mem = 2GB
   ```

2. **Write-Ahead Log (WAL) Settings:**
   ```plaintext
   wal_buffers = 32MB
   checkpoint_completion_target = 0.9
   ```

3. **Connection Management:**
   ```plaintext
   max_connections = 1000
   ```

### Zabbix Optimization

1. **Cache Settings in `zabbix_server.conf`:**
   ```plaintext
   CacheSize=512M
   HistoryCacheSize=256M
   HistoryTextCacheSize=128M
   ValueCacheSize=256M
   ```

2. **Enable More Preprocessors:**
   ```plaintext
   StartPreprocessors=16
   ```

3. **Database Housekeeping:**
   - Regularly archive historical data to optimize query performance.

---

## **Step 6: Testing and Maintenance**

1. **Failover Test:**
   - Simulate a failure by stopping Zabbix Server on srv01.zabb.vaheed.net. Verify that srv02.zabb.vaheed.net becomes the active master.

2. **Replication Test:**
   - Insert data into the primary database and ensure it replicates to the standby database.

3. **Monitoring:**
   - Continuously monitor Zabbix and PostgreSQL performance metrics using built-in tools or external monitoring solutions.

---

This setup ensures robust high availability, reliable database replication, and optimal performance for large-scale deployments. If additional customization is required, feel free to adjust the configuration or reach out for assistance!

