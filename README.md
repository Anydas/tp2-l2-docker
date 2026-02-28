# TP Docker — Bases

Ce TP d'une durée d'environ 3h couvre les bases de Docker : commandes de base, exécution avec `docker run`, et orchestration simple avec `docker compose`.

**Objectifs**
- Comprendre et utiliser les commandes Docker de base (`ps`, `images`, `volume`, `logs`, `stats`).
- Lancer et contrôler des conteneurs avec `docker run` (options courantes : `-d`, `--name`, `-p`, `-v`, `-e`, `--rm`).
- Définir et exécuter un petit stack multi-conteneurs avec `docker compose`.

**Prérequis**
- Machine avec Docker Engine et Docker CLI installés.
- `docker compose` (v2) idéal; sinon `docker-compose` fonctionne.
- Accès terminal et privilèges pour exécuter Docker.
- Utiliser le WSL pour windows ou un poste linux

Pour les pc fac, mettre en place le proxy:
ouvrir docker desktop puis aller dans Settings => ressources => proxy et entrer les valeurs suivantes:

- Dans http => proxy:3128
- Dans https => proxy:3128

Puis appuyer sur appliquer.

**Fichiers fournis**
- `docker-compose.yml` : exemple pour la partie Compose.
- `www/index.html` : contenu web de démonstration.

**Instructions**
1. Cloner / récupérer le dépôt de TP.
2. Suivre les exercices ci-dessous dans un terminal.

## Partie A — Commandes de base

### Exercice A1 — Lister et inspecter

Executer :
```bash
docker run -d --name tp-nginx nginx:alpine
docker ps
docker ps -a
docker images
docker volume ls
```
Questions : différence entre `docker ps` et `docker ps -a` ? Trouver un IMAGE ID.

### Exercice A2 — Statistiques & logs
```bash
docker stats tp-nginx
docker logs tp-nginx
docker stop tp-nginx
docker rm tp-nginx
```

### Exercice A3 — Volumes
```bash
docker volume create tp-data
docker run -d --name tp-busybox -v tp-data:/data busybox sleep 300
docker exec tp-busybox sh -c "echo hello > /data/hello.txt && ls -l /data"
docker rm -f tp-busybox
```
Question : où sont stockés les volumes sur l’hôte ? (voir `docker info`).

## Partie B — `docker run` et exécution (30 min)

### Exercice B1 — Run simple web
```bash
docker run -d --name web-demo -p 8080:80 nginx:alpine
```
Ouvrir `http://localhost:8080` pour vérifier que le serveur web est bien actif(page par defaut de nginx).

### Exercice B2 — Variables d'environnement et volumes

```bash
mkdir -p ~/tp-www
# Création d'un fichier contenant notre variable d'environnement
echo "Hello TP Docker" > ~/tp-www/index.html
# creation du volume contenant notre variable d'environnement à l'aide du -v
docker run -d --name apache-demo -p 8081:80 -v ~/tp-www:/usr/local/apache2/htdocs/ httpd:2.4
```

Nettoyage
```bash
docker rm -f web-demo apache-demo || true
docker network rm tp-net || true
docker volume rm tp-data || true
```

## Partie C — Docker Compose

### Exercice C1 — Commande de base d'un docker-compose

Fichier `docker-compose.yml` fourni (web + redis).
```bash
docker compose up -d
docker compose ps
```
Ouvrir `http://localhost:8082` pour vérifier que le serveur web est bien actif (page par defaut de nginx).

### Exercice C2 — logs
```bash
docker compose logs --tail=50 --follow
docker compose down
```

### Exercice C3 - Création d'un docker compose

Le but de cet exercise est de créer votre premier docker compose. Ce docker compose nous permettra de mettre en place un wordpress.

Dans un premier temps, nous allons définir nos services et nos images. Nous aurons besoin de deux services :
- un service wordpress avec comme image : wordpress:latest
- un service db (notre base de donnée) avec comme image : mysql:8.0

```docker
services:

  wordpress:
    image: wordpress:latest

  db:
    image: mysql:8.0

```

Nous allons maintenant rajouter une politique de redémarrage afin de s'assurer que les conteneurs se relance en cas de crash / probleme. Pour se faire, nous allons rajouter l'instruction restart: always

```docker
services:

  wordpress:
    image: wordpress:latest
    restart: always

  db:
    image: mysql:8.0
    restart: always

```

A l'aide de l'instruction port, nous allons permettre à notre application d'être accessible depuis l'exterieur sur le port 8080

```docker
services:

  wordpress:
    image: wordpress:latest
    restart: always
    ports:
      - 8080:80

  db:
    image: mysql:8.0
    restart: always
```

Wordpress et MySQL ont besoin tous les deux d'information pour s'executer. Grace aux variables d'environnement, nous allons leur donner ces informations. L'instruction environment va nous permettre de set à l'interieur du conteneur un jeu de variable d'environnement

