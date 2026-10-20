# EpicBook CI/CD Pipeline Documentation Using Azure DevOps

## 1. Overview

This document explains how to configure and execute an automated Continuous Integration and Continuous Deployment (CI/CD) pipeline for the EpicBook application using Azure DevOps.

The pipeline automates building Docker images for the frontend and backend, pushing the images to Docker Hub, and deploying the containerized application to an AWS EC2 virtual machine using Docker Compose.

The objective is to simplify application deployment, reduce manual intervention, and make future application updates easier to release.

## 2. Architecture and Tools

The deployment process uses the following technologies:

- **GitHub:** Source-code version control and development repository.
- **Azure Repos:** Hosts the repository imported from GitHub for Azure DevOps pipeline integration.
- **Azure Pipelines:** Automates the build, image publishing, deployment, and verification stages.
- **Docker:** Packages the frontend and backend into container images.
- **Docker Hub:** Stores and distributes the published Docker images.
- **AWS EC2:** Hosts the production application containers.
- **Docker Compose:** Coordinates the frontend, backend, and MySQL database containers.
- **Nginx:** Acts as the reverse proxy for incoming application requests.
- **Azure DevOps Variable Groups and Secure Files:** Store deployment variables, database credentials, Docker Hub credentials, and the VM SSH private key.

## 3. CI/CD Deployment Workflow

The deployment follows this sequence:

1. Develop or update the application code.
2. Commit and push changes to GitHub.
3. Import or synchronize the repository with Azure Repos.
4. Configure Azure DevOps pipeline variables and secure files.
5. Create and configure the Azure Pipelines YAML file.
6. Run the pipeline to build and publish Docker images.
7. Transfer deployment configuration files to the EC2 instance.
8. Pull the published images and restart the application containers.
9. Verify that the application responds successfully.

## 4. Configuration and Implementation

### Step 1: Prepare Docker Hub

1. Sign in to Docker Hub.
2. Create or confirm the repositories for the EpicBook frontend and backend.
3. Ensure the repositories match the image names used by the pipeline, for example:
   - `epicbook-frontend`
   - `epicbook-backend`
4. Generate a Docker Hub access token with the permissions required to push and pull images.
5. Store the token securely for use by Azure DevOps.

The pipeline builds the frontend and backend images and pushes them to Docker Hub with version tags. This allows the deployment server to pull the exact images produced by the build stage.

### Step 2: Create an Azure DevOps Variable Group

1. Open the Azure DevOps organization and project.
2. Navigate to **Pipelines → Library**.
3. Select **Variable groups** and create a group named `Epicbook-docker-secrets`.
4. Add the required variables.

| Variable | Purpose | Secret? |
|---|---|---|
| `dockerHubUsername` | Docker Hub account username | No |
| `dockerHubToken` | Docker Hub access token | Yes |
| `dbUserSecret` | MySQL application username | Preferably yes |
| `dbPasswordSecret` | MySQL application password | Yes |
| `dbRootPasswordSecret` | MySQL root password | Yes |
| `dbNameSecret` | Application database name | Preferably yes |
| `vmPublicIp` | Public IP address of the EC2 instance | No |

5. Mark sensitive variables as secret.
6. Authorize the pipeline to access the variable group.

The pipeline references the variable group to obtain the values required for image publishing, database configuration, and deployment.

**Security note:** Never commit database passwords, Docker Hub tokens, or other secrets to GitHub or Azure Repos.

### Step 3: Upload the EC2 SSH Private Key

Azure DevOps needs secure SSH access to the EC2 instance to transfer configuration files and execute deployment commands.

1. Navigate to **Pipelines → Library → Secure files**.
2. Upload the SSH private key associated with the EC2 instance.
3. Name the secure file `epicbook-docker-vm-key`.
4. Grant the deployment pipeline permission to access the file.

The pipeline downloads the key at runtime and applies restrictive file permissions before using it to connect to the EC2 instance.

The corresponding EC2 security group must allow SSH access from the deployment agent's network. Avoid opening SSH to all internet addresses where possible.

### Step 4: Create the Azure Pipelines YAML File

Create a pipeline file named `azure-pipelines-docker.yml` in the project repository.

The YAML file defines the automated build and deployment process. It includes the following main stages:

**A. Build and publish Docker images**

- Check out the application source code.
- Authenticate to Docker Hub using the configured credentials.
- Build the frontend and backend images.
- Tag the images using the pipeline's image-tagging strategy.
- Push the images to Docker Hub.

**B. Deploy to AWS EC2**

- Download the SSH private key from Azure DevOps Secure Files.
- Validate the Docker Compose file, Nginx configuration, and SQL initialization files.
- Transfer the required deployment files to the EC2 instance.
- Create or update the protected `.env` file containing the deployment variables.
- Validate the resolved Docker Compose configuration.
- Pull the exact frontend and backend image tags from Docker Hub.
- Restart the application using Docker Compose.

**C. Verify the deployment**

- Check the container status.
- Request the application's health endpoint through Nginx.
- Fail the pipeline if the deployment does not pass the configured health check.

For production deployment, the Compose file should reference the published images using the configured Docker Hub username and image tag, rather than relying on images built locally on the EC2 instance.

