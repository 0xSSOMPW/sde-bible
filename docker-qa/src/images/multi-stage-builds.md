# Q: What are Multi-Stage Builds and why are they important?

**Answer:**

Multi-stage builds allow you to use **multiple `FROM` statements** in a single Dockerfile. Each `FROM` starts a new "stage" of the build. You can selectively copy artifacts from one stage to another, leaving behind everything you don't need in the final image.

### The Problem Without Multi-Stage Builds
In a typical build, you need compilers, build tools, and dev dependencies to compile your application. If you use a single stage, all of those tools end up in your final production image, making it bloated and insecure.

```dockerfile
# ❌ Single-stage: Final image includes Go compiler, source code, build tools
FROM golang:1.21
WORKDIR /app
COPY . .
RUN go build -o myapp
CMD ["./myapp"]
# Final image size: ~800MB (includes entire Go toolchain!)
```

### The Solution: Multi-Stage Build

```dockerfile
# Stage 1: Build (named "builder")
FROM golang:1.21 AS builder
WORKDIR /app
COPY . .
RUN go build -o myapp

# Stage 2: Production (tiny final image)
FROM alpine:3.18
WORKDIR /app
COPY --from=builder /app/myapp .
CMD ["./myapp"]
# Final image size: ~15MB (just the binary + Alpine!)
```

### How It Works
1. **Stage 1 ("builder")**: Uses the full `golang` image (800MB+) to compile the Go binary.
2. **Stage 2**: Starts fresh from a tiny `alpine` image (5MB) and copies *only* the compiled binary from the builder stage using `COPY --from=builder`.
3. The final image contains nothing from stage 1 except the single file you explicitly copied.

### Real-World Node.js Example

```dockerfile
# Stage 1: Install dependencies and build
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 2: Production
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY --from=builder /app/dist ./dist
EXPOSE 3000
CMD ["node", "dist/index.js"]
```

### Benefits
- **Dramatically smaller images** (often 10-50x reduction).
- **Better security** (no compilers, source code, or dev tools in production).
- **Faster deployments** (smaller images push/pull faster).
- **Single Dockerfile** (no need for separate `Dockerfile.dev` and `Dockerfile.prod`).

> [!TIP]
> You can also copy from external images without defining them as a stage:
> `COPY --from=nginx:latest /etc/nginx/nginx.conf /etc/nginx/`
