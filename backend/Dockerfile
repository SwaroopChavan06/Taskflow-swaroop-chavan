# ── Stage 1: Build ────────────────────────────────────────────────────────────
FROM golang:1.23-alpine AS builder

# git: required by go mod for VCS-stamped modules
# ca-certificates: needed for TLS in the final stage
RUN apk add --no-cache git ca-certificates tzdata

WORKDIR /app

# Copy module manifests first.  go.sum may not exist yet on a fresh clone —
# the wildcard prevents the COPY from failing in that case.
COPY go.mod go.sum* ./

# go mod tidy downloads dependencies AND generates/updates go.sum.
# This layer is cached until go.mod changes.
RUN go mod tidy

# Copy remaining source code
COPY . .

# Build a fully static binary.
# -s -w strips debug info (~30% smaller); -trimpath makes builds reproducible.
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
    go build \
    -ldflags="-s -w -extldflags=-static" \
    -trimpath \
    -o /bin/api \
    ./cmd/api

# ── Stage 2: Runtime ──────────────────────────────────────────────────────────
# distroless/static has no shell, no package manager, no root user by default.
# It is the minimal possible attack surface for a static Go binary.
FROM gcr.io/distroless/static-debian12:nonroot AS runtime

COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=builder /usr/share/zoneinfo                 /usr/share/zoneinfo

# The binary includes all migration SQL via embed.FS — no volume mounts needed.
COPY --from=builder /bin/api /api

EXPOSE 4000

ENTRYPOINT ["/api"]
