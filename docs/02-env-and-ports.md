# EpicBook Environment Variables, Ports, and Health Checks

## 1. Environment Variables

The following environment-variable values were identified for the EpicBook application and its MySQL database.

| Variable | Value | Description |
|---|---|---|
| `DB_NAME` | `` | MySQL database name |
| `DB_USER` | `` | MySQL username |
| `DB_PASS` | Empty string (`""`) | MySQL password |
| `DB_HOST` | `` | Database host as supplied; must be verified for Docker networking |
| `DB_DIALECT` | `` | Database dialect used by the application |

**Security note:** An empty database password is not recommended outside a disposable local development environment. Avoid committing real credentials to source control. Use environment files excluded by `.gitignore` or a secrets manager.

**Docker networking note:** If EpicBook and MySQL run in separate Compose containers, `127.0.0.1` inside the application container points to that same application container, not the MySQL container. Use the Compose service name, such as `mysql`, as `DB_HOST` when connecting over the Compose network.

## 2. Internal Ports

| Service | Internal port | Purpose |
|---|---:|---|
| EpicBook Node.js application | `3001` | Serves the application and handles API requests |
| MySQL database | `3306` | Accepts database connections from the application |

Internal ports are used for communication between containers. Only publish ports to the host when external access is required.

## 3. Persistent Data Location

- **Docker volume:** `mysql_data`
- **Container mount location:** `/var/lib/mysql`
- **Purpose:** Stores MySQL database files persistently across container restarts and container recreation, provided the volume is retained.

The volume is declared in the Docker Compose configuration:

```yaml
services:
  mysql:
    volumes:
      - mysql_data:/var/lib/mysql

volumes:
  mysql_data:
```

Do not remove the volume when stopping or recreating containers if the database data must be preserved. Maintain backups for important data.

## 4. Service Health Checks

### MySQL health check

The Compose configuration checks whether MySQL responds to `mysqladmin ping`:

```yaml
healthcheck:
  test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
  interval: 10s
  timeout: 5s
  retries: 5
```

Check the container health status with:

```bash
docker compose ps
docker inspect mysql_container --format='{{.State.Health.Status}}'
```

### NGINX configuration check

If NGINX is installed in the relevant container, validate its configuration with:

```bash
nginx -t
```

This checks NGINX configuration syntax and referenced configuration files. It does not verify application or database health.

### EpicBook application check

Test the application endpoint from the host, assuming port `3001` is published:

```bash
curl -i --max-time 5 http://127.0.0.1:3001/
```

A successful HTTP response confirms that the endpoint responds, though the expected status depends on the route. For a dedicated health check, implement an application endpoint such as `/health` and test it directly.

## 5. Operational Summary

- Application port: `3001`
- MySQL port: `3306`
- Database name: `epicbook`
- Database persistence: `mysql_data`
- MySQL health check: `mysqladmin ping`
- NGINX configuration validation: `nginx -t`
- Application endpoint test: `curl`