### Step 5: Commit and Push the Pipeline Changes to GitHub

After creating or updating the pipeline YAML file:

1. Validate the YAML structure and review the deployment commands.
2. Confirm that no secrets are included in the source files.
3. Commit the changes to the appropriate Git branch.
4. Push the changes to GitHub.

Example commands:

```bash
git add azure-pipelines-docker.yml docker-compose.yml
git commit -m "Add Azure DevOps Docker deployment pipeline"
git push origin Epicbook-Container-Production
```

Replace the branch name if your project uses a different deployment branch.

### Step 6: Import the GitHub Repository into Azure Repos

1. Open the Azure DevOps project.
2. Navigate to **Repos → Files**.
3. Select the option to import a repository.
4. Enter the GitHub repository URL and any required authentication details.
5. Import the project into Azure Repos.
6. Confirm that the application source code, Docker Compose file, and pipeline YAML file are present.

If you intend to keep making changes in GitHub, configure an appropriate synchronization process or ensure changes are mirrored into Azure Repos. An import does not necessarily keep the repositories synchronized automatically.

### Step 7: Create the Azure Pipeline

1. Navigate to **Pipelines → Pipelines**.
2. Select **New pipeline**.
3. Choose **Azure Repos Git** as the source.
4. Select the imported EpicBook repository.
5. Choose the existing Azure Pipelines YAML file.
6. Select `azure-pipelines-docker.yml`.
7. Review the pipeline configuration.
8. Save and run the pipeline.

Confirm that the pipeline has access to the variable group and Secure File. Also verify that the selected branch contains the latest pipeline configuration.

If the YAML defines a branch trigger, make sure your changes are pushed to the branch configured in that trigger.

### Step 8: Run and Monitor the Pipeline

When the pipeline starts, monitor the job logs for each stage.

The expected execution sequence is:

1. **Build:** Frontend and backend Docker images are built.
2. **Publish:** The images are pushed successfully to Docker Hub.
3. **Transfer:** Docker Compose, Nginx, and SQL files are copied to EC2.
4. **Deploy:** The EC2 host pulls the intended images and restarts the containers.
5. **Verify:** The pipeline checks the application health endpoint.

A successful pipeline run indicates that all configured stages and checks completed successfully. It is still good practice to confirm that the application works correctly from a browser.

## 5. Deployment Verification

After the pipeline finishes, verify the deployment using the following checks.

### Check container status

Connect to the EC2 instance and run:

```bash
cd ~/theepicbook
docker compose --env-file .env ps
```

Confirm that the expected frontend, backend, and database services are running.

### Check application health

Open the application's public URL in a browser or use:

```bash
curl -i http://<EC2-PUBLIC-IP>/health
```

Replace `<EC2-PUBLIC-IP>` with the configured EC2 public IP address.

The pipeline should receive the HTTP status code expected by its health-check configuration, typically `200 OK`.

### Review deployment logs

If the deployment fails, inspect the container logs:

```bash
docker compose --env-file .env logs --tail=100 backend
docker compose --env-file .env logs --tail=100 frontend
docker compose --env-file .env logs --tail=100 database
```

Use the service names defined in the actual Compose file. Do not share logs containing passwords, tokens, or other sensitive values.

## 6. Common Issues and Troubleshooting

| Issue | Possible cause | Recommended action |
|---|---|---|
| Docker Hub image pull denied | Incorrect repository name, tag, or authentication | Check the image reference and Docker Hub permissions |
| Compose pulls `epicbook-backend` instead of the Docker Hub image | Outdated Compose file or missing image variables | Check `docker compose config --images` on EC2 |
| SSH connection fails | Incorrect key, permissions, or security-group rules | Validate the secure file and SSH access |
| SQL initialization mount fails | SQL file missing or a directory exists at the expected file path | Validate the local SQL files and their destination paths before deployment |
| Database initialization scripts do not run | MySQL data volume was already initialized | Check database state and initialization logs; back up data before considering any reset |
| Health check fails | Containers are unhealthy, Nginx routing is incorrect, or the endpoint is unavailable | Inspect container status, application logs, and Nginx configuration |
| Pipeline cannot access secrets | Variable group or Secure File permissions are missing | Authorize the pipeline to access the required resources |

## 7. Security and Operational Best Practices

- Store credentials in Azure DevOps secret variables or an approved secrets manager.
- Use Docker Hub access tokens instead of account passwords.
- Restrict SSH access to trusted sources.
- Use versioned Docker image tags so deployments are reproducible.
- Validate the Compose configuration before restarting production services.
- Keep the MySQL data volume persistent and back up important data.
- Avoid printing secrets in pipeline logs.
- Configure health checks and retain deployment logs for troubleshooting.
- Test pipeline changes in a non-production environment when possible.

## 8. Conclusion

The EpicBook CI/CD pipeline automates the process of building, publishing, and deploying the frontend and backend containers. Azure DevOps orchestrates the workflow, Docker Hub stores the versioned images, and AWS EC2 hosts the application through Docker Compose.

This approach reduces repetitive manual deployment tasks, improves consistency between releases, and provides a repeatable process for delivering future application updates. The deployment is considered successful when the pipeline stages complete and the application's health endpoint responds as expected.