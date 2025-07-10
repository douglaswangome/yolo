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