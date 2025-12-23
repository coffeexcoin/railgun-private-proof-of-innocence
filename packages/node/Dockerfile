# =============================================================================
# POI Node - Multi-stage Alpine Build
# =============================================================================
# Build: docker build -t poi-node .
# Run:   docker run -e DOCUMENT_DB_URL=mongodb://host:27017 -p 3010:3010 poi-node
# =============================================================================

# -----------------------------------------------------------------------------
# Stage 1: Builder
# -----------------------------------------------------------------------------
FROM node:20-alpine AS builder

# Install build dependencies for native modules (leveldown)
RUN apk add --no-cache python3 make g++ git

WORKDIR /app

# Copy package files first for better layer caching
COPY package.json yarn.lock ./

# Install all dependencies (including devDependencies for build)
# Use legacy-peer-deps to handle peer dependency conflicts
RUN yarn install --frozen-lockfile

# Copy source files
COPY tsconfig.json ./
COPY src ./src

# Build TypeScript to dist/
RUN yarn build

# Prune dev dependencies for production
RUN yarn install --frozen-lockfile --production && \
    yarn cache clean

# -----------------------------------------------------------------------------
# Stage 2: Runtime
# -----------------------------------------------------------------------------
FROM node:20-alpine AS runtime

# Add tini for proper signal handling
RUN apk add --no-cache tini

WORKDIR /app

# Create non-root user for security
RUN addgroup -g 1001 -S poi && \
    adduser -u 1001 -S poi -G poi

# Copy built artifacts and production dependencies from builder
COPY --from=builder --chown=poi:poi /app/dist ./dist
COPY --from=builder --chown=poi:poi /app/node_modules ./node_modules
COPY --from=builder --chown=poi:poi /app/package.json ./

# Create directory for LevelDB persistence
RUN mkdir -p /app/engine.db && chown poi:poi /app/engine.db

# Switch to non-root user
USER poi

# Environment variables
ENV NODE_ENV=production
ENV HOST=0.0.0.0
ENV PORT=3010
ENV DEBUG=poi:*

# Expose default port
EXPOSE 3010

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
    CMD node -e "require('http').get('http://localhost:' + (process.env.PORT || 3010) + '/', (r) => process.exit(r.statusCode === 200 ? 0 : 1)).on('error', () => process.exit(1))"

# Use tini as init system
ENTRYPOINT ["/sbin/tini", "--"]

# Run the application directly (no pm2)
CMD ["node", "dist/main.js"]

