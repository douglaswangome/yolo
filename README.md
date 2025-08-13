# Kubernetes Deployment Plan for IP2 Application

The public URL of the frontend is: `http://143.244.213.243:3000/`

## Overview

This document describes the comprehensive process I have completed in preparing my **IP2 Application** for deployment and running it on a live cluster. The application is composed of a React.js frontend, a Node.js backend, and a MongoDB database. The entire infrastructure is hosted on **Kubernetes** using **DigitalOcean Kubernetes Service (DOKS)**.

The deployment has been successfully carried out, and this README now serves as a comprehensive and finalized guide to the architecture, key design choices, and a detailed record of the debugging measures I applied during the process. The most significant discovery I made was the critical importance of explicitly managing Docker image tags to guarantee that Kubernetes would pull the most recent version of an image, which is a fundamental practice for reliable deployments.

---

## 1. Application Structure

The application is modular and built from three main components that communicate with each other:

1.  **Frontend**

    - A modern React.js single-page application that provides the user interface.
    - The application was containerized using Docker and is now hosted on Docker Hub. The finalized image uses the specific version tag: `douglaswangome/ip2-client:v1.0.0`. This intentional use of a version tag was a direct result of debugging efforts to ensure consistency.
    - A key architectural decision was to inject the backend API URL into the frontend at deployment time. This is achieved securely using a Kubernetes Secret, which prevents sensitive configuration from being hardcoded in the application and allows for easy updates without rebuilding the Docker image.

2.  **Backend**

    - A Node.js REST API that handles business logic and data persistence.
    - It was also containerized and pushed to Docker Hub under the image name: `douglaswangome/ip2-backend:latest`. While the frontend now uses a versioned tag, the backend image is still being managed with the `:latest` tag for simplicity in this specific context.
    - The crucial MongoDB connection string is managed via a Kubernetes Secret, ensuring that sensitive credentials are not exposed in the deployment manifest or container image.

