# Java Dockerfile
Let's create a production-ready Java Application based Dockerfile with best practices and I would be explaining every component in detail.

## Get Started
1. **Multi-Stage Build**
- **Purpose**: Multi-stage builds allow you to use one image for building and another for running
- **Advantage**: Final image contains only runtime components, not build tools
- **Maven image**: Contains both Java and Maven for building the applicationsize
```bash
FROM maven:3.8.6-openjdk-11 AS build
```

2. **Dependency Caching**
- **Docker layer caching**: By copying and resolving dependencies separately, Docker can cache this layer
- **Optimization**: Speeds up subsequent builds when only source code changes, not dependencies
```bash
COPY pom.xml .
RUN mvn dependency:go-offline
```

3. **Custom JRE Creation (jlink)**
- **Purpose**: Creates a minimal custom JRE with only necessary modules
- **jlink tool**: Java 9+ feature that creates optimized runtime images
- **Benefits**: Significantly reduces image size, improves startup time, reduces attack surface
```bash
FROM openjdk:11-slim AS jre-build
RUN jlink \
    --add-modules java.base,java.logging,java.sql,java.naming,java.desktop,java.management,java.security.jgss,java.instrument,jdk.unsupported \
    --strip-debug \
    --no-man-pages \
    --no-header-files \
    --compress=2 \
    --output /javaruntime
```

4. **Build Arguments (ARGs)**
- **ARG vs ENV**: ARGs are only available during build time; ENVs persist in the final image
- **Purpose**: Enables parameterization of the build process
- **Use case**: These values can be passed at build time, e.g., docker build --build-arg BUILD_DATE=$(date -u +'%Y-%m-%dT%H:%M:%SZ')
```bash
ARG APP_VERSION=1.0.0
ARG BUILD_DATE
ARG VCS_REF
ARG PORT=8080
```

5. **OCI Image Labels**
- **Purpose**: Metadata that follows the Open Container Initiative (OCI) standard
- **Benefits**: Provides standardized information about the image
- **Usage**: These labels are visible when inspecting the image with docker inspect
```bash
LABEL org.opencontainers.image.title="Java Application" \
      org.opencontainers.image.description="Production-ready Java application" \
      ...
```

6. **Volume Definitions**
- **Purpose**: Defines mountable volumes for persistent data
- **Benefit**: Data in these directories persists beyond container lifecycle
- **Usage**: Can be mounted to host directories or managed volumes
```bash
VOLUME [ "${APP_HOME}/logs", "${APP_HOME}/data" ]
```

7. **Healthcheck Configuration**
- **Purpose**: Tells Docker how to check if the container is healthy
- **Interval**: How often to perform the check (30 seconds)
- **Timeout**: How long to wait for a response (3 seconds)
- **Start-period**: Initial grace period to allow application startup (60 seconds)
- **Retries**: Number of consecutive failures before marking unhealthy (3)
```bash
HEALTHCHECK --interval=30s --timeout=3s --start-period=60s --retries=3 \
  CMD curl -f http://localhost:${PORT}/actuator/health || exit 1
```

8. **Optimized Java Runtime Settings**
- **UseContainerSupport**: Enables container-aware memory limits
- **MaxRAMPercentage**: Sets maximum heap size as a percentage of available memory
- **PrintFlagsFinal**: Logs all JVM flags for debugging
- **exec form**: Ensures proper signal handling
- **urandom sourcing**: Improves startup time by using a non-blocking entropy source
```bash
ENTRYPOINT ["sh", "-c", "exec java \
            -XX:+UseContainerSupport \
            -XX:MaxRAMPercentage=75.0 \
            -XX:+PrintFlagsFinal \
            -Djava.security.egd=file:/dev/./urandom \
            -Dserver.port=${PORT} \
            -jar app.jar"]
```

9. **STOPSIGNAL Instruction**
- **Purpose**: Specifies the signal that Docker should send to stop the container
- **SIGTERM**: Gives the application a chance to shut down gracefully
- **Behavior**: Java applications can register shutdown hooks that execute when SIGTERM is received
```bash
STOPSIGNAL SIGTERM
```