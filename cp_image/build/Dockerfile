# Multi-stage build for minimal size
FROM python:3.12-alpine3.19 AS builder

# Install build dependencies only for building packages
RUN apk add --no-cache \
    gcc \
    musl-dev \
    libffi-dev \
    openssl-dev \
    cargo \
    make

# Copy requirements and install packages
COPY cp_image/build/requirements.txt /tmp/requirements.txt
COPY cp_image/build/pip.conf /etc/pip.conf

# Install packages globally (no venv) and clean up aggressively
RUN pip install --no-cache-dir --prefix=/tmp/packages -r /tmp/requirements.txt && \
    find /tmp/packages -type d -name "__pycache__" -exec rm -rf {} + && \
    find /tmp/packages -type d -name "tests" -exec rm -rf {} + && \
    find /tmp/packages -type f -name "*.pyc" -delete && \
    find /tmp/packages -type f -name "*.pyo" -delete && \
    find /tmp/packages -type f -name "*.so" ! -name "*.so" -exec strip {} \; 2>/dev/null || true

# Final lightweight image
FROM python:3.12-alpine3.19 AS runtime

# Install only essential runtime dependencies
RUN apk add --no-cache \
    libffi \
    openssl \
    ca-certificates

# Copy installed packages from builder
COPY --from=builder /tmp/packages /usr/local
 
# Copy application files
COPY cp_image/scripts /module/scripts
COPY base_modules/scripts /module/scripts/

# Set working directory
WORKDIR /module

# Default command
CMD ["python3"]
