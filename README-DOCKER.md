# Despliegue con Docker Compose

Contenedorización de los 5 microservicios de la aplicación TODO, más Redis y Zipkin.

## Requisitos

- Docker Engine 20.10+
- Docker Compose v2 (`docker compose`) o v1 (`docker-compose`)

## Cómo levantar todo

```bash
docker compose build      # la primera vez tarda: compila Go, Maven y npm
docker compose up -d
docker compose ps
```

La UI queda en **http://localhost:8080**.

Usuarios de prueba (vienen hardcodeados en el auth-api):

| Usuario | Contraseña |
|---------|------------|
| admin   | admin      |
| johnd   | foo        |
| janed   | ddd        |

Para bajar todo:

```bash
docker compose down
```

## Puertos

| Servicio              | Puerto host | Notas                                  |
|-----------------------|-------------|----------------------------------------|
| frontend              | 8080        | punto de entrada del usuario           |
| auth-api              | 8000        | expuesto solo para pruebas con curl    |
| todos-api             | 8082        | expuesto solo para pruebas con curl    |
| users-api             | 8083        | expuesto solo para pruebas con curl    |
| zipkin                | 9411        | UI de trazas                           |
| redis                 | —           | solo accesible dentro de la red interna |

## Verificación manual

```bash
# 1. Obtener un token
TOKEN=$(curl -s -X POST http://localhost:8000/login \
  -d '{"username":"admin","password":"admin"}' | jq -r .accessToken)

# 2. Consultar el perfil del usuario
curl -H "Authorization: Bearer $TOKEN" http://localhost:8083/users/admin

# 3. Listar y crear TODOs
curl -H "Authorization: Bearer $TOKEN" http://localhost:8082/todos
curl -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  http://localhost:8082/todos -d '{"content":"probar la entrega"}'

# 4. Ver que el mensaje llegó a la cola
docker compose logs log-message-processor
```

El paso 4 es el que demuestra que el flujo asíncrono funciona: al crear o borrar
un TODO, el `todos-api` publica en el canal de Redis y el procesador en Python lo
imprime por stdout.

## Decisiones de diseño

**Una red bridge propia (`app-net`).** Los servicios se resuelven entre sí por
nombre de servicio (`http://users-api:8083`, `redis:6379`), no por IP ni por
`localhost`. Solo se publican al host los puertos que hacen falta.

**Redis no se expone al host.** Es un componente interno; nadie fuera de la red
de la aplicación necesita hablarle.

**El frontend corre el dev-server, no un Nginx con los estáticos.** No es
descuido: el código Vue llama a `/login` y `/todos` como rutas relativas al mismo
origen (`window.location.host`), y quien traduce esas rutas hacia `auth-api` y
`todos-api` es el proxy que define `config/index.js`. Si se sirvieran los
estáticos con Nginx habría que reescribir ese proxy en la config de Nginx. Para
producción real ese sería el siguiente paso.

**Builds multi-etapa en Go y Java.** La imagen final del `auth-api` es un
binario estático sobre Alpine, y la del `users-api` es solo el JAR sobre un JRE.
Así el compilador y las dependencias de build no viajan en la imagen final.

**El `auth-api` genera `go.mod` durante el build.** El repo trae `Gopkg.toml`
(dep, ya obsoleto) pero no `go.mod`. El Dockerfile ejecuta `go mod init` y
`go mod tidy`, que es exactamente lo que indica el README del servicio.

**`JWT_SECRET` centralizado en `.env`.** Los tres servicios que firman o validan
tokens leen la misma variable. Si no coincide, el login funciona pero cualquier
llamada posterior devuelve 401.

**El `log-message-processor` apunta a `/api/v1/spans` y los demás a
`/api/v2/spans`.** La librería `py_zipkin` serializa en Thrift, que es el formato
del endpoint v1; el resto de servicios usan JSON v2.

**Versiones fijadas a las que declara cada README** (Go 1.18, Node 8, Java 8,
Python 3.6, Redis 7). Son versiones viejas, pero el código depende de ellas:
subir Node rompe `node-sass` y subir Go rompe los imports sin sufijo de módulo.

## Problemas conocidos

- El primer `docker compose build` puede tardar bastante (Maven descarga el
  repositorio de dependencias y npm compila `node-sass`).
- El `frontend` tarda cerca de un minuto en responder después de `up`, porque
  webpack compila el bundle al arrancar. Si el navegador da "connection reset",
  esperar y recargar.
- Los TODOs viven en memoria del `todos-api`: al reiniciar ese contenedor se
  pierden. Es comportamiento original de la aplicación, no del despliegue.