```
services:

  wordpress:
    image: wordpress
    restart: always
    ports:
      - 8080:80
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_USER: exampleuser
      WORDPRESS_DB_PASSWORD: examplepass
      WORDPRESS_DB_NAME: exampledb

  db:
    image: mysql:8.0
    restart: always
    environment:
      MYSQL_DATABASE: exampledb
      MYSQL_USER: exampleuser
      MYSQL_PASSWORD: examplepass
      MYSQL_RANDOM_ROOT_PASSWORD: '1'
```

Il nous reste plus qu'a creer nos volumes de sauvegarde et notre docker compose sera terminé.

Dans un premier temps, au meme niveau que service, nous allons creer nos volumes (un volume db pour notre bdd et un volume wordpress pour wordpress)

```
services:

  wordpress:
    image: wordpress
    restart: always
    ports:
      - 8080:80
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_USER: exampleuser
      WORDPRESS_DB_PASSWORD: examplepass
      WORDPRESS_DB_NAME: exampledb

  db:
    image: mysql:8.0
    restart: always
    environment:
      MYSQL_DATABASE: exampledb
      MYSQL_USER: exampleuser
      MYSQL_PASSWORD: examplepass
      MYSQL_RANDOM_ROOT_PASSWORD: '1'

volumes:
  wordpress:
  db:
```

Dans un second temps, nous allons, dans chaque service, faire le mappage entre notre volume et le repertoire à sauvegarder dans le conteneur grace à l'instruction volumes

```
services:

  wordpress:
    image: wordpress
    restart: always
    ports:
      - 8080:80
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_USER: exampleuser
      WORDPRESS_DB_PASSWORD: examplepass
      WORDPRESS_DB_NAME: exampledb
    volumes:
      - wordpress:/var/www/html

  db:
    image: mysql:8.0
    restart: always
    environment:
      MYSQL_DATABASE: exampledb
      MYSQL_USER: exampleuser
      MYSQL_PASSWORD: examplepass
      MYSQL_RANDOM_ROOT_PASSWORD: '1'
    volumes:
      - db:/var/lib/mysql

volumes:
  wordpress:
  db:
```

La commande suivante va vous permettre de lancer / arreter votre docker compose:

```
docker compose -f lienversdockercompose up/down
```

Nettoyage final
```bash
docker compose down --volumes
docker rm -f $(docker ps -aq) || true
docker rmi $(docker images -q) || true
docker volume prune -f
```

## Annexe

 Résumé des commandes Docker principales
 
 - `docker --version` : afficher la version de Docker.
 - `docker info` : information sur l'installation et le daemon.
 - `docker ps` : lister conteneurs en cours.
 - `docker ps -a` : lister tous les conteneurs (y compris arrêtés).
 - `docker images` : lister les images locales.
 - `docker pull <image>` : télécharger une image depuis le registre.
 - `docker run [OPTIONS] IMAGE [CMD]` : créer et démarrer un conteneur.
	 - Exemples courants :
		 - `docker run -d --name mynginx -p 8080:80 nginx:alpine`
		 - `docker run --rm busybox echo hello`
		 - options utiles : `-d` (détaché), `--name`, `-p host:container`, `-v host:container`, `-e VAR=val`, `--rm`.
 - `docker stop <container>` / `docker start <container>` / `docker restart <container>` : arrêter/démarrer/redémarrer.
 - `docker rm <container>` : supprimer un conteneur (utiliser `-f` pour forcer).
 - `docker rmi <image>` : supprimer une image.
 - `docker logs <container>` : afficher les logs d'un conteneur.
 - `docker exec -it <container> <cmd>` : exécuter une commande dans un conteneur en cours.
 - `docker stats` : afficher l'utilisation ressources en temps réel.
 - `docker inspect <object>` : voir la configuration JSON détaillée d'un conteneur/image/réseau.
 - `docker network ls` / `docker network create|rm <name>` : gérer les réseaux Docker.
 - `docker volume ls` / `docker volume create|rm <name>` : gérer les volumes.
 - `docker system df` : utilisation disque par Docker.
 - `docker system prune` : nettoyer conteneurs arrêtés, réseaux inutilisés, images non référencées (à utiliser avec précaution).
 
 Rappels `docker compose` (v2+)
 - `docker compose up -d` : lancer les services définis dans `docker-compose.yml`.
 - `docker compose ps` : lister les conteneurs du projet Compose.
 - `docker compose logs [service]` : voir les logs.
 - `docker compose down` : arrêter et supprimer les ressources créées par Compose.
 - `docker compose up --scale service=N` : démarrer plusieurs réplicas (selon le driver).
 
 Conseils rapides
 - Préférer `--rm` pour les conteneurs one-shot pour nettoyer automatiquement.
 - Toujours vérifier quels ports sont libres avant d'exposer (`-p`).
 - Nettoyer avec `docker system prune` uniquement si vous comprenez les suppressions.