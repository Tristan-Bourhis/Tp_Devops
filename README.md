# TP part 01 - Docker

## 1-1 Document your database container essentials: commands and Dockerfile.

Le Dockerfile fournit à Docker les informations nécessaires à la création à la création d'une image docker.


```
FROM postgres:14.1-alpine

ENV POSTGRES_DB=db \
   POSTGRES_USER=usr \
   POSTGRES_PASSWORD=pwd
```
Ici on définit notre base données postgres ainsi que les variables d'environnement comme notamment l'utilisateur et le mot de passe.

Les deux fichiers sql permettent quant à eux d'initialiser les tables de la base de données et de les remplir.

Ensuite on créer un network sur lequel on pourra faire fonctionner et mettre en relation nos différents containers.
```
docker network create networkName
```

La commande suivante va nous permettre de construire une image Docker.
```docker build -t database ```

On peut ensuite créer un container Docker a partir de cette image grâce à la commande suivante
```
docker run --name database -v database-data:/var/lib/postgresql/data --network=networkName -d database
```
Le paramètre "-v" nous permet de créer une variable d'environnement. Cela permet de stocker les données de manière permanente car un container docker est volatile. C'est à dire qu'une fois éteint, si les données ne sont pas stockées on les perd.

On run ensuite le container adminer qui nous permettra de faire le lien avec la base de données.
```
docker run -p "8090:8080" --net=networkName --name=adminer -d adminer
```
On définit ici un port avec "-p", grâce auquel on pourra accéder à la bdd, et on définit aussi le network en mettant le même que la database afin de les relier.

## 1-2 Why do we need a multistage build? And explain each step of this dockerfile.

Une utilisation multistage offre de nombreux avantages. Notamment pour la maintenance mais aussi pour l'optimisation des ressources.

```# Build
FROM maven:3.6.3-jdk-11 AS myapp-build
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
COPY pom.xml .			# On importe le fichier pom.xml
COPY src ./src			# On importe le dossier src
RUN mvn package -DskipTests

# Run
FROM openjdk:11-jre
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar

ENTRYPOINT java -jar myapp.jar
```

## 1-3 Document docker-compose most important commands. 

```
version: '3.3'
services:
  backend:					# permet de définir un sevice backend avec les paramètres suivant
    container_name: backend			# nom
    build: ./simple-api				# chemin d'accès vers le dossier contenant les fichiers necessaire pour build le container
    networks:					
      - app-network				# le network définit plus bas
    depends_on:
      - database				
    restart: on-failure
    volumes:					# le volume définit plus bas
      - database-data:/var/lib/postgresql/data

  database:					# On définit ici le service de la database
    container_name: database
    restart: always
    build: ./database				# chemin d'accès vers le dossier contenant les fichiers necessaire pour build le container
    networks:
      - app-network				# On utilise le même network afin de pouvoir relier nos containers
    env_file:
      - database/.env

  httpd:					# On définit ici le service du backend
    container_name: backend
    build: ./httpd				# chemin d'accès vers le dossier contenant les fichiers necessaire pour build le container
    ports:
      - "80:80"					# Il s'agit du port qui nous permettra de communiquer avec le container
    networks:
      - app-network				# On utilise le même network afin de pouvoir relier nos containers

volumes:		# Ici on définit les volumes que l'on va utiliser (pour stocker les données)
  database-data:

networks:		# Ici on définit les snetworks que l'on va utiliser
  app-network:
```


## 2-1 What are testcontainers?

Il s'agit de librairies qui permettent de créer des créer des containers docker afin d'effectuer des tests de configuration pour vérifier que le deploiement de l'application soit fonctionnel.

## 2-2 Document your Github Actions configurations.

