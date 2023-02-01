# devops-sitot

## TP 1

### 1-1 Document your database container essentials: commands and Dockerfile.
<code>Docker pull postgre</code> pour récupérer l'image<br>
<code>Docker build  -t some-postgres .</code> pour construire l'image avec le DockerFile contenu dans le répertoire <br>

Dockerfile :

<code>FROM postgres:14.1-alpine
ENV POSTGRES_DB=db \
    POSTGRES_USER=sitot \
    POSTGRES_PASSWORD=admin
COPY 01-CreateScheme.sql /docker-entrypoint-initdb.d
COPY 02-InsertData.sql /docker-entrypoint-initdb.d</code>

Dans ce Dockerfile on renseigne différentes variables d'environements utilisé pour se connecter, on copie aussi des scripts SQL dans l'image qui vont être exécuter au démarage du container<br>
<code>docker run --network app-network -d --name some-postgres postgres-sitot</code> pour récupérer l'image

### 1-2 Why do we need a multistage build? And explain each step of this dockerfile.
