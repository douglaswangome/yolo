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

# Command to run serve on the build directory
CMD [ "node", "server.js" ]
