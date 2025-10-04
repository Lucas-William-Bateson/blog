# Multi-stage build for production optimization
FROM alpine:3.21.3 AS base

# Add metadata labels
LABEL maintainer="${MAINTAINER_EMAIL:-your-email@example.com}"
LABEL project="${APP_NAME:-myproject}"
LABEL version="${VERSION:-1.0.0}"
LABEL description="Mini deploy template application"

# Install necessary packages and create non-root user
RUN apk add --no-cache wget curl && \
    addgroup -g 1001 -S appgroup && \
    adduser -S appuser -u 1001 -G appgroup

# Set working directory and change ownership to non-root user
WORKDIR /app
RUN chown -R appuser:appgroup /app

# Copy application files (if any)
# COPY . /app

# Switch to non-root user
USER appuser

# Expose port
EXPOSE ${APP_PORT:-8000}

# Add health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD wget --quiet --tries=1 --spider http://localhost:${APP_PORT:-8000} || exit 1

# Default command
CMD ["sh", "-c", "echo 'Application starting on port ${APP_PORT:-8000}...' && \
     echo 'This is a test Docker container for ${APP_NAME:-myproject}' && \
     echo 'Container is healthy and running' && \
     while true; do \
       echo 'Service is running...' && \
       sleep 30; \
     done"]