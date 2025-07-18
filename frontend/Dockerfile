# Build stage (not used in final image, Jenkins does the build)
FROM nginx:alpine

# Copy the build output from Jenkins workspace (assume build/ is present)
COPY build /usr/share/nginx/html

EXPOSE 3000

CMD ["nginx", "-g", "daemon off;"]
