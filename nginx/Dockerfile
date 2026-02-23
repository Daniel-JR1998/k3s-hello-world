# Use the official lightweight Nginx image
FROM nginx:1.25-alpine

# Remove the default page
RUN rm -rf /usr/share/nginx/html/*

# Copy our custom Hello World page
COPY index.html /usr/share/nginx/html/index.html

# Use a non-root user for security
RUN addgroup -g 1001 -S appgroup && \
    adduser -u 1001 -S appuser -G appgroup && \
    chown -R appuser:appgroup /usr/share/nginx/html && \
    chown -R appuser:appgroup /var/cache/nginx && \
    chown -R appuser:appgroup /var/log/nginx && \
    touch /var/run/nginx.pid && \
    chown appuser:appgroup /var/run/nginx.pid

# Nginx config that listens on 8080 (non-privileged port)
COPY nginx.conf /etc/nginx/nginx.conf

USER appuser

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD wget -qO- http://localhost:8080/ || exit 1
