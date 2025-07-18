FROM node:14-alpine

WORKDIR /usr/src/app

# Copy only what is needed for runtime
COPY package*.json ./
COPY node_modules/ ./node_modules/
COPY index.js .
COPY db.js .
COPY models/ ./models/
COPY routes/ ./routes/

EXPOSE 5000

CMD ["node", "index.js"]