# TP1 Docker : application 3-tiers

Ce dépôt contient une application en trois couches conteneurisée avec Docker :

- **database** : PostgreSQL 17.2, initialisée avec des scripts SQL
- **backend** : API Spring Boot (Java 21) connectée à la base
- **http-server** : Apache httpd configuré en reverse proxy devant le backend

L'ensemble est orchestré avec Docker Compose.

## Structure

```
TP1/
├── database/
│   ├── Dockerfile
│   └── sql/
│       ├── 01-CreateScheme.sql
│       └── 02-InsertData.sql
├── backend/
│   ├── hello/              # Hello World Java (étape Basics)
│   │   ├── Main.java
│   │   └── Dockerfile
│   └── simpleapi/          # API Spring Boot
│       ├── Dockerfile
│       ├── pom.xml
│       └── src/
├── http-server/
│   ├── Dockerfile
│   ├── index.html
│   └── httpd.conf
├── docker-compose.yml
├── .env.sample
└── README.md
```

## Lancer le projet

1. Copier le fichier de variables :
   ```bash
   cp .env.example .env
   ```
2. Lancer la stack :
   ```bash
   docker compose up -d --build
   ```
3. Tester :
   ```
   http://localhost/departments/IRC/students
   ```

---

## 1. Database

### Dockerfile

```dockerfile
FROM postgres:17.2-alpine

ENV POSTGRES_DB= \
    POSTGRES_USER=

COPY ./sql/ /docker-entrypoint-initdb.d/
```

J'ai retiré le mot de passe du Dockerfile : il est passé au lancement avec `-e`.

### Commandes

Création du réseau :
```bash
docker network create app-network
```

Build de l'image :
```bash
docker build -t tp1-database ./database
```

Lancement de la base avec un volume pour persister les données :
```bash
docker run -d --name database --network app-network \
  -e POSTGRES_PASSWORD=pwd \
  -v tp1-db-data:/var/lib/postgresql/data \
  tp1-database
```

Lancement d'Adminer sur le même réseau :
```bash
docker run -d -p "8090:8080" --net=app-network --name=adminer adminer
```

Adminer est accessible sur `http://localhost:8090` (système PostgreSQL, serveur `database`).

### Initialisation

Postgres exécute au premier démarrage tous les scripts présents dans `/docker-entrypoint-initdb.d`, par ordre alphabétique. J'ai donc préfixé mes scripts par `01-` et `02-` pour créer les tables avant d'insérer les données.

Vérification dans les logs :
```bash
docker logs database
```
On y voit l'exécution de `01-CreateScheme.sql` puis `02-InsertData.sql`. Dans Adminer, la table `departments` contient 3 lignes et `students` en contient 4.

### Persistance

J'ai vérifié la persistance en ajoutant un étudiant dans Adminer, en supprimant le conteneur (`docker rm -f database`) puis en le recréant avec le même volume : la ligne ajoutée était toujours présente. Les scripts d'initialisation ne se relancent pas dans ce cas, car le dossier de données n'est plus vide.

### Questions

**1-1 Pourquoi passer les variables d'environnement avec `-e` plutôt que dans le Dockerfile ?**

Une valeur écrite dans le Dockerfile est figée dans l'image. N'importe qui ayant accès à l'image peut la retrouver avec `docker history` ou `docker inspect`, ce qui pose problème pour un mot de passe, surtout si l'image est publiée. Avec `-e`, le secret reste en dehors de l'image et la même image peut être réutilisée dans plusieurs environnements avec des valeurs différentes.

**1-2 Pourquoi attacher un volume au conteneur Postgres ?**

Le système de fichiers d'un conteneur est supprimé en même temps que lui. Sans volume, toutes les données de la base sont perdues à chaque `docker rm`. Le volume stocke les données en dehors du conteneur, ce qui permet de les conserver quand le conteneur est recréé ou mis à jour.

---

## 2. Backend API

### Basics : Hello World

Compilation en ciblant Java 21 pour rester compatible avec le JRE de l'image :
```bash
javac --release 21 Main.java
```

