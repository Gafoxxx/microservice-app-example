# Microservice App - PRFT Devops Training

This is the application you are going to use through the whole traninig. This, hopefully, will teach you the fundamentals you need in a real project. You will find a basic TODO application designed with a [microservice architecture](https://microservices.io). Although is a TODO application, it is interesting because the microservices that compose it are written in different programming language or frameworks (Go, Python, Vue, Java, and NodeJS). With this design you will experiment with multiple build tools and environments. 

## Components
In each folder you can find a more in-depth explanation of each component:

1. [Users API](/users-api) is a Spring Boot application. Provides user profiles. At the moment, does not provide full CRUD, just getting a single user and all users.
2. [Auth API](/auth-api) is a Go application, and provides authorization functionality. Generates [JWT](https://jwt.io/) tokens to be used with other APIs.
3. [TODOs API](/todos-api) is a NodeJS application, provides CRUD functionality over user's TODO records. Also, it logs "create" and "delete" operations to [Redis](https://redis.io/) queue.
4. [Log Message Processor](/log-message-processor) is a queue processor written in Python. Its purpose is to read messages from a Redis queue and print them to standard output.
5. [Frontend](/frontend) Vue application, provides UI.

## Architecture

Take a look at the components diagram that describes them and their interactions.
![microservice-app-example](/arch-img/Microservices.png)

---

## Informe de contenerización

Esta sección resume el trabajo que se hizo para ejecutar la aplicación completa con Docker Compose. El detalle técnico está en [README-DOCKER.md](README-DOCKER.md).

### Objetivo

Empaquetar los 5 microservicios en contenedores y orquestarlos junto con Redis y Zipkin, de forma que toda la aplicación se levante con un solo comando.

### Archivos creados

| Archivo | Propósito |
|---------|-----------|
| `auth-api/Dockerfile` | Build multi-etapa: compila Go 1.18 y deja solo el binario sobre Alpine (~8 MB) |
| `users-api/Dockerfile` | Build multi-etapa: empaqueta con Maven y ejecuta el JAR sobre un JRE 8 |
| `todos-api/Dockerfile` | Node 8 Alpine, solo dependencias de producción, arranca con `node server.js` |
| `log-message-processor/Dockerfile` | Python 3.6 slim con `gcc` para compilar las dependencias de `py_zipkin` |
| `frontend/Dockerfile` | Node 8 con el dev-server, que hace de proxy hacia `auth-api` y `todos-api` |
| `*/.dockerignore` | Excluye `node_modules`, `target`, `.git` y artefactos locales del contexto de build |
| `docker-compose.yml` | Define los 7 servicios, la red `app-net`, las variables de entorno y los puertos |
| `.env` | Secreto `JWT_SECRET` compartido por los servicios que firman o validan tokens |
| `README-DOCKER.md` | Guía de uso, puertos, verificación y decisiones de diseño |

### Arquitectura desplegada

| Servicio | Tecnología | Puerto host | Se comunica con |
|----------|------------|-------------|-----------------|
| frontend | Vue / Node 8 | 8080 | auth-api, todos-api, zipkin |
| auth-api | Go 1.18 | 8000 | users-api, zipkin |
| users-api | Spring Boot / Java 8 | 8083 | zipkin |
| todos-api | Node 8 | 8082 | redis, zipkin |
| log-message-processor | Python 3.6 | — | redis, zipkin |
| redis | Redis 7 | — (solo red interna) | — |
| zipkin | Zipkin 2 | 9411 | — |

Todos los servicios están en la red bridge `app-net` y se localizan por nombre de servicio (por ejemplo `http://users-api:8083` o `redis:6379`).

### Cómo ejecutarlo

```bash
docker compose build
docker compose up -d
```

La aplicación queda disponible en http://localhost:8080 (usuario `admin`, contraseña `admin`). Para detenerla: `docker compose down`.

### Verificación realizada

| Prueba | Resultado |
|--------|-----------|
| Arranque de los 7 contenedores | Todos en estado `Up` |
| `POST /login` en auth-api | Devuelve un token JWT |
| `GET /users/admin` en users-api con el token | Devuelve el perfil del usuario |
| `GET /todos` y `POST /todos` en todos-api | Lista y crea TODOs |
| log-message-processor | Recibe por Redis el mensaje `CREATE` del TODO creado |
| Frontend en `localhost:8080` y login por su proxy | Responden HTTP 200 |
| Zipkin (`/api/v2/services`) | Registra trazas de auth-api, todos-api y users-api |

### Problemas encontrados y soluciones

- **auth-api no tenía `go.mod`.** El repositorio usa `dep` (`Gopkg.toml`), que ya no se usa. El Dockerfile ejecuta `go mod init` y `go mod tidy` durante el build.
- **El script `npm start` de todos-api usa `nodemon`**, que es una dependencia de desarrollo. El contenedor arranca con `node server.js`.
- **`node-sass` no funciona en Alpine con Node 8.** El frontend usa la imagen Debian `node:8` para poder descargar el binario precompilado.
- **`py_zipkin` envía las trazas en formato Thrift.** Por eso el log-message-processor apunta a `/api/v1/spans` y el resto de servicios a `/api/v2/spans`.
- **El log-message-processor se cae si Redis todavía no acepta conexiones.** Se añadió `restart: on-failure` para que se reinicie solo.
- **El secreto JWT debe coincidir en tres servicios.** Se centralizó en `.env`; si no coincide, el login funciona pero las demás llamadas devuelven 401.
- **Se quitó el campo `version`** de `docker-compose.yml`, porque Docker Compose v2 ya no lo usa y muestra una advertencia.

### Limitaciones conocidas (código original de la aplicación)

- Los TODOs se guardan en memoria del todos-api y se pierden al reiniciar el contenedor.
- todos-api puede asignar a un TODO nuevo un `id` que ya existe.
- El log-message-processor muestra un error al recibir el mensaje de confirmación de la suscripción a Redis, y no envía a Zipkin los mensajes que llegan sin `traceId`. Los mensajes de TODOs se procesan correctamente.
- El frontend usa el dev-server de webpack, que tarda cerca de un minuto en compilar al arrancar. Para producción habría que servir los estáticos con Nginx y trasladar el proxy a su configuración.
