# NeDi Dockerized

![NeDi logo](https://nedi.ch/wp-content/uploads/nedi.png)

A Docker-based deployment of [NeDi](https://www.nedi.ch/) (Network Discovery) — a powerful network management platform for device discovery, SNMP monitoring, syslog collection, and topology mapping.

> Continuing the work of [klinnex](https://github.com/klinnex). Special thanks to Remo Rickli for creating NeDi.

---

## Architecture

The stack consists of four Docker services connected via a private `backend` bridge network:

| Service | Image / Build | Purpose |
|---------|--------------|---------|
| **nedi** | `php:8.1-fpm` (custom build) | Core NeDi application + monitoring daemons |
| **nginx** | `nginx:1.25` (custom build) | Reverse proxy, serves web UI and Adminer |
| **mariadb** | `mariadb:10.11` | Relational database backend |
| **adminer** | `adminer:fastcgi` | Web-based database management UI |

### Exposed Ports

| Port | Protocol | Service |
|------|----------|---------|
| `80` | HTTP | NeDi Web UI |
| `443` | HTTPS | NeDi Web UI (SSL) |
| `514` | UDP/Syslog | Syslog event receiver |
| `8443` | HTTPS | Adminer database UI (SSL) |

### Persistent Volumes

| Volume | Contents |
|--------|---------|
| `nedifiles` | NeDi application, configuration, and RRD data |
| `mysqldata` | MariaDB database files |

---

## Quick Start

**Prerequisites**: Docker with Compose plugin

```bash
git clone https://github.com/mcruise40/nedi-dockerized.git
cd nedi-dockerized
docker compose up -d
```

On first startup the NeDi container will:
1. Wait for MariaDB to become ready
2. Download and install NeDi 2.3C
3. Initialize the database schema
4. Download MAC vendor OUI databases
5. Start PHP-FPM, cron, and monitoring daemons

Allow a few minutes for the initial setup to complete. Check progress with:

```bash
docker compose logs -f nedi
```

Once ready, open **https://\<host\>** in your browser (self-signed certificate).
Adminer is available at **https://\<host\>:8443**.

---

## Configuration

### Environment Variables

All variables can be overridden in `docker-compose.yml`.

**MariaDB**

| Variable | Default | Description |
|----------|---------|-------------|
| `MARIADB_ROOT_PASSWORD` | `N3diRox` | MariaDB root password |

**NeDi**

| Variable | Default | Description |
|----------|---------|-------------|
| `NEDI_DB_HOST` | `mariadb` | Database hostname |
| `NEDI_DB_NAME` | `nedi` | Database name |
| `NEDI_DB_USER` | `root` | Database user |
| `NEDI_DB_PASS` | `N3diRox` | Database password |
| `NEDI_VERSION` | `2.3C` | NeDi version to download |
| `NEDI_SOURCE` | `http://www.nedi.ch/pub` | NeDi download base URL |
| `NEDI_ZIP_PASSWORD` | `RR4JC` | Decryption password for the NeDi zip |
| `TZ` | `Europe/Vaduz` | Container timezone |

> **Security note:** Change the default database password before deploying in any non-isolated environment.

### SSL Certificates

Nginx generates a **self-signed certificate** automatically on first start. To use your own certificates, uncomment and map the `./ssl` volume in `docker-compose.yml` and place your files there.

### Custom Nginx Config

To override the Nginx configuration, uncomment the `./nginx-conf` volume mount in `docker-compose.yml` and place your `.conf` files in that directory.

---

## Monitoring Daemons

Two background daemons run inside the NeDi container:

| Daemon | Script | Purpose |
|--------|--------|---------|
| nedi-monitor | `moni.pl -D` | Polls devices via SNMP/ping on a schedule |
| nedi-syslog | `syslog.pl -Dp 1514` | Receives and stores syslog events on port 1514 |

Both are managed via init.d scripts and start automatically with the container.

---

## Backup

To fully back up the deployment, save both Docker volumes:

```bash
# Example: export volumes to tar archives
docker run --rm -v nedifiles:/data -v $(pwd):/backup alpine \
  tar czf /backup/nedifiles.tar.gz -C /data .

docker run --rm -v mysqldata:/data -v $(pwd):/backup alpine \
  tar czf /backup/mysqldata.tar.gz -C /data .
```

Restore by reversing the process (extract into a fresh named volume before starting the stack).

---

## Acknowledgements

- [klinnex](https://github.com/klinnex) — original dockerized NeDi project this is based on
- [Remo Rickli](https://www.nedi.ch/) — creator of NeDi
