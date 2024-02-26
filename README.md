# Dockerisez votre application Node.js pour la production

![docker Node](/images/node-docker.png)

## Objectif

Le but de cet travail est de procéder à l'intégration d'une application **Node.js** dans un conteneur Docker exécuté avec **nginx** et **supervisor**.

**Dockeriser** une application fait référence à son intégration dans une image Docker pour qu'elle s'exécute dans un ou plusieurs conteneurs. Ces conteneurs représentent des environnements isolés qui fournissent tout le nécessaire pour exécuter l'application.

Dans ce travail, nous allons:

* Ecrire un **Dockerfile**,
* Créer un environnement de dossier Docker,
* Créer et exécuter une **image Docker**.

### Conditions préalables
* Fresh Ubuntu 20.04 server
* Docker
* NodeJs

### Parie 1: 📌 Créer un dossier nommé **docker** contenant les éléments suivants

![Docker Folder](/images/docker-folder.png)

Sans inquiétude aucune, nous couvrirons le contenu de chaque fichier.

### Parie 2: 📌 Créer un Dockerfile

Créez votre fichier **Dockerfile**

```
touch Dockerfile
```
* La première chose que nous devons faire est de définir à partir de quelle image nous voulons construire

```
FROM image node:14-alpine3.15
```
* Installez Nginx et Supervisor

```
RUN apk update
RUN apk add nginx
RUN apk add supervisor
```
* Remplacez le **nginx** & **nginx.conf** par défaut par le vôtre :

```
RUN rm -f /etc/nginx/http.d/default.conf
ADD ./docker/nginx/http.d/default.conf /etc/nginx/http.d/default.conf
```
* Votre fichier **default.conf** devrait ressembler à ceci

```
server {
  listen 80;
  location / {
	proxy_pass http://localhost:"your_node_app_port";
	}
}
```
N'oubliez pas de remplacer « your_node_app_port » par votre vrai !!

* Copiez les fichiers de configuration de votre supervisor

```
COPY ./docker/supervisord.conf /etc/supervisor/supervisord.conf
COPY ./docker/supervisor.conf /etc/supervisor/conf.d/supervisor.conf
```
* Le fichier **superviserd.conf** est celui qui permet de démarrer le superviseur dans votre conteneur :

```
[supervisord]
nodaemon=true

[include]
files = /etc/supervisor/conf.d/*.conf
```
* Le fichier **supervisor.conf** est celui qui permet de démarrer **npm** & **nginx**.

```
[program:node]
directory=/home/www/node/
command=npm start
autostart=true
autorestart=true
stderr_logfile=/var/log/supervisor/node.err.log
stdout_logfile=/var/log/supervisor/node.out.log

[program:nginx]
command=nginx -g "daemon off;"
autostart=true
autorestart=true
stderr_logfile=/var/log/supervisor/nginx.err.log
stdout_logfile=/var/log/supervisor/nginx.out.log
```
* Créez votre répertoire de travail et votre répertoire de journaux 

```
RUN mkdir -p /home/www/node/node_modules && chown -R node:node /home/www/node
RUN mkdir -p /var/log/supervisor && chown -R node:node /var/log/supervisor
```
sachez qu'après avoir défini l'environnement de l'application, explorons le répertoire du node:

```
WORKDIR /home/www/node
COPY package*.json ./
RUN npm install
RUN npm ci --only=production
COPY --chown=node:node . ./
EXPOSE "YOUR_NODE_APP_PORT"
```
Les lignes ci-dessus installent **npm** et copient votre code source dans le répertoire de travail du conteneur.

Encore une fois, n'oubliez pas de remplacer « your_node_app_port » par votre vrai port!!

Enfin, lancez la commande qui fera toute la magie à votre place:

```
CMD ["/usr/bin/supervisord", "-n", "-c", "/etc/supervisor/supervisord.conf"]
```
Cette ligne démarrera le supervisor.

* Votre Dockerfile devrait maintenant ressembler à ceci:

```
FROM node:14-alpine3.15


#INATALL NGINX AND SUPERVISOR

RUN apk update

RUN apk add nginx

RUN apk add supervisor

#REPPLACE NGINX DEFAULT WITH YOUR CODE

RUN rm -f /etc/nginx/http.d/default.conf

ADD ./docker/nginx/http.d/default.conf /etc/nginx/http.d/default.conf

#COPY YOUR SUPERVISOR CONFIG FILES INSIDE SUPERVISOR FOLDER

COPY ./docker/supervisord.conf /etc/supervisor/supervisord.conf

COPY ./docker/supervisor.conf /etc/supervisor/conf.d/supervisor.conf

#MAKE WORKING DIRECTORY AND LOGS DIRECTORY

RUN mkdir -p /home/www/node/node_modules && chown -R node:node /home/www/node

RUN mkdir -p /var/log/supervisor && chown -R node:node /var/log/supervisor

#INSTALL AND RUN NPM 

WORKDIR /home/www/node

COPY package*.json ./

RUN npm install

RUN npm ci --only=production

COPY --chown=node:node . ./

EXPOSE "YOUR_NODE_APP_PORT"

CMD ["/usr/bin/supervisord", "-n", "-c", "/etc/supervisor/supervisord.conf"]
```

### Parie 3: 📌 Build des fichiers applicatifs

Maintenant que nous avons à la fois le dossier **docker** et le **Dockerfile** dans à la racine de notre application de node, créons l'image Docker :

```
docker build -t node_app_image .
```

### Parie 4: 📌 Lancer le container

```
docker run --name node_app -p 80:80 node_app_image
```
Si tout va bien, vous verrez cette sortie :

![Docker Folder](/images/docker-folder.png)

Sachez ouvrir votre navigateur et tapez :

```
http://localhost : YOUR_NODE_APP_PORT
```