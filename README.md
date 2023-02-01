# devops-sitot

## TP 1

### 1-1 Document your database container essentials: commands and Dockerfile.
<code>Docker pull postgre</code> pour récupérer l'image<br>
<code>Docker build  -t some-postgres .</code> pour construire l'image avec le DockerFile contenu dans le répertoire <br>

Dockerfile :

```FROM postgres:14.1-alpine
ENV POSTGRES_DB=db \
    POSTGRES_USER=sitot \
    POSTGRES_PASSWORD=admin
COPY 01-CreateScheme.sql /docker-entrypoint-initdb.d
COPY 02-InsertData.sql /docker-entrypoint-initdb.d
```

Dans ce Dockerfile on renseigne différentes variables d'environements utilisé pour se connecter, on copie aussi des scripts SQL dans l'image qui vont être exécuter au démarage du container<br>
<code>docker run --network app-network -d --name some-postgres postgres-sitot</code> pour récupérer l'image

### 1-2 Why do we need a multistage build? And explain each step of this dockerfile.

Dockerfile :
```
\# Build

FROM maven:3.8.6-amazoncorretto-17 AS myapp-build \# On précise depuis quelle image générique on veux build notre image<br>
ENV MYAPP_HOME /opt/myapp \# on déclare la variable d'environement MYAPP_HOME avec le chemin /opt/myapp<br>
WORKDIR $MYAPP_HOME\# on affècte le répéertoire de travail avec la variable MYAPP_HOME<br>
COPY simpleapi/pom.xml . \# On copie le pom.xml et le code source  dans l'image<br>
COPY simpleapi/src ./src<br>
RUN mvn package -DskipTests \# On execute cette commande pour générer l'api<br>

\# Run

<code>FROM amazoncorretto:17<br>
ENV MYAPP_HOME /opt/myapp<br>
WORKDIR $MYAPP_HOME<br>
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar \# on récupère le fichier générer dans le build<br>
ENTRYPOINT java -jar myapp.jar \# on lance l'api générer</code>
```
Le multistage build permet de générer l'application avec l'image  maven:3.8.6-amazoncorretto-17 et de la lancer avec l'image amazoncorretto-17.

### 1-3 Document docker-compose most important commands.
<code>docker-compose build</code> pour constuire l'image<br>
<code>docker-compose up -d</code> pour lancer le container en tache de fond
### 1-4 Document your docker-compose file.
```
version: '3.3'services:
  backend:
    container_name: backend
    build: ./simple-api
    networks:
      - app-network
    depends_on:
      - database

  database:
    container_name: database
    restart: always
    build: ./database
    networks:
      - app-network
    env_file:
      - database/.env

  httpd:
    container_name: reverse_proxy
    build: ./httpd
    ports:
      - "80:80"
    networks:
      - app-network

volumes:
  my_db_volume:
    driver: local

networks:
  app-network:
```

