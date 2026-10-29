FROM golang:1.26-alpine AS builder

# go-sqlite3 requires CGO, which needs a C compiler and musl headers on Alpine
RUN apk add --no-cache gcc musl-dev

WORKDIR /app

COPY go.mod go.sum ./
RUN --mount=type=cache,target=/go/pkg/mod go mod download

COPY . .
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    CGO_ENABLED=1 go build -o /bin/whatsapp-bridge .

FROM alpine:3.21

RUN apk add --no-cache ca-certificates

WORKDIR /app

COPY --from=builder /bin/whatsapp-bridge /app/whatsapp-bridge

# Session, message DB and media live in /app/store (mount a volume here)
EXPOSE 8080

CMD ["/app/whatsapp-bridge"]