```dockerfile
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY Main.class .
CMD ["java", "Main"]
```

```bash
docker build -t tp1-hello ./backend/hello
docker run --rm --name hello tp1-hello
```

Résultat : `Hello World!`

### Simple API Spring Boot

Projet généré sur Spring Initializr (Maven, Java 21, Spring Boot 3.4.5, dépendance Spring Web), avec un `GreetingController` qui expose l'endpoint `/`.

```dockerfile
# Build stage
FROM eclipse-temurin:21-jdk-alpine AS myapp-build
ENV MYAPP_HOME=/opt/myapp
WORKDIR $MYAPP_HOME
RUN apk add --no-cache maven
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn package -DskipTests

# Run stage
FROM eclipse-temurin:21-jre-alpine
ENV MYAPP_HOME=/opt/myapp
WORKDIR $MYAPP_HOME
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar
ENTRYPOINT ["java", "-jar", "myapp.jar"]
```

```bash
docker build -t tp1-backend ./backend/simpleapi
docker run -d --name backend -p 8080:8080 tp1-backend
curl http://localhost:8080/
```

Résultat : `{"id":1,"content":"Hello, World!"}`

### Questions

**1-4 Pourquoi un build multistage ? Explication du Dockerfile**

Le multistage sépare la compilation de l'exécution. Le JDK et Maven ne servent qu'à construire le jar : ils restent dans le premier stage et ne se retrouvent pas dans l'image finale. L'image finale ne contient que le JRE et le jar, elle est donc plus légère et expose moins d'outils en cas d'attaque. Autre avantage : le build ne dépend plus de ce qui est installé sur ma machine.

Étapes du Dockerfile :

1. `FROM eclipse-temurin:21-jdk-alpine AS myapp-build` : image de build avec le JDK, nommée pour pouvoir y faire référence plus loin.
2. `ENV MYAPP_HOME` et `WORKDIR` : définition et positionnement dans le dossier de travail.
3. `RUN apk add --no-cache maven` : installation de Maven.
4. `COPY pom.xml .` puis `RUN mvn dependency:go-offline` : téléchargement des dépendances. Comme cette étape ne dépend que du `pom.xml`, Docker la garde en cache tant que le pom ne change pas, même si le code source change.
5. `COPY src ./src` : copie du code source.
6. `RUN mvn package -DskipTests` : compilation et création du jar, sans lancer les tests.
7. `FROM eclipse-temurin:21-jre-alpine` : nouvelle image, avec seulement le JRE.
8. `COPY --from=myapp-build ...` : récupération du jar produit dans le premier stage.
9. `ENTRYPOINT ["java", "-jar", "myapp.jar"]` : commande lancée au démarrage du conteneur.

### Backend API connecté à la base

J'ai remplacé `src` et `pom.xml` par ceux du projet `simple-api`. La configuration de la base se trouve dans `src/main/resources/application.yml` :

```yaml
  datasource:
    url: jdbc:postgresql://${DB_HOST}:${DB_PORT}/${POSTGRES_DB}
    username: ${POSTGRES_USER}
    password: ${POSTGRES_PASSWORD}
    driver-class-name: org.postgresql.Driver
```

Les valeurs viennent des variables d'environnement. Le backend accède à la base via le réseau Docker : le nom du service (`database`) sert de nom d'hôte. `localhost` ne fonctionnerait pas, car dans un conteneur il désigne le conteneur lui-même.

Test :
```
http://localhost/departments/IRC/students
```

```json
[
  {
    "id": 1,
    "firstname": "Eli",
    "lastname": "Copter",
    "department": { "id": 1, "name": "IRC" }
  }
]
```

---

## 3. HTTP server

### Image

J'ai choisi `httpd:2.4-alpine`, cohérent avec les autres images basées sur Alpine.

```dockerfile
FROM httpd:2.4-alpine
COPY ./index.html /usr/local/apache2/htdocs/
COPY ./httpd.conf /usr/local/apache2/conf/httpd.conf
```

### Configuration

J'ai récupéré la configuration par défaut depuis un conteneur en marche :
```bash
docker cp httpd:/usr/local/apache2/conf/httpd.conf ./http-server/httpd.conf
```

