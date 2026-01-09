# Build stage
FROM rust:1.82-bookworm AS builder

WORKDIR /app

# Install build dependencies
RUN apt-get update && apt-get install -y \
    libpq-dev \
    pkg-config \
    && rm -rf /var/lib/apt/lists/*

# Clone and build rfd-api
RUN git clone https://github.com/oxidecomputer/rfd-api.git .
RUN cargo build --release -p rfd-api -p rfd-processor -p rfd-installer

# Runtime stage
FROM debian:bookworm-slim

WORKDIR /app

# Install runtime dependencies
RUN apt-get update && apt-get install -y \
    libpq5 \
    ca-certificates \
    ruby \
    ruby-dev \
    build-essential \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Install Node.js (for mermaid-cli)
RUN curl -fsSL https://deb.nodesource.com/setup_20.x | bash - \
    && apt-get install -y nodejs \
    && rm -rf /var/lib/apt/lists/*

# Install asciidoctor and related gems
RUN gem install asciidoctor asciidoctor-pdf asciidoctor-mermaid rouge

# Install mermaid CLI
RUN npm install -g @mermaid-js/mermaid-cli

# Copy binaries from builder
COPY --from=builder /app/target/release/rfd-api /usr/local/bin/
COPY --from=builder /app/target/release/rfd-processor /usr/local/bin/
COPY --from=builder /app/target/release/rfd-installer /usr/local/bin/

# Create directories for configs
RUN mkdir -p /app/rfd-api /app/rfd-processor

EXPOSE 8080

# Default command
CMD ["rfd-api"]
