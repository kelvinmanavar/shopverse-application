FROM golang:1.27-alpine AS builder

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download && go mod verify

COPY . .

RUN CGO_ENABLED=0 GOOS=linux go build -trimpath -ldflags="-w -s" -o /shopverse-api ./cmd/main.go

FROM gcr.io/distroless/static-debian12:nonroot

COPY --from=builder /shopverse-api /shopverse-api

USER nonroot:nonroot

EXPOSE 8080

ENTRYPOINT ["/shopverse-api"]
