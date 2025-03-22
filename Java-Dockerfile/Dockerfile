# Multi-stage build: Build stage
FROM maven:3.8.6-openjdk-11 AS build

# Set working directory for the build stage
WORKDIR /build

# Cache Maven dependencies by copying only pom.xml first
COPY pom.xml .
RUN mvn dependency:go-offline

# Copy source code
COPY src/ ./src/

# Build the application
RUN mvn clean package -DskipTests

# Extract runtime JRE using jlink to create a custom, minimal JRE
FROM openjdk:11-slim AS jre-build
RUN mkdir -p /javaruntime
RUN jlink \
    --add-modules java.base,java.logging,java.sql,java.naming,java.desktop,java.management,java.security.jgss,java.instrument,jdk.unsupported \
    --strip-debug \
    --no-man-pages \
    --no-header-files \
    --compress=2 \
    --output /javaruntime

# Final stage: Runtime image
FROM debian:buster-slim

# Set ARGs and ENVs for configuration
ARG APP_VERSION=1.0.0
ARG BUILD_DATE
ARG VCS_REF
ARG PORT=8080

# Set environment variables
ENV APP_HOME=/app \
    JAVA_HOME=/opt/java \
    APP_VERSION=${APP_VERSION} \
    PORT=${PORT} \
    SPRING_PROFILES_ACTIVE=production

# Add Labels following OCI Image Spec
LABEL org.opencontainers.image.title="Java Application" \
      org.opencontainers.image.description="Production-ready Java application" \
      org.opencontainers.image.version=${APP_VERSION} \
      org.opencontainers.image.created=${BUILD_DATE} \
      org.opencontainers.image.source="https://github.com/yourusername/yourrepo" \
      org.opencontainers.image.revision=${VCS_REF} \
      org.opencontainers.image.vendor="Your Organization" \
      org.opencontainers.image.authors="you@example.com" \
      org.opencontainers.image.licenses="MIT"

# Install minimal required packages
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    fontconfig \
    tzdata \
    && rm -rf /var/lib/apt/lists/*

# Setup custom JRE
COPY --from=jre-build /javaruntime $JAVA_HOME
ENV PATH="${JAVA_HOME}/bin:${PATH}"

# Create a non-root user to run the application
RUN groupadd -r javauser && useradd -r -g javauser javauser

# Create app directory and set permissions
RUN mkdir -p ${APP_HOME}/config ${APP_HOME}/logs ${APP_HOME}/data
WORKDIR ${APP_HOME}

# Copy the built JAR from the build stage
COPY --from=build /build/target/*.jar ${APP_HOME}/app.jar

# Copy application configuration
COPY config/ ${APP_HOME}/config/

# Create volumes for persistent data
VOLUME ["${APP_HOME}/logs", "${APP_HOME}/data"]

# Change ownership to non-root user
RUN chown -R javauser:javauser ${APP_HOME}
USER javauser

# Define a healthcheck
HEALTHCHECK --interval=30s --timeout=3s --start-period=60s --retries=3 \
  CMD curl -f http://localhost:${PORT}/actuator/health || exit 1

# Document which port the application uses
EXPOSE ${PORT}

# Use a shell form with exec to properly handle signals
ENTRYPOINT ["sh", "-c", "exec java \
            -XX:+UseContainerSupport \
            -XX:MaxRAMPercentage=75.0 \
            -XX:+PrintFlagsFinal \
            -Djava.security.egd=file:/dev/./urandom \
            -Dserver.port=${PORT} \
            -jar app.jar"]

# Override STOPSIGNAL to use SIGTERM
STOPSIGNAL SIGTERM