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
  backend: #création de l'image de l'api
    container_name: backend #nom du conteneur
    build: ./simple-api # chemin du dossier
    networks: #reseau du conteneur
      - app-network
    depends_on: #Ordre de démarage ici database en premier
      - database

  database:
    container_name: database
    restart: always #si erreur le conteneur restart
    build: ./database
    networks:
      - app-network
    env_file: #fichier pour les variables d'environement
      - database/.env

  httpd:
    container_name: reverse_proxy
    build: ./httpd
    ports: #port du conteneur
      - "80:80"
    networks:
      - app-network

volumes: #pour la persistance des données de database
  my_db_volume:
    driver: local

networks:
  app-network:
```

### 1-5 Document your publication commands and published images in dockerhub.

```docker tag docker-compose_httpd:latest simonmachadocpe/httpd```  pour associer l'image voulue avec le chemin du compte docker<br>
```docker push simonmachadocpe/httpd``` sert à push ce l'image asscoié à ce tag 

## TP2

### 2-1 What are testcontainers?

Ce sont des bibliothèques Java qui permettent d'exécuter un ensemble de conteneurs Docker pendant les tests.

### 2-2 Document your Github Actions configurations.

Notre fichier .github/workflows/main.yaml à la fin du TP2 :

<code>
name: CI devops 2023
on:
  push:
    branches: main
  pull_request: null
jobs:
  test-backend:
    runs-on: ubuntu-22.04
    steps:
      - uses: actions/checkout@v2.5.0
      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: 17
          distribution: adopt
      - name: Build and test with Maven
        run: cd ./docker-compose/simple-api && mvn -B verify sonar:sonar -Dsonar.projectKey=SimonMachadoCPE_devops-sitot -Dsonar.organization=simonmachadocpe -Dsonar.host.url=https://sonarcloud.io -Dsonar.login=${{ secrets.SONAR_TOKEN }}  --file ./pom.xml

  build-and-push-docker-image:
    needs: test-backend
    runs-on: ubuntu-22.04
    steps:
      - name: Login to DockerHub
        run: docker login -u ${{ secrets.DOCKERHUB_LOGIN }} -p ${{ secrets.DOCKERHUB_TOKEN }}
      - name: Checkout code
        uses: actions/checkout@v2.5.0
      - name: Build image and push backend
        uses: docker/build-push-action@v3
        with:
          context: ./docker-compose/simple-api
          tags: ${{secrets.DOCKERHUB_LOGIN}}/backend:latest
          push: ${{ github.ref == 'refs/heads/main' }}
      - name: Build image and push database
        uses: docker/build-push-action@v3
        with:
          context: ./docker-compose/database
          tags: ${{secrets.DOCKERHUB_LOGIN}}/database:latest
          push: ${{ github.ref == 'refs/heads/main' }}
      - name: Build image and push httpd
        uses: docker/build-push-action@v3
        with:
          context: ./docker-compose/httpd
          tags: ${{secrets.DOCKERHUB_LOGIN}}/httpd:latest
          push: ${{ github.ref == 'refs/heads/main' }}
</code>


### 2-3 Document your quality gate configuration.

## TP3

### 3-1 Document your inventory and base commands
```
all:
 vars:
   ansible_user: centos # Utilisateur ansible
   ansible_ssh_private_key_file: /home/tp/TP3/id_rsa #chemin vers la clé Privée rsa
 children:
   prod:
     hosts: simon.machado.takima.cloud # Nom de domaine 
```

<code> ansible all -i inventories/setup.yml -m ping</code> Commande pour tester l'inventory avec un ping : réponse pong valide<br>

### 3-2 Document your playbook

Playbook contenant l'instalation de docker sans appel de role

```
- hosts: all
  gather_facts: false #pas de lancement du module de paramétrage des hotes par défaut
  become: yes #pour lancer ansible depuis un utilisateur root

# Install Docker
  tasks: # taches éxécuté au lancement
  - name: Clean packages
    command:
      cmd: dnf clean -y packages #supprime les packets dnf

  - name: Install device-mapper-persistent-data #instalation de module
    dnf:
      name: device-mapper-persistent-data
      state: latest

  - name: Install lvm2  #instalation de module
    dnf:
      name: lvm2
      state: latest

  - name: add repo docker #création d'un repository docker
    command:
      cmd: sudo dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo

  - name: Install #Configuration Docker dnf
    dnf:
      name: docker-ce
      state: present

  - name: install python3 #instalation de python
    dnf:
      name: python3

  - name: Pip install #instalation de Docker avec python
    pip:
      name: docker

  - name: Make sure Docker is running #Test de la configuration docker
    service: name=docker state=started
    tags: docker
```

### 3-3 Document your docker_container tasks configuration.

Playbook qui va appeler chaque role dans l'ordre:
```
- hosts: all
  gather_facts: false
  become: yes

# Install Docker
  roles:
    - docker
    - network
    - database
    - app
    - proxy
```
Le premier role va instaler docker (voir question précédente)
Le deuxième role crée le réseau pour les prochains conteneur.
```
- name: Create a network
  docker_network:
      name: app-network
```
Le role database va lancer le conteneur postgre contenu sur docker hub



