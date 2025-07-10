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