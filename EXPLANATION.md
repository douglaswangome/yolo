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

While building the Dockerfile above I had one major error:
- <b>My Internet</b> - This was a major issue as I was planning to use <code>node:16</code> so as to avoid potential compatibility issues. However, <code>node:16</code> is bigger in size in comparison to its alpine sister. So as to be able to run the container locally after building the image, I switched to alpine. I faced no comptability issues but that may be something to watch out for.

To run the image, I use (while in the client directory):
```bash
docker run -p 3000:3000 #image_id
```

## Backend
Below is the Dockerfile for the client with comments to explain everything in great detail:
```Dockerfile
# No stages needed in this Dockerfile as "serve" is not require
# Specifies the base image upon which the new image will be built upon
FROM node:16-alpine

# Sets the working directory for any instructions that come after this
WORKDIR /app

# Copies new files or directories from the host machine(source) to inside the image(destination)
# The asterisk(*) is a wildcard to indicate any files that has the word package as a prefix
# The json(.json) is to specify that only json files that match the wildcard above.
COPY package*.json ./

# This installs all the dependencies in the image
# I would use "npm i" but "npm ci" ensures that the correct versions are installed
# RUN npm i
RUN npm ci

# Copies new files or directories from the host machine(source) to inside the image(destination)
COPY . .

# Expose the port 
EXPOSE 5000

# Command to run the backend server
CMD [ "node", "server.js" ]

```
While building the Dockerfile above I had one major error:
- <b>The .env file</b> - It is customary to ignore the .env file while copying the directory and its contents. My env variable had double quotes and this led to errors while running the command below with the <code>--env-file .env</code> flag. This led to the env being misinterpreted and the backend not starting up.

To run the image, I use (while in the backend directory):
```bash
docker run -p 5000:5000 --env-file .env #image_id
```