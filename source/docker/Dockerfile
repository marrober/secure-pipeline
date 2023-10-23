FROM quay.io/marrober/rhel8-nodejs-14

# Create app directory
RUN mkdir -p /usr/src/app
WORKDIR /usr/src/app

COPY . .

RUN ls -al

# Install app dependencies
RUN npm install

# Expose port 8080
EXPOSE 8080

# Run container
CMD ["npm", "start"]
