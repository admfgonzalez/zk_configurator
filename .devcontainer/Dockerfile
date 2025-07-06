# Use an official Node.js runtime as a parent image
FROM node:20

# Set the working directory in the container
WORKDIR /app

# Copy package.json and yarn.lock to leverage Docker cache
COPY package.json yarn.lock ./

# Install dependencies
RUN yarn install

# Copy the rest of the application's source code
COPY . .

# Expose the port the app runs on
EXPOSE 5173

# Default command to run when starting the container
CMD [ "yarn", "dev" ]
