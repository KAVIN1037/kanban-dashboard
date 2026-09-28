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

Images are versioned using Jenkins build numbers:

```text
daddykavin/kanban-dashboard:build-<BUILD_NUMBER>
eof

