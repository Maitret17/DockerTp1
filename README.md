# DockerTp1

## 1.1

If you put your data in your dockerfile, allows for anyone with access to the image or the repository to read them. Which is obviously not wanted with sensitive data. The flag -e allows you to provide them at runtime instead, so they are never stored in the image. It also means you don’t have to fully rebuild the image when you want to change the variables, as you can just pass different values when starting a new container with the flag -e.

## 1.2

A container is disposable, when it is deleted, everything written inside is deleted, including the database. A volume is a folder outside the container that contains data you want to keep, so that it survives when the container is deleted or recreated. A new container can then use the same volume and find all the data again.

## 1.3

Dockerfile:

```dockerfile
FROM postgres:17.2-alpine

COPY 01-CreateScheme.sql /docker-entrypoint-initdb.d/
COPY 02-InsertData.sql /docker-entrypoint-initdb.d/
```

Commands:

```powershell
# Create a network so the containers can communicate by name
docker network create app-network
# Build the image
docker build -t tp/database ./database
# Run the database with credentials and a persistent volume
docker run -d --name database --network app-network -e POSTGRES_DB=db -e POSTGRES_USER=usr -e POSTGRES_PASSWORD=pwd -v "${PWD}/database/data:/var/lib/postgresql/data" tp/database
# Run Adminer to browse the database (http://localhost:8090)
docker run -d --name adminer -p 8090:8080 --network app-network adminer
```

## 1.4

To build and run a java application, we need build tools (JDK and Maven), as well as a Java runtime (JRE) and the .jar. However once the application is built we no longer need the build tools. Without a multistage build, they would stay in the final image, which is both inefficient and a potential security risk. A multistage build compiles the Java code in an image first, then copies only the resulting .jar needed to run the program into a new image, alongside the JRE.

**Build stage**

`FROM eclipse-temurin:21-jdk-alpine AS myapp-build`:
We start from an image containing the JDK 21, which we need to compile the code. We name this stage myapp-build so we can get files from it later.

`ENV MYAPP_HOME=/opt/myapp`:
We create a variable with the path of the folder where the application will be.

`WORKDIR $MYAPP_HOME`:
We move into this folder, so every next command runs inside it.

`RUN apk add --no-cache maven`:
We install Maven, which we need to build the project.

`COPY pom.xml .`:
We copy the pom.xml, which tells Maven which libraries the project needs and how to build it.

`RUN mvn dependency:go-offline`:
Maven downloads all the libraries listed in the pom.xml. Since this step only depends on the pom.xml, Docker caches it, so the libraries are not downloaded again on every build unless the pom.xml changes.

`COPY src ./src`:
We copy the source code of the application.

`RUN mvn package -DskipTests`:
Maven compiles the code and packages it into a .jar, without running the tests to make the build faster.

**Run stage**

`FROM eclipse-temurin:21-jre-alpine`:
We start a new image, containing only the Java runtime (JRE), since we only need to run the program now.

`ENV MYAPP_HOME=/opt/myapp`:
`WORKDIR $MYAPP_HOME`
Same as before. Since this is a new image, nothing from the first stage is kept, so we define the folder again.

`COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar`:
We copy only the .jar from the build stage into this image, and rename it myapp.jar

`ENTRYPOINT ["java", "-jar", "myapp.jar"]`
The command executed when the container starts, which runs the application.

## 1.5

The reverse proxy is the only entry point of the application. It receives the requests from the outside and forwards them to the backend through a private network. This way, the backend and the database are never directly exposed to outsiders, preventing any unwanted access or tampering.

## 1.6

If we don't have a docker compose, we would have to write a long docker run command for each container, in the right order, every time we want to start the application. With docker compose, everything is described once in a single file, and one command (docker compose up) starts, stops or rebuilds the whole application. Since this file is saved in the repository, anyone can launch the exact same application on any machine.

## 1.7

- `docker compose up -d –build` : Builds the images and starts all containers in the background
- `docker compose down`: Stops and removes the containers and networks
- `docker compose down -v`: Same, but also deletes the volumes
- `docker compose ps`: Shows the state of each container
- `docker compose logs httpd`: Shows the logs of a service
- `docker compose up -d --build httpd`: Rebuilds and restarts only one service
- `docker compose restart backend`: Restarts one service

## 1.8

```yaml
services:
  backend:
    build: ./simple-api
    container_name: backend
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://database:5432/${POSTGRES_DB}
      SPRING_DATASOURCE_USERNAME: ${POSTGRES_USER}
      SPRING_DATASOURCE_PASSWORD: ${POSTGRES_PASSWORD}
    networks:
      - back-network
      - front-network
    depends_on:
      database:
        condition: service_healthy
    restart: unless-stopped

  database:
    build: ./database
    container_name: database
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - db-data:/var/lib/postgresql/data
    networks:
      - back-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 5s
      timeout: 5s
      retries: 10
    restart: unless-stopped

  httpd:
    build: ./http-server
    container_name: httpd
    ports:
      - "80:80"
    networks:
      - front-network
    depends_on:
      - backend
    restart: unless-stopped

networks:
  front-network:
  back-network:

volumes:
  db-data:
```

Each service is built from its own folder. The passwords come from a .env file. Only httpd is exposed (port 80). Two networks keep httpd away from the database. A volume keeps the database data. The backend waits for the database to be ready, and crashed containers restart automatically.

## 1.9

```powershell
docker login
docker tag dockertp1-database maitret17/tp-database:1.0
docker tag dockertp1-backend maitret17/tp-backend:1.0
docker tag dockertp1-httpd maitret17/tp-httpd:1.0
docker push maitret17/tp-database:1.0
docker push maitret17/tp-backend:1.0
docker push maitret17/tp-httpd:1.0
```

## 1.10

Putting images in an online repository lets anyone download and run the exact same image without rebuilding it. Version tags also make it easy to know which version is used and to go back to an older one if needed.
