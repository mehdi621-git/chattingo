# Stage 1: Nodejs build environment
FROM node:20-alpine AS main
WORKDIR /app
COPY package*.json ./
RUN npm install

# Stage 2: Build React app
FROM main AS build
WORKDIR /app
COPY . .
ARG REACT_APP_API_URL
ENV REACT_APP_API_URL=$REACT_APP_API_URL
RUN npm run build

# Stage 3:  Nginx runtine server
FROM nginx:alpine
WORKDIR /app
COPY --from=build /app/build /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
