# IP2
For consistency I used, <code>node:16-alpine</code> across both Dockerfiles.

## Client
The <code>package.json</code> for the Client indicates the use of Node.js v13. This version is unstable and no longer actively supported, which could pose security risks for the Docker image in the future. As stated above, I opted for <code>node:16-alpine</code> because it is a stable version that still receives regular updates and security patches, thus mitigating future security concerns.\
\
Below is the Dockerfile for the Client with detailed comments:

```Dockerfile
# Stage 1: This stage builds the React application before serving it in Stage 2.

# Specifies the base image upon which the new image will be built.
# "AS builder" names this stage, allowing it to be referenced in Stage 2.
# Initially, I did not use "alpine" in Stage 1 to maximize compatibility. However,
# downloading the larger node:16 image proved problematic with my unstable internet connection.
# I switched to alpine to successfully complete Stage 1.
FROM node:16-alpine AS builder

# Sets the working directory for subsequent instructions.
WORKDIR /app

# Copies new files or directories from the host machine (source) to inside the image (destination).
# The asterisk (*) is a wildcard, and the .json extension specifies that only JSON files
# matching the 'package' prefix should be copied.
COPY package*.json ./

# Installs all dependencies in the image.
# I chose "npm ci" over "npm i" to ensure that the exact versions specified in package-lock.json are installed,
# promoting consistent builds.
# RUN npm i
RUN npm ci

# Copies the rest of the application files from the host machine to the image.
COPY . .

# This command builds the React application.
RUN npm run build

# Stage 2: This is where the application is "served."

# Specifies the base image for this stage.
# In Stage 2, the focus is on reducing the final image size rather than broad compatibility,
# which explains the continued use of "alpine."
FROM node:16-alpine

# Sets the working directory for subsequent instructions.
WORKDIR /app

# Installs the 'serve' package globally.
RUN npm install -g serve

# Copies the built React application from the 'builder' stage (Stage 1).
# "--from=builder" references the named Stage 1.
COPY --from=builder /app/build ./build

# Exposes port 3000, making it accessible from outside the container.
EXPOSE 3000

# Defines the command to run when the container starts, serving the React application
# from the 'build' directory on port 3000.
CMD [ "serve", "-s", "build", "-l", "3000" ]
```

During the creation of the above Dockerfile, I encountered one major challenge:

- Internet Connectivity: My unstable internet connection was a significant issue. I initially planned to use node:16 to avoid potential compatibility problems. However, node:16 is considerably larger than its alpine counterpart. To successfully build and run the container locally, I switched to the alpine variant. I did not face any compatibility issues, but this is something to monitor in the future.

To run the client image, I use the following command (executed from the client directory):
```bash
docker run -p 3000:3000 #image_id
```

## Backend
Below is the Dockerfile for the client with comments to explain everything in great detail:
```Dockerfile
# No multi-stage build is required for this Dockerfile as 'serve' is not needed.

# Specifies the base image upon which the new image will be built.
FROM node:16-alpine

# Sets the working directory for subsequent instructions.
WORKDIR /app

# Copies new files or directories from the host machine (source) to inside the image (destination).
# The asterisk (*) is a wildcard, and the .json extension specifies that only JSON files
# matching the 'package' prefix should be copied.
COPY package*.json ./

# Installs all dependencies in the image.
# I chose "npm ci" over "npm i" to ensure that the exact versions specified in package-lock.json are installed,
# promoting consistent builds.
# RUN npm i
RUN npm ci

# Copies the rest of the application files from the host machine to the image.
COPY . .

# Exposes port 5000, making it accessible from outside the container.
EXPOSE 5000

# Defines the command to run when the container starts, launching the Node.js server.
CMD [ "node", "server.js" ]
```

During the creation of the above Dockerfile, I encountered one major challenge:

- The .env file: It is customary to exclude the .env file when copying the application directory. However, my environment variable (specifically the MONGODB_URI value) contained double quotes within the .env file. This led to errors when attempting to run the container with the --env-file .env flag, as Docker misinterpreted the environment variable, preventing the backend from starting correctly.

To run the backend image, I use the following command (executed from the backend directory):
```bash
docker run -p 5000:5000 --env-file .env #image_id
```