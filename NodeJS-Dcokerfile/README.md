# NodeJS Dockerfile
Let's create a production-ready NodeJS Application based Dockerfile with best practices and I would be explaining every component in detail.

## Get Started
1. **Multi-Stage Build Approach**
- **Purpose**: Separates building from running the application
- **Benefits**:
a. Significantly smaller final image (no build tools or source code)
b. Clear separation of build and runtime environments
c. Better security by not including development tools in production
```bash
FROM node:18-alpine AS builder

FROM node:18-alpine AS runtime
```

2. **Base Image**
- **Alpine Linux**: An extremely small Linux distribution (~5MB)
- **Node.js 18**: LTS (Long Term Support) version for stability
- **Size advantage**: ~113MB vs ~900MB for non-Alpine Node.js images
```bash
FROM node:18-alpine
```

3. **Non-Root User Creation**
- **Security best practice**: Prevents running as root
- **Alpine syntax**: Uses Alpine's lightweight user management
- **Explicit UID/GID**: Enables consistent permissions across environments
```bash
RUN addgroup -g 1001 -S nodejs && \
    adduser -S -u 1001 -G nodejs nodeuser
```

4. **Environment Variable Configuration**
- **NODE_ENV=production**: Activates production optimizations in Node.js and npm
- **NPM_CONFIG_LOGLEVEL**=error: Reduces npm output to errors only
- **NPM_CONFIG_CACHE**: Places npm cache in a temporary location
```bash
ENV NODE_ENV=production \
    NPM_CONFIG_LOGLEVEL=error \
    NPM_CONFIG_CACHE=/tmp/.npm
```

5. **Dependency Caching Strategy**
- **Separate copy step**: Leverages Docker caching - dependencies only reinstall if
 package files change
- **npm ci**: More reliable than npm install for CI/CD environments
- **--production flag**: Skips devDependencies
- **--no-audit/--no-fund**: Skips security audits and funding messages to speed up install
```bash
COPY package.json package-lock.json* ./
RUN npm ci --production --no-audit --no-fund
```

6. **Build Process**
- **chown**: Sets proper ownership during copy
- **bu**ild step**: Compiles TypeScript/transpiles code/bundles assets as needed
```bash
COPY --chown=nodeuser:nodejs . .
RUN npm run build
```

7. **Cleanup**
- **Source removal**: Removes source files not needed in production
- **Cache/test cleanup**: Removes files only needed during development
- **Configuration removal**: Removes linting and npm configuration files
```bash
RUN find src -mindepth 1 -delete || true && \
    rm -rf node_modules/.cache tests/ .eslintrc .npmrc
```

8. **Runtime Dependencies**
- **dumb-init**: Proper init system for signal handling
- **curl**: For healthchecks
- **--no-cache**: Prevents caching package lists to keep image small
```bash
RUN apk --no-cache add dumb-init curl
```

9. **Selective File Copying**
- **Selective copying**: Only brings necessary files to runtime image
- **Proper permissions**: Sets ownership appropriately for security
```bash
COPY --from=builder --chown=nodeuser:nodejs /build/node_modules ./node_modules
COPY --from=builder --chown=nodeuser:nodejs /build/dist ./dist
COPY --from=builder --chown=nodeuser:nodejs /build/package.json ./
```

10. **Volume Configuration**
- **Directory preparation**: Creates directories with proper permissions
- **Volume declaration**: Marks directories as potentially externally mounted
- **Use case**: Allows persistent storage for logs and application data
```bash
RUN mkdir -p logs data tmp && \
    chown -R nodeuser:nodejs logs data tmp
VOLUME ["$APP_HOME/logs", "$APP_HOME/data"]
```

11. **Advanced Healthcheck**
- **Interval**: Checks every 30 seconds
- **Timeout**: Allows 5 seconds for response
- **Start period**: Gives 10 seconds grace period at startup
- **Retries**: Allows 3 failures before marking container unhealthy
```bash
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD curl -f http://localhost:$PORT/health || exit 1
```

12. **OCI Labels**
- Follows Open Container Initiative standards
- Provides metadata about the image
```bash
OCI Labels
```

13. **Build Arguments**
- Allows customization at build time
- Sets defaults if not specified
```bash
ARG PORT=3000
ARG NODE_ENV=production
```

14. **Process Management**
- **dumb-init**: Acts as PID 1 to handle signals properly
- **Separate ENTRYPOINT/CMD**: Allows for command overriding while keeping the init system
- **Exec form**: Uses JSON array syntax for proper signal handling
```bash
ENTRYPOINT ["/usr/bin/dumb-init", "--"]
CMD ["node", "dist/server.js"]
```

15. **Signal Handling**
- **SIGINT**: The signal Node.js handles best for graceful shutdowns
- **Purpose**: Allows proper cleanup before container stops
```bash
STOPSIGNAL SIGINT
```