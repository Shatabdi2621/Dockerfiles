# Python Dockerfile
Let's create a production-ready Python Application based Dockerfile with best practices and I would be explaining every component in detail.

## Get Started
1. **Base **
- **Purpose**: Defines the foundation of our Docker image
- **Why Python 3.9**: A stable Python version with good library support
- **Why slim-buster**: A stripped-down Debian-based image that's much smaller than the standard Python image, reducing final image size
```bash
FROM python:3.9-slim-buster
```

2. **Environment Variables**
- **PYTHONDONTWRITEBYTECODE=1**: Prevents Python from writing .pyc files (saves space)
- **PYTHONUNBUFFERED=1**: Ensures Python output is sent straight to the terminal without buffering
- **PYTHONFAULTHANDLER=1**: Helps debug crashes by displaying tracebacks
- **PIP_NO_CACHE_DIR=off**: Disables pip's cache to reduce image size
- **PIP_DISABLE_PIP_VERSION_CHECK=on**: Stops pip from checking for new versions
- **PIP_DEFAULT_TIMEOUT=100**: Sets a default timeout for pip operations
- **/POETRY_HOME/POETRY_VIRTUALENVS_IN_PROJECT/POETRY_NO_INTERACTION/PYSETUP_PATH**: Environment variable 
```bash
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PYTHONFAULTHANDLER=1 \
    PIP_NO_CACHE_DIR=off \
    PIP_DISABLE_PIP_VERSION_CHECK=on \
    PIP_DEFAULT_TIMEOUT=100 \
    POETRY_VERSION=1.1.13 \
    POETRY_HOME="/opt/poetry" \
    POETRY_VIRTUALENVS_IN_PROJECT=true \
    POETRY_NO_INTERACTION=1 \
    PYSETUP_PATH="/opt/pysetup"
```

3. **Working Directory**
- Sets the working directory inside the container to /app
- All subsequent commands will run from this directory
```bash
WORKDIR /app
```

4. **System Dependencies**
- **build-essential**: Required for compiling some Python packages
- **curl**: For downloading files and health checks
- **netcat**: Useful for network diagnostics
- **gcc**: C compiler needed for some Python dependencies
- The cleanup commands remove package lists to reduce image size
```bash
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        build-essential \
        curl \
        netcat \
        gcc \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*
```

5. **Poetry Installation**
- Downloads and installs Poetry for dependency management
- Creates a symlink so poetry is available in the PATH
```bash
RUN curl -sSL https://raw.githubusercontent.com/python-poetry/poetry/master/get-poetry.py | python - \
    && ln -s /opt/poetry/bin/poetry /usr/local/bin/poetry
```

6. **Dependency Installation**
- Copies just the dependency files first (not the whole app)
- This leverages Docker caching - dependencies only reinstall if these files change
- **--no-dev** flag ensures only production dependencies are installed
```bash
RUN curl -sSL https://raw.githubusercontent.com/python-poetry/poetry/master/get-poetry.py | python - \
    && ln -s /opt/poetry/bin/poetry /usr/local/bin/poetry
```

7. **Security: Non-Root User**
- Creates a system user and group called "app"
- Switches to this non-root user for security (limits potential damage if the container is compromised)
```bash
RUN addgroup --system app \
    && adduser --system --ingroup app app

USER app
```

8. **Application Code**
- Copies the application code into the container
- Changes ownership of all files to the non-root user
```bash
COPY . /app/
RUN chown -R app:app /app
```

9. **Health Check**
- Configures Docker to periodically check if the application is healthy
- Checks every 30 seconds, with a 30-second timeout
- Allows a 5-second grace period when starting
- Tries 3 times before marking the container as unhealthy
```bash
HEALTHCHECK --interval=30s --timeout=30s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1
```

10. **Port Configuration**
- Documents that the container listens on port 8000
- This is informational only (doesn't actually publish the port)
```bash
EXPOSE 8000
```

11. **Startup Command**
- Specifies the command to run when the container starts
- Uses Gunicorn as a production-ready WSGI server
- Binds to all interfaces (0.0.0.0) on port 8000
- Points to the application entry point
```bash
CMD ["gunicorn", "--bind", "0.0.0.0:8000", "app.wsgi:application"]
``` 