# =============================================================================
# POI Dashboard - Multi-stage Alpine Build
# =============================================================================
# Build: docker build -t poi-dashboard .
# Run:   docker run -p 3000:3000 poi-dashboard
# =============================================================================

# -----------------------------------------------------------------------------
# Stage 1: Builder
# -----------------------------------------------------------------------------
FROM node:20-alpine AS builder

# Install git for GitHub dependencies
RUN apk add --no-cache git

WORKDIR /app

# Copy package files first for better layer caching
COPY package.json yarn.lock ./

# Install all dependencies (including devDependencies for build)
# Use --ignore-scripts to skip postinstall (which runs lint before src/ exists)
RUN yarn install --frozen-lockfile --ignore-scripts

# Copy source and config files
COPY tsconfig.json tsconfig.paths.json tsconfig.test.json ./
COPY craco.config.js ./
COPY public ./public
COPY src ./src

# Build production React app
ENV PUBLIC_URL=/dashboard
ENV NODE_ENV=production
RUN yarn build-prod

# -----------------------------------------------------------------------------
# Stage 2: Runtime
# -----------------------------------------------------------------------------
FROM node:20-alpine AS runtime

# Add tini for proper signal handling
RUN apk add --no-cache tini

WORKDIR /app

# Create non-root user for security
RUN addgroup -g 1001 -S dashboard && \
    adduser -u 1001 -S dashboard -G dashboard

# Install only production dependencies needed for the server
RUN yarn add express@^4.18.2 --production && \
    yarn cache clean

# Copy built static files from builder
COPY --from=builder --chown=dashboard:dashboard /app/build ./build

# Copy the Express server
COPY --chown=dashboard:dashboard src/server.js ./src/server.js

# Switch to non-root user
USER dashboard

# Environment variables
ENV NODE_ENV=production
ENV PORT=3000

# Expose default port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=10s --retries=3 \
    CMD node -e "require('http').get('http://localhost:' + (process.env.PORT || 3000) + '/dashHealth', (r) => process.exit(r.statusCode === 200 ? 0 : 1)).on('error', () => process.exit(1))"

# Use tini as init system
ENTRYPOINT ["/sbin/tini", "--"]

# Run the Express server directly (no pm2)
CMD ["node", "src/server.js"]

