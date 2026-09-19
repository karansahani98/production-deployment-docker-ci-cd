# Production Dockerfile Example
#
# This is a generalized example based on the deployment process.
# Do not place production secrets inside this file.

FROM node:22-bookworm-slim

WORKDIR /app

# Copy package files first.
# This allows Docker to cache dependency installation
# when application source code changes.
COPY package*.json ./

# Install production dependencies only.
RUN npm ci --omit=dev

# Copy application source.
COPY . .

# Application port.
EXPOSE 3000

# Start the production application.
CMD ["npm", "start"]