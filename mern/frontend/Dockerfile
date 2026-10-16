# Dockerfile for the client side of the MERN stack application
FROM node:18.9.1

WORKDIR /app

COPY package*.json ./
RUN npm install

# Build-time value passed in by docker build / CI, available to Vite
ARG VITE_API_URL
ENV VITE_API_URL=$VITE_API_URL

COPY . .

EXPOSE 5173

CMD ["npm", "run", "dev"]