3.  **Database**
    - The MongoDB database is hosted externally to the Kubernetes cluster.
    - This decision was made to leverage a production-grade, managed database service (either a cloud provider's database-as-a-service or a self-hosted instance on a separate machine). This approach simplifies the Kubernetes deployment by removing the complexities of running a stateful database inside containers, which often requires more advanced concepts like Persistent Volumes and StatefulSets.

---

## 2. Why Kubernetes on DigitalOcean?

Moving my application from a **local Docker setup** to **Kubernetes on DOKS** provides several key advantages that were critical for a professional and scalable deployment:

- **Scalability**: Kubernetes provides an elegant solution for scaling. The platform can easily run multiple replicas of both the frontend and backend to handle increased user traffic. I can simply adjust the `replicas` count in the Deployment manifests, and Kubernetes will automatically create new pods. Furthermore, it manages **rolling updates**, gracefully replacing old pods with new ones without any downtime, which is essential for a seamless user experience.
- **Networking**: Kubernetes' built-in networking is powerful. It handles **service discovery** automatically. This means that my frontend pods can reach the backend pods simply by calling the service name `ip2-backend`, which resolves internally to the correct IP address and port. This is a far more robust and flexible approach than relying on hardcoded IP addresses.
- **Managed Infrastructure**: DigitalOcean's managed Kubernetes service takes care of the underlying infrastructure. This means I don't have to worry about managing the control plane components like the API server, scheduler, and etcd database. This greatly reduces the operational overhead and lets me focus on the application itself.
- **Public URL**: The `LoadBalancer` type service is a killer feature. DigitalOcean's DOKS automatically provisions and configures a public load balancer with a stable IP address for the frontend service. This provides a single, reliable public URL for users to access the application, distributing traffic evenly across all running frontend pods.

---

## 3. Kubernetes Manifests

The entire deployment is managed by two sets of Kubernetes manifest files, which have been successfully applied to the DigitalOcean cluster.

### 3.1 Frontend (`frontend-deployment.yaml`)

This manifest defines both the `Deployment` and `Service` for the React frontend, orchestrating how the application runs and is exposed to the public.

- **Deployment**: This resource guarantees that a single replica of my frontend container is always running. The manifest explicitly uses the image `douglaswangome/ip2-client:v1.0.0`. It declares that the container listens on port `3000`. Crucially, it fetches the `REACT_APP_BACKEND_URL` value from a Kubernetes Secret named `frontend-config`, ensuring that the frontend knows how to communicate with the internal backend service without exposing sensitive information.
- **Service**: A `LoadBalancer` type service is specified here to expose the frontend to the public internet. This type of service automatically provisions a cloud provider-specific load balancer (in this case, from DigitalOcean) and assigns a public IP address to it. The service directs all incoming traffic on port `3000` to the frontend pods, providing a stable entry point to the application.

**Frontend Manifest:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ip2-frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ip2-frontend
  template:
    metadata:
      labels:
        app: ip2-frontend
    spec:
      containers:
        - name: frontend
          image: douglaswangome/ip2-client:v1.0.0
          ports:
            - containerPort: 3000
          env:
            - name: REACT_APP_BACKEND_URL
              valueFrom:
                secretKeyRef:
                  name: frontend-config
                  key: REACT_APP_BACKEND_URL
---
apiVersion: v1
kind: Service
metadata:
  name: ip2-frontend
spec:
  selector:
    app: ip2-frontend
  ports:
    - port: 3000
      targetPort: 3000
  type: LoadBalancer
```

### 3.2 Backend (`backend-deployment.yaml`)

This manifest defines the `Deployment` and `Service` for the Node.js backend. The key difference here is that this service is intended for internal communication only.

- **Deployment**: This deployment also creates a single replica of the backend container. It uses the `douglaswangome/ip2-backend:latest` image and exposes port `5000`. The connection to the external MongoDB is configured by pulling the `MONGODB_URI` from a Kubernetes Secret, named `mongodb-secret`, reinforcing the principle of not storing credentials in plain text.
- **Service**: A `ClusterIP` service is used here. This service type exposes the backend on an internal IP address only, making it inaccessible from the public internet. This is a critical security measure. The frontend pods can still communicate with the backend using the service name `ip2-backend` because of Kubernetes' internal DNS system, but the outside world cannot.

**Backend Manifest:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ip2-backend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ip2-backend
  template:
    metadata:
      labels:
        app: ip2-backend
    spec:
      containers:
        - name: backend
          image: douglaswangome/ip2-backend:latest
          ports:
            - containerPort: 5000
          env:
            - name: MONGODB_URI
              valueFrom:
                secretKeyRef:
                  name: mongodb-secret
                  key: MONGODB_URI

---
apiVersion: v1
kind: Service
metadata:
  name: ip2-backend
spec:
  selector:
    app: ip2-backend
  ports:
    - port: 5000
      targetPort: 5000
  type: LoadBalancer
```

---

## 4. Debugging and Key Learnings

During the deployment process, a significant challenge arose: Kubernetes was not pulling the new Docker image even after I had pushed it to the registry. The pods continued to run with an older version of the application. This issue was particularly tricky because I was using the `:latest` tag, which is often mistakenly assumed to force a pull.

### **The Problem: Kubernetes Image Caching**

My initial assumption was that using the `:latest` tag would guarantee that Kubernetes would pull the most up-to-date image from Docker Hub. However, I learned that Kubernetes, by default, caches container images on each node. The `kubelet` process on each node only checks for a new image if the image tag in the deployment manifest changes. Simply pushing a new image to the registry with the same `:latest` tag does not change the tag in the manifest. Therefore, Kubernetes saw no change, and the `kubectl apply -f` command was essentially a no-op, leading to the pods continuing to use the stale, locally cached image.

### **The Solution: Using Versioned Tags and Explicit Commands**

The definitive solution was to **stop using the `:latest` tag** and instead use a unique, immutable version tag for each new image. This is a fundamental best practice for a reason: it ensures every deployment is intentional and reproducible.

1.  **I built and pushed a new image with a specific version tag:**
    `docker build -t douglaswangome/ip2-client:v1.0.0 ./client/`\
    `docker push douglaswangome/ip2-client:v1.0.0`\
    This provides a single, traceable version of the application that can be deployed at any time.

2.  **Then I updated the deployment manifest (`frontend-deployment.yaml`) to reference this specific tag.**
    - This change to the manifest's image tag (`ip2-client:v1.0.0`) is what signals to Kubernetes that a new version of the application needs to be deployed.
    - The command `kubectl apply -f frontend-deployment.yaml` then successfully triggered a rolling update, and the new pods were created using the new image.

### **The Solution to the `LoadBalancer` Problem in the Backend**

The initial backend manifest mistakenly used `LoadBalancer` for its service type. A `LoadBalancer` is designed to expose a service to the public internet, which is completely unnecessary and a potential security risk for the backend API, which should only be accessed by the frontend. I corrected this by changing the backend service's type to `ClusterIP`, which makes it accessible only from within the Kubernetes cluster. The frontend is able to communicate with the backend using its service name `ip2-backend`, which is a much more secure and reliable pattern.

### **Good Practices**

My final deployment plan incorporates a number of best practices that are crucial for a robust and maintainable system:

- **Using Unique Image Tags**: This ensures that every update is deliberate and traceable, preventing unexpected behavior from image caching and providing a simple rollback mechanism.
- **Separating Services**: The frontend and backend have distinct Service types—`LoadBalancer` for the public-facing frontend and `ClusterIP` for the internal backend—to optimize both accessibility and security.
- **Using Secrets for Configuration**: Sensitive information like the MongoDB URI and the backend API URL are stored in Kubernetes Secrets. This is a critical security measure that prevents credentials from being exposed in public repositories or container images.
