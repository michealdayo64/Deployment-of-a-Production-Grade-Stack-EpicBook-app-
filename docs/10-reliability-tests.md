# EpicBook Reliability Tests

## 1. Purpose

This document describes the reliability tests used to validate the EpicBook application after deployment through the Azure DevOps CI/CD pipeline.

The tests verify that the frontend, backend, MySQL database, and Nginx reverse proxy operate correctly, and that the application remains accessible after a deployment or container restart.

## 2. Test Environment

The reliability tests use the following components:

- Azure DevOps Pipelines for automated deployment.
- Docker Hub for storing frontend and backend images.
- AWS EC2 as the deployment host.
- Docker Compose for managing application containers.
- Nginx for reverse proxy and HTTP traffic routing.
- MySQL for application data persistence.
- An HTTP health endpoint, such as `/health`, for deployment verification.

## 3. Reliability Test Cases

### Test 1: Container Availability

**Objective:** Confirm that the expected containers start successfully after deployment.

**Procedure:**

Connect to the EC2 instance and run:

```bash
cd ~/theepicbook
docker compose --env-file .env ps
```

**Expected result:**

- Frontend container is running.
- Backend container is running.
- Database container is running.
- No unexpected container exits or restart loops are present.

**Pass criteria:** All required services are running and their health checks, where configured, report healthy.

### Test 2: Application Health Endpoint

**Objective:** Verify that the application responds to HTTP health checks.

**Procedure:**

```bash
curl -i --max-time 10 http://localhost/health
```

To test externally, run the request from a machine outside EC2:

```bash
curl -i --max-time 10 http://<EC2-PUBLIC-IP>/health
```

Replace `<EC2-PUBLIC-IP>` with the actual public IP address.

**Expected result:** The configured health endpoint returns the expected HTTP response, typically `200 OK`.

**Pass criteria:** The endpoint responds within the configured timeout and returns the expected status code.

*Note:* The health endpoint must be routed correctly through Nginx. If `/health` is implemented only on the backend, ensure the proxy forwards the request to it.

### Test 3: Frontend Accessibility

**Objective:** Confirm that users can access the EpicBook frontend.

**Procedure:**

1. Open the EC2 public URL in a browser.
2. Alternatively, execute:

```bash
curl -I --max-time 10 http://<EC2-PUBLIC-IP>/
```

**Expected result:** The server returns a successful HTTP response, and the frontend loads without a proxy or server error.

**Pass criteria:** The application page is accessible through the configured public endpoint.

### Test 4: Backend and Database Connectivity

**Objective:** Confirm that the backend can connect to MySQL.

**Procedure:**

1. Inspect backend logs:

```bash
docker compose --env-file .env logs --tail=100 backend
```

2. Inspect database logs:

```bash
docker compose --env-file .env logs --tail=100 database
```

3. If the application exposes a database readiness or health endpoint, call it and verify that the database connection succeeds.

**Expected result:**

- The MySQL server starts successfully.
- The backend can connect to the configured database service.
- No recurring authentication, DNS resolution, or connection-refused errors occur.

**Pass criteria:** The backend successfully accesses the database using the configured application credentials.

### Test 5: Container Restart Recovery

**Objective:** Verify that services recover after a container restart.

**Procedure:**

Restart the backend container:

```bash
docker compose --env-file .env restart backend
```

Then inspect the service state and health endpoint:

```bash
docker compose --env-file .env ps
curl -i --max-time 10 http://localhost/health
```

**Expected result:** The backend restarts and resumes serving requests without requiring a full deployment.

**Pass criteria:** The backend returns to a healthy state and the application health endpoint recovers within the expected startup period.

### Test 6: Database Persistence

**Objective:** Confirm that application data persists across container recreation.

**Procedure:**

1. Create or identify a test record in the application.
2. Record a safe identifier for that record.
3. Recreate the database container without removing named volumes:

```bash
docker compose --env-file .env up -d --force-recreate database
```

4. Wait for MySQL to become ready.
5. Verify that the test record still exists.

**Expected result:** Previously stored data remains available after the database container is recreated.

**Pass criteria:** The test record remains present and readable.

**Safety requirement:** Run this test only in a suitable test environment or during an approved maintenance window. Back up important data first. Never use `docker compose down -v` for this test because it can delete the named database volume.

### Test 7: Deployment Image Consistency

**Objective:** Confirm that the EC2 instance runs the intended Docker Hub image versions.

**Procedure:**

