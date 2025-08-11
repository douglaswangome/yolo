# Kubernetes Deployment Plan for IP2 Application

The public URL of the frontend is: http://206.189.252.191:3000/

## Overview

This document describes the work I have completed so far in preparing my **IP2 Application** (React.js frontend + Node.js backend + MongoDB database) for deployment on **Kubernetes** using **DigitalOcean Kubernetes Service (DOKS)**.

**Important:**  
As of the time of writing, **everything has been deployed to DigitalOcean yet**.  
This README documents the _preparation phase_, including:

- Docker image creation
- Pushing images to Docker Hub
- Kubernetes manifests for deployment
- CLI setup instructions for DigitalOcean

These steps form the **complete deployment plan** which will be executed in the next phase.

---

## 1. Application Structure

The application consists of:

1. **Frontend**

   - React.js single-page application
   - Dockerized and pushed to Docker Hub as:  
     `douglaswangome/ip2-client:latest`
   - No `.env` file at build time, meaning API URL is currently hardcoded or defaults to localhost (to be addressed before deployment).

2. **Backend**

   - Node.js REST API
   - Connects to MongoDB (hosted externally — _not_ dockerized for production)
   - Dockerized and pushed to Docker Hub as:  
     `douglaswangome/ip2-backend:latest`

3. **Database**
   - MongoDB hosted externally
   - Not running in Kubernetes to avoid unnecessary complexity and ensure a production-grade database.

---

## 2. Why Kubernetes on DigitalOcean?

Moving from a **local Docker setup** to **Kubernetes** provides:

- **Scalability**: Run multiple replicas of frontend/backend as needed.
- **Networking**: Built-in service discovery (`ip2-backend` can be reached by `ip2-frontend` internally).
- **Managed Infrastructure**: DigitalOcean handles Kubernetes master nodes and provides a LoadBalancer for public access.
- **Public URL**: A `LoadBalancer` type service for the frontend will give a publicly accessible IP.

---

## 3. Kubernetes Manifests (Prepared, Not Yet Applied)

I have prepared two KuberneteRes deployment + service configurations:

### Backend (`backend-deployment.yaml`)

- Deployment for `douglaswangome/ip2-backend:latest`
- `ClusterIP` service for internal communication
- Environment variable for `MONGO_URI`

### Frontend (`frontend-deployment.yaml`)

- Deployment for `douglaswangome/ip2-client:latest`
- `LoadBalancer` service for public access
- API URL intended to point to `http://ip2-backend:5000` inside the cluster

---