```
name: CI devops 2022 EPF
on:
  #to begin you want to launch this job in main and develop
  push:
    branches: ["main"]			# On commence par définir la branche sur laquelle on travaille, ici la branche main
  pull_request:
    branches: ["main"]
    

jobs:					# On définit ensuite ici les jobs, donc les différentes tâches que l'on va tester
  test-simple-api: 
    runs-on: ubuntu-22.04		# Il faut commencer par définir un système d'exploitation sur lequel effectuer les tests
    steps:
      - uses: actions/checkout@v2.3.3		# Grâce à cette commande, on check le code pour savoir s'il est correct.

      - name: Set up JDK 11			
        uses: actions/setup-java@v3		# Ici on va effectuer des tests sur java
        with: 					# Pour cela il faut définir les paramètres relatifs comme la version et la distribution.
          java-version: 11
          java-package: jdk
          distribution: 'zulu'

      - name: Build and test with Maven				# Pour finir on va build l'application avec la commande suivante
        run: mvn clean verify --file simple-api/pom.xml		# pour vérifier la configuration de maven en renvoyant vers le pom.xml du projet
```

## 2-3 Document your quality gate configuration.

On a obtenu les notation suivante : 
	- Reliability : A, 0 bug signalé
	- Maintenaibility : A, avec 23 lignes de codes difficilement maintenable, ce qui correspond à un ratio inférieur à 5%
	- Security : D, avec 2 vulnérabilités dont 2 critiques pouvant être exploité
	- Coverage : 51.9 %, qui correspond au pourcentage de lignes de code testées

## 3-1 Document your inventory and base commands

```
all:
 vars:
   ansible_user: centos				
   ansible_ssh_private_key_file: ../../../../../../../../../../../bin/id_rsa  # le chemin d'accès (sur ma machine personnelle) vers la clé rsa 
 children:
   prod:
     hosts: tristan.bourhis.takima.cloud		# le nom de domaine du server
```

Le but est de pouvoir se connecter a un server distant. La communication se fait via la clé privé stocké dans id_rsa.
La commande suivante nous permet de ping le server pour tester la communication avec celui-ci

```
ansible all -i inventories/setup.yml -m ping
```

## 3-2 Document your playbook & Document your docker_container tasks configuration.


```
- hosts: all
  gather_facts: false
  become: yes

  tasks:
  roles:			# On définit ici les différents roles que l'on souhaite lancer
    - docker			# on renvoie simplement vers les sous dossiers correspondant dans le dossier rôles puis dans test
    - create_network		# par exemple "docker" renvoie vers /roles/docker/tasks/main.yml
    - httpd 			# de même pour les autres tâches
    - database
    - backend
```


### Docker

On va avoir ici toute les commandes nécessaire pour l'installation et l'utilisation de docker sur le server.
```
- name: Clean packages
  command:
    cmd: dnf clean -y packages

- name: Install device-mapper-persistent-data
  dnf:
    name: device-mapper-persistent-data
    state: latest

- name: Install lvm2
  dnf:
    name: lvm2
    state: latest

- name: add repo docker
  command:
    cmd: sudo dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo

- name: Install Docker
  dnf:
    name: docker-ce
    state: present

- name: install python3
  dnf:
    name: python3

- name: Pip install
  pip:
    name: docker

- name: Make sure Docker is running
  service: name=docker state=started
  tags: docker
```

### Create network

Va nous permettre de créer le network pour mettre en relation nos différents container.
```
- name: Create a network		
  docker_network:
    name: app_network
    state: present
```

### httpd

On lance ici un container docker avec les paramètre correspondant.
A savoir une image pour build le container, le network créé juste avant et le port qui permettra de communiquer avec le container.

```
- name: Run HTTPD
  docker_container:
    name: httpd
    image: tristanbourhis/tp_devops:httpd
    networks:
      - name: app_network
    ports:
      - 80:80
```

### database

On lance ici un container docker avec les paramètre correspondant.
A savoir une image pour build le container, le network et les variables d'environnement pour se connecter à la base de donnée POSTGRES.

```
- name: Launch database and connect to network
  docker_container:
    name: database
    image: tristanbourhis/tp_devops:database
    state: started
    env:
      POSTGRES_USER: "usr"
      POSTGRES_PASSWORD: "pwd"
      POSTGRES_DB: "db"

    networks:
      - name: app_network
```

### backend

On lance ici un container docker avec les paramètre correspondant.
A savoir une image pour build le container et le network

```
- name: Launch App/ Backend
  docker_container:
    name: backend
    image: tristanbourhis/tp_devops:simple-api
    networks:
      - name: app_network    
```
