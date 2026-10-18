# Stage-1: Builder stage
FROM golang:1.27-alpine AS builder

WORKDIR /app

COPY go.mod go.sum ./

RUN go mod download

COPY . .

RUN CGO_ENABLED=0 go build -o devboard-backend .

# Stage-2: Runner stage

FROM alpine:latest

WORKDIR /app

RUN adduser -D -u 10001 app && chown -R app:app /app

COPY --from=builder /app/devboard-backend ./devboard-backend

USER app

Expose 8080

ENTRYPOINT ["./devboard-backend"]
