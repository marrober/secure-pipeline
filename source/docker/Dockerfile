FROM quay.io/marrober/rhel8-nodejs-14:latest

USER root

# Create app directory
RUN mkdir -p /usr/src/app
WORKDIR /usr/src/app

COPY . .

RUN ls -al

# Install app dependencies
RUN npm install

USER 1001

# Expose port 8080
EXPOSE 8080

# Run container
CMD ["npm", "start"]
