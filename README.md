# Kanban Dashboard – Docker & Jenkins CI/CD

A React + TypeScript Kanban Dashboard deployed using Docker and automated with Jenkins CI/CD on AWS EC2.

## Project Overview

This project demonstrates:

- React + TypeScript application
- Docker containerization
- Docker Hub image registry
- Jenkins CI/CD pipeline
- AWS EC2 deployment
- Docker health checks
- Container resource limits
- Non-root Docker container
- Deployment rollback on failure
- Blue-Green deployment
- Zero/minimal downtime deployment
- Nginx reverse proxy
- GitHub webhook integration

## Technologies Used

- React
- TypeScript
- Vite
- Docker
- Docker Hub
- Jenkins
- AWS EC2
- GitHub
- Nginx

## Application

The application is packaged as a Docker image and deployed on an AWS EC2 instance.

### Docker Image

`daddykavin/kanban-dashboard`

Images are versioned using Jenkins build numbers.

Example:

`daddykavin/kanban-dashboard:build-27`

## Docker Containerization

The application is containerized using a Dockerfile.

The Docker configuration includes:

- Multi-stage build
- Production-ready runtime image
- Non-root container user
- Docker health check
- CPU and memory resource limits

## Jenkins CI/CD Pipeline

The Jenkins pipeline automates the complete deployment process.

Pipeline stages:

1. Checkout source code from GitHub
2. Build Docker image
3. Tag image using Jenkins build number
4. Login to Docker Hub
5. Push image to Docker Hub
6. Deploy application to AWS EC2
7. Perform container health check
8. Verify application response
9. Switch traffic using Nginx
10. Stop the old container after successful verification
11. Perform final health check
12. Roll back deployment if deployment fails

## Blue-Green Deployment

The project uses a Blue-Green deployment strategy to minimize application downtime.

A new container is started alongside the currently active container.

The deployment process is:

1. Identify the currently active container.
2. Start the new container on the inactive port.
3. Connect the new container to the Docker network.
4. Wait for the Docker health check to become healthy.
5. Verify the new container directly.
6. Update the Nginx upstream configuration.
7. Test the Nginx configuration.
8. Reload Nginx.
9. Verify the application through the Nginx proxy.
10. Stop and remove the old container only after successful verification.

This ensures that the old container remains available until the new container has passed its health checks and traffic verification.

## Rollback

The Jenkins pipeline includes automatic rollback on deployment failure.

If a deployment fails:

- The new container is removed.
- The previous Nginx configuration is restored.
- Nginx is reloaded.
- The previous application container remains available.

## Deployment Verification

Successful Jenkins deployment verification includes:

- New container becomes healthy.
- New container responds successfully.
- Nginx configuration test passes.
- Nginx reload succeeds.
- Application responds through the proxy.
- Old container is stopped only after traffic verification.
- Final health check passes.

## Jenkins Build

The successful Blue-Green deployment was completed in:

**Jenkins Build #27**

Docker image:

`daddykavin/kanban-dashboard:build-27`

Deployment result:

**SUCCESS**

## Repository

GitHub Repository:

`https://github.com/KAVIN1037/kanban-dashboard`

## Conclusion

This project demonstrates an automated Docker-based CI/CD workflow using Jenkins, Docker Hub, AWS EC2 and Nginx, including health checks, rollback handling and Blue-Green deployment with minimal downtime.
