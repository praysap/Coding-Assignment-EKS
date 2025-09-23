# Use official Node.js 18 image
FROM node:18-alpine

# Create and set working directory
WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm install

# Copy all application files
COPY . .


# Expose port 3000
EXPOSE 3000

# Start the application
CMD ["node", "index.js"]