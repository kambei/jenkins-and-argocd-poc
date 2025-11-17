# Use a minimal Alpine-based Nginx image
FROM nginx:alpine
LABEL maintainer="POC Pipeline"

# Remove default index file and add a simple static file
RUN rm /usr/share/nginx/html/index.html
COPY index.html /usr/share/nginx/html/index.html