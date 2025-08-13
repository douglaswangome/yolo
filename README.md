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

# IP3: Automating Deployment with Ansible and Vagrant

IP3, focuses on automating the deployment of the IP2 application using **Ansible and Vagrant**. This approach significantly streamlines the process, ensuring consistent and reproducible deployments across different environments, from local development to production. I used Vagrant to provision a local virtual machine (VM) and Ansible to automate the entire deployment workflow within that VM. This section details the role of each configuration file and explains the deployment architecture.

---

## 1\. Local Development Environment with Vagrant

To create a consistent and isolated environment for testing the Ansible playbook, I used **Vagrant**. The `Vagrantfile` is the core of this setup, defining the VM's specifications and provisioning steps.

### **The `Vagrantfile`**

```ruby
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.box = "geerlingguy/ubuntu2004"
  config.vm.box_version = "1.0.4"

  config.vm.network "forwarded_port", guest: 3000, host: 3000, auto_correct: false
  config.vm.network "forwarded_port", guest: 5000, host: 5000, auto_correct: false

  config.vm.provision "ansible" do |ansible|
    ansible.playbook = "playbook.yaml"
    ansible.verbose = "v"
    ansible.ask_vault_pass = true
  end
end
```

The `Vagrantfile` is configured to create a VM based on the `geerlingguy/ubuntu2004` box, version `1.0.4`. This ensures that everyone working on the project starts with the same clean Ubuntu 20.04 environment. [It also sets up port forwarding, directing traffic from the host machine's port `3000` to the guest VM's port `3000` (for the frontend) and from the host's port `5000` to the guest's port `5000` (for the backend). This allows me to access the application running inside the VM directly from my host's browser. The most critical part of this file is the `ansible.provision` block, where I tell Vagrant to use Ansible to provision the VM by executing the `playbook.yaml` file. This ensures that as soon as the VM is up and running, Ansible automatically takes over to install dependencies, clone the repository, and deploy the entire application.

---

## 2\. Automating the Deployment with Ansible

Ansible is an automation tool that simplifies complex tasks like software installation and application deployment. I used it to define a repeatable and reliable deployment process.

### **The `ansible.cfg` file**

```cfg
[defaults]
inventory = hosts
remote_user = vagrant
private_key_file = .vagrant/machines/default/virtualbox/private_key
roles_path = ./ansible/roles
host_key_checking = False
```

This file contains the default configurations for the Ansible playbook. It specifies the inventory file as `hosts`, which tells Ansible where to find the target machines. The `remote_user` is set to `vagrant`, which is the default user for the VM, and the `private_key_file` is specified to allow Ansible to authenticate without a password. This configuration is essential for establishing a secure and automated connection to the VM.

### **The `hosts` file**

```ini
[all]
ansible_host=127.0.0.1 port=2222
```

The `hosts` file serves as the inventory for Ansible. In this simple setup, it defines a single target machine group called `all`. The `ansible_host` is set to `127.0.0.1` and the `port` to `2222`, which is how Vagrant forwards the SSH connection from the host machine to the guest VM.

### **The `playbook.yaml` file**

```yaml
- name: Deploy YOLO on VM using Ansible
  hosts: all
  become: yes
  vars_files:
    - ansible/secrets.yml

  roles:
    - install_docker
    - install_git
    - clone_repo
    - network_deployment
    - backend_deployment
    - frontend_deployment
```

This is the main orchestration file for Ansible. It defines a series of roles that are executed in a specific order to deploy the IP2 application. The playbook is designed to be idempotent, meaning it can be run multiple times without causing unintended side effects. The roles include:

- `install_docker`: Installs Docker on the VM.
- `install_git`: Installs Git to clone the application repository.
- `clone_repo`: Clones the application's source code from a remote repository.
- `network_deployment`: Configures the network for the application.
- `backend_deployment`: Deploys the Node.js backend.
- `frontend_deployment`: Deploys the React.js frontend.

This structured approach makes the deployment process modular, easy to understand, and simple to debug.

**Debugging Note:** I initially struggled with the `secrets.yml` file, where I mistakenly saved sensitive information as a `.env` file instead of the correct YAML format required by Ansible. This caused the playbook to fail, and I had to debug and correct the file format to ensure the secrets were correctly loaded. \
**While running `vagrant provision`, you will be asked for a Vault password which is 12345.\
This should be used while running the app, is a placeholder for sensitive information.**

# What I've Learnt

During the deployment process, I encountered several key challenges that taught me crucial lessons about both local development and live deployments on Kubernetes. These debugging experiences were fundamental to creating a robust and professional deployment plan.

### **Kubernetes Image Caching**

A significant issue I faced was Kubernetes not pulling the new Docker image after a successful push to the registry. The pods would continue to run with a stale, older version of the application. My initial assumption was that using the `:latest` tag would force a new pull, but I discovered that Kubernetes, by default, caches images on each node. The `kubelet` only checks for a new image if the image tag in the deployment manifest is changed. Simply pushing a new image with the same `:latest` tag does not trigger an update.

The definitive solution was to adopt a best practice of using a unique, immutable version tag for each new image (e.g., `douglaswangome/ip2-client:v1.0.0`). By updating the deployment manifest to reference this specific tag, I could guarantee that Kubernetes would pull the correct version and trigger a seamless rolling update. This ensures every deployment is intentional, traceable, and reproducible.

### **Network Service Types and Security**

Another key learning came from an error in my initial Kubernetes manifest for the backend. I had mistakenly used a `LoadBalancer` service type for the backend, which is designed to expose a service to the public internet. This was unnecessary and a significant security risk, as the backend API should only be accessible from within the cluster.

I corrected this by changing the backend service's type to `ClusterIP`. This made the backend accessible only to other services within the cluster, such as the frontend. The frontend can still communicate with the backend using its internal service name (`ip2-backend`), a much more secure and reliable pattern than using hardcoded public IP addresses. I learned that carefully selecting the appropriate service type is critical for balancing accessibility with security in a Kubernetes environment.

### **Ansible Secrets Management**

Finally, a specific debugging experience with my Ansible setup taught me about the importance of correct file formatting. I had trouble with my secrets management because I saved sensitive information as a `.env` file instead of the required `.yml` format. This simple mistake caused the playbook to fail and highlighted the necessity of paying close attention to file types and syntax when working with configuration management tools.
