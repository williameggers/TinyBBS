FROM mcr.microsoft.com/devcontainers/base:trixie

# Build prerequisites for server/CMakeLists.txt (server phases 1-4):
#   build-essential   C/C++ toolchain with C++17 support
#   cmake             CMake >= 3.20
#   libmbedtls-dev    mbedtls/sha256.h and libmbedcrypto (matching password hashes)
#   libcivetweb-dev   civetweb.h and libcivetweb (HTTP/WebSocket, phase 3)
# pthread comes with glibc; miniz is vendored in server/platform/.
#
# PlatformIO builds and flashes the ESP32-S3 firmware. Keep it in a virtualenv:
# Debian marks Python externally managed, and pip must not touch system Python.
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        build-essential \
        cmake \
        libmbedtls-dev \
        libcivetweb-dev \
        python3 \
        python3-venv \
    && rm -rf /var/lib/apt/lists/* \
    && python3 -m venv /opt/platformio-venv \
    && /opt/platformio-venv/bin/pip install --no-cache-dir --upgrade pip platformio

ENV PATH="/opt/platformio-venv/bin:${PATH}"
ENV PLATFORMIO_SETTING_ENABLE_TELEMETRY=false

# Allow the default non-root user to open the USB serial port passed through
# to the container. The host may use a different dialout GID; see BUILD.md.
RUN if getent group dialout >/dev/null && id vscode >/dev/null 2>&1; then \
        usermod -aG dialout vscode; \
    fi
