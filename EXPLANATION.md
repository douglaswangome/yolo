# IP2
For consitency I used <code>node:16-alpine</code> across both Dockerfiles.

## Client
The package.json indicates that the client uses node v13. This version of node is unstable and is not supported. This might pose security issues for the docker image in the future. As stated above, I used the node v16 as it is stable and still reeives regular updates and security patches. This will not pose a security risk for us in the future.\
\
Below is the Dockerfile for the client with comments to explain everything in great detail:

```Dockerfile
# Stage 1: This stage is used to build the react application before serving it on Stage 2

# Specifies the base image upon which the new image will be built upon
# "AS builder" this will be referenced in Stage 2
# Initially I had not used "alpine" on Stage 1 to maximize compatibility but that had issues with the time it took to download node:16 with my unstable internet
# I switched to alpine so as to be able to complete Stage 1 succesfully
FROM node:16-alpine AS builder

# Sets the working directory for any instructions that follow this
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

# This command builds the react application
RUN npm run build

# Stage 2: This is where the application is "served"

# Specifies the base image upon which the new image will be built upon
# This Stage 2 build, I am not focusing on compatibility, I want to reduce the size of the image
# This explains the use of "alpine"
FROM node:16-alpine

# Sets the working directory for any instructions that follow this
WORKDIR /app

# Install serve globally
RUN npm install -g serve

# Copy the built React app from the build stage in Stage 1
# From the line FROM node:16 as "builder"
# --from=builder references Stage 1 using the name above
COPY --from=builder /app/build ./build

# Expose the port 
EXPOSE 3000

# Command to run serve on the build directory
CMD [ "serve", "-s", "build", "-l", "3000" ]
```

While building the Dockerfile above I had one major error:\
- <b>My Internet</b> - This was a major issue as I was planning to use <code>node:16</code> so as to avoid potential compatibility issues. However, <code>node:16</code> is bigger in size in comparison to its alpine sister. So as to be able to run the container locally after building the image, I switched to alpine. I faced no comptability issues but that may be something to watch out for.

## Server
