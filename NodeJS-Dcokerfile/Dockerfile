# Multi-stage build approach for a Node.js application

# Build stage
FROM node:18-alpine AS builder

# Set working directory for build stage
WORKDIR /build

# Add node user as non-root
RUN addgroup -g 1001 -S nodejs && \
    adduser -S -u 1001 -G nodejs nodeuser

# Set environment variables
ENV NODE_ENV=production \
    NPM_CONFIG_LOGLEVEL=error \
    NPM_CONFIG_CACHE=/tmp/.npm

# Copy package files first for better layer caching
COPY package.json package-lock.json* ./

# Install dependencies
# Use --production flag to avoid installing devDependencies
# Use ci instead of install for more reliable builds
RUN npm ci --production --no-audit --no-fund && \
    # Optional: If you need to build from source
    npm rebuild && \
    # Clean npm cache to reduce image size
    npm cache clean --force

# Copy application code
COPY --chown=nodeuser:nodejs . .

# If your app needs building (TypeScript, webpack, etc.)
RUN npm run build && \
    # Remove source files not needed in production
    find src -mindepth 1 -delete || true && \
    # Remove unnecessary files
    rm -rf node_modules/.cache tests/ .eslintrc .npmrc

# Runtime stage
FROM node:18-alpine AS runtime

# Add labels following OCI Image Spec
LABEL org.opencontainers.image.title="Node.js Application" \
      org.opencontainers.image.description="Production-ready Node.js application" \
      org.opencontainers.image.version="1.0.0" \
      org.opencontainers.image.authors="developers@example.com" \
      org.opencontainers.image.source="https://github.com/example/node-app" \
      org.opencontainers.image.licenses="MIT"

# Set build arguments with defaults
ARG PORT=3000
ARG NODE_ENV=production
ARG APP_HOME=/app

# Set environment variables
ENV PORT=$PORT \
    NODE_ENV=$NODE_ENV \
    APP_HOME=$APP_HOME

# Set working directory for runtime
WORKDIR $APP_HOME

# Install only the packages needed at runtime
RUN apk --no-cache add dumb-init curl && \
    addgroup -g 1001 -S nodejs && \
    adduser -S -u 1001 -G nodejs nodeuser

# Copy from build stage
COPY --from=builder --chown=nodeuser:nodejs /build/node_modules ./node_modules
COPY --from=builder --chown=nodeuser:nodejs /build/dist ./dist
COPY --from=builder --chown=nodeuser:nodejs /build/package.json ./

# Setup for persistent data
RUN mkdir -p logs data tmp && \
    chown -R nodeuser:nodejs logs data tmp

# Define volumes for persistent data
VOLUME ["$APP_HOME/logs", "$APP_HOME/data"]

# Switch to non-root user
USER nodeuser

# Define healthcheck
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD curl -f http://localhost:$PORT/health || exit 1

# Expose port
EXPOSE $PORT

# Use dumb-init as PID 1 to handle signals properly
ENTRYPOINT ["/usr/bin/dumb-init", "--"]

# Start the application
CMD ["node", "dist/server.js"]

# Override STOPSIGNAL to use SIGINT (what Node.js handles best)
STOPSIGNAL SIGINT