On the EC2 instance, run:

```bash
cd ~/theepicbook
docker compose --env-file .env config --images
docker compose --env-file .env ps
```

Inspect the image tags and confirm they match the version published by the Azure DevOps build stage.

**Expected result:** The Compose configuration references the intended Docker Hub repositories and image tags.

**Pass criteria:** The deployed services use the expected images rather than outdated local image names or unintended tags.

### Test 8: Deployment Failure Detection

**Objective:** Verify that deployment failures are detected instead of being reported as successful.

**Procedure:**

In a non-production environment, test a controlled failure, such as temporarily configuring an invalid image tag.

Run the deployment pipeline and observe the pull and deployment stages.

**Expected result:** The pipeline fails when the required image cannot be pulled or the deployment health check does not pass.

**Pass criteria:** The pipeline reports failure and provides enough logs to identify the problem.

Restore the valid image tag after the test.

## 4. Automated Reliability Checks in Azure DevOps

The deployment pipeline should perform the following checks automatically:

1. Validate required configuration and SQL files before copying them to EC2.
2. Validate the Docker Compose configuration.
3. Pull the exact frontend and backend image tags.
4. Start or update the application containers.
5. Check that the required containers are running and healthy where health checks are configured.
6. Request the application health endpoint and fail the pipeline if the expected response is not received.

Example HTTP verification command:

```bash
set -euo pipefail

URL="http://$(vmPublicIp)/health"

HTTP_CODE=$(curl \
  --silent \
  --show-error \
  --output /tmp/epicbook-health-response.txt \
  --write-out "%{http_code}" \
  --max-time 10 \
  "$URL")

if [ "$HTTP_CODE" != "200" ]; then
  echo "ERROR: Health check failed. HTTP status: $HTTP_CODE"
  cat /tmp/epicbook-health-response.txt
  exit 1
fi

echo "SUCCESS: EpicBook health check returned HTTP 200."
```

The pipeline should not declare deployment success solely because Docker Compose accepted the configuration. The health endpoint must also pass.

## 5. Troubleshooting Guide

| Symptom | Investigation | Corrective action |
|---|---|---|
| Container exits immediately | Inspect container logs and exit status | Correct the application configuration or startup command |
| Backend cannot connect to MySQL | Check service name, port, credentials, and database readiness | Correct the connection settings and verify network connectivity |
| Docker Hub pull fails | Check image repository, tag, and registry authentication | Publish the image or correct the image reference and credentials |
| HTTP 502 from Nginx | Inspect Nginx configuration and backend status | Correct the upstream address or restore the backend service |
| Health endpoint times out | Check application logs, routing, and security-group rules | Restore the endpoint and verify network access |
| Data disappears after recreation | Inspect the database volume configuration | Restore the persistent volume configuration and recover data from backup if necessary |
| Pipeline passes but application is unavailable | Review health-check scope and deployment verification | Add external HTTP verification and container readiness checks |

Useful commands:

```bash
# Service status
docker compose --env-file .env ps

# Application logs
docker compose --env-file .env logs --tail=100 backend

# Nginx logs
docker compose --env-file .env logs --tail=100 frontend

# Database logs
docker compose --env-file .env logs --tail=100 database

# Disk usage
df -h

# Docker disk usage
docker system df
```

Use the actual Compose service names if they differ from `backend`, `frontend`, or `database`.

## 6. Test Results Record

Record the outcome of each test during a deployment or release validation.

| Test | Result | Evidence / Notes |
|---|---|---|
| Container availability | Pending | Record `docker compose ps` output |
| Application health endpoint | Pending | Record HTTP status |
| Frontend accessibility | Pending | Record browser or HTTP result |
| Backend-to-database connectivity | Pending | Record application and database logs |
| Container restart recovery | Pending | Record recovery outcome |
| Database persistence | Pending | Record test record verification |
| Deployment image consistency | Pending | Record resolved image references |
| Deployment failure detection | Pending | Record pipeline run outcome |

Use **Pass**, **Fail**, or **Not Run** after executing each test. Do not mark a test as passed until its expected result has been verified.

## 7. Conclusion

These reliability tests provide a repeatable way to evaluate the EpicBook deployment before and after application releases. They cover service availability, HTTP access, database connectivity, restart recovery, data persistence, image consistency, and deployment failure detection.

Running these checks as part of the Azure DevOps CI/CD workflow helps identify deployment problems early and provides evidence that the application is operating as expected.