Puis j'ai ajouté à la fin du fichier la configuration du reverse proxy :
```apache
<VirtualHost *:80>
ProxyPreserveHost On
ProxyPass / http://${BACKEND_HOST}:${BACKEND_PORT}/
ProxyPassReverse / http://${BACKEND_HOST}:${BACKEND_PORT}/
</VirtualHost>
LoadModule proxy_module modules/mod_proxy.so
LoadModule proxy_http_module modules/mod_proxy_http.so
```

- `ProxyPass` transfère les requêtes reçues vers le backend.
- `ProxyPassReverse` réécrit les en-têtes des réponses (redirections) pour qu'elles pointent vers le proxy et non vers le backend.
- `ProxyPreserveHost On` transmet au backend l'en-tête `Host` d'origine.
- Les deux `LoadModule` activent les modules proxy, qui ne sont pas chargés par défaut.

Apache remplace `${BACKEND_HOST}` et `${BACKEND_PORT}` par les variables d'environnement du conteneur au démarrage.

Commandes utiles pour vérifier le conteneur : `docker stats`, `docker inspect httpd`, `docker logs httpd`.

### Questions

**1-5 Pourquoi un reverse proxy ?**

Le reverse proxy sert de point d'entrée unique à l'application. Le backend n'est plus exposé directement sur la machine hôte, seul le proxy l'est. C'est aussi à cet endroit qu'on peut gérer le HTTPS, répartir la charge entre plusieurs instances du backend ou servir un front-end statique, sans modifier le backend.

---

## 4. Docker Compose

### Fichier `.env`

Toutes les variables sont centralisées dans `.env`, qui n'est pas versionné (voir `.gitignore`). Le fichier `.env.esample` sert de modèle.

### Questions

**1-6 Pourquoi docker-compose est-il important ?**

Sans Compose, il faut lancer chaque conteneur à la main avec une longue commande `docker run`, dans le bon ordre, en recréant réseaux et volumes. Compose décrit toute l'application dans un seul fichier versionné : images, variables, réseaux, volumes, dépendances entre services. Une seule commande suffit pour tout lancer, tout arrêter ou tout reconstruire, et n'importe qui peut reproduire exactement le même environnement.

**1-7 Commandes docker-compose principales**

| Commande | Rôle |
|---|---|
| `docker compose up -d --build` | Build les images et lance tous les services en arrière-plan |
| `docker compose up -d --build backend` | Rebuild et relance un seul service |
| `docker compose ps` | Affiche l'état des services |
| `docker compose logs -f <service>` | Affiche les logs d'un service en direct |
| `docker compose stop` / `start` | Arrête / relance les services sans les supprimer |
| `docker compose down` | Supprime les conteneurs et les réseaux |
| `docker compose down -v` | Supprime aussi les volumes, donc les données de la base |

**1-8 Explication du fichier docker-compose**

- **build** : chaque service est construit à partir du Dockerfile de son dossier.
- **environment** : les valeurs sont lues dans `.env`. Chaque service ne reçoit que les variables dont il a besoin : httpd n'a pas accès au mot de passe de la base.
- **ports** : seul httpd expose un port (80) sur la machine hôte. Le backend et la base ne sont accessibles que depuis les réseaux Docker internes.
- **networks** : j'ai séparé deux réseaux. `front-network` relie httpd et le backend, `back-network` relie le backend et la base. Le backend est sur les deux. Ainsi httpd, le seul service exposé, ne peut pas joindre la base directement.
- **volumes** : le volume nommé `db-data` conserve les données de Postgres entre deux redémarrages.
- **healthcheck** : `pg_isready` vérifie que Postgres accepte les connexions. Combiné à `condition: service_healthy`, le backend ne démarre qu'une fois la base réellement prête, et pas seulement quand son conteneur est lancé.
- **depends_on** : définit l'ordre de démarrage (database, puis backend, puis httpd).
- **restart: unless-stopped** : un service qui plante est relancé automatiquement, sauf si je l'ai arrêté volontairement.

---


test