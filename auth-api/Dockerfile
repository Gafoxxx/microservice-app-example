# ---------- build ----------
FROM golang:1.18-alpine AS build

RUN apk add --no-cache git ca-certificates

WORKDIR /src
COPY . .

# El repo trae dep (Gopkg.toml) pero Go 1.18 usa modulos.
# El README del servicio indica justamente estos pasos.
RUN go mod init github.com/bortizf/microservice-app-example/auth-api && \
    go mod tidy && \
    CGO_ENABLED=0 GOOS=linux go build -o /out/auth-api .

# ---------- runtime ----------
FROM alpine:3.18

RUN apk add --no-cache ca-certificates
WORKDIR /app
COPY --from=build /out/auth-api .

ENV AUTH_API_PORT=8000
EXPOSE 8000

CMD ["./auth-api"]
