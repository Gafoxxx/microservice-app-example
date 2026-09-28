# ---------- build ----------
FROM maven:3.8-openjdk-8 AS build

WORKDIR /src
COPY pom.xml .
COPY src ./src

RUN mvn -B clean package -DskipTests

# ---------- runtime ----------
FROM eclipse-temurin:8-jre-alpine

WORKDIR /app
COPY --from=build /src/target/users-api-0.0.1-SNAPSHOT.jar app.jar

ENV SERVER_PORT=8083
EXPOSE 8083

ENTRYPOINT ["java", "-jar", "app.jar"]
