# Projet en local

Nous allons travailler avec un projet NodeJS, contenant la base d'un API. Vous trouverez le projet ici :

[devops-sample-main.zip](https://dev.glassworks.tech:18081/courses/devops/devops-sample/-/archive/main/devops-sample-main.zip)

Téléchargez, décompressez, et ouvrez le projet dans VSCode, et relancez votre projet dans un Dev Container (avec Docker). Suivez les instructions dans le `README.md` afin de lancer une version fonctionnelle de l'api.

Lancez votre api en local :

```sh
npm install

npm run server
```

If all goes well you can query the endpoint /info on this api (use Postman or curl) :

```sh
curl http://127.0.0.1:5050/info

# Resultat :
{"title":"DevOps Code Samples API","host":"4c320d7f5a06","platform":"linux","type":"Linux"}
```

Jetez un coup d'œil au fichier docker-compose.dev.yml. Il s'agit de l'environnement de développement qui crée deux conteneurs, un pour VSCode (basé sur l'image node:18), et un autre pour faire tourner une base de données MariaDB.

```yml
version: '3.9'

services:
  vscode_api:
    image: rg.fr-par.scw.cloud/devops-code-samples-vscode/vscode_api:1.0.0
    command: /bin/bash -c "while sleep 1000; do :; done"
    working_dir: /home/dev
    networks:
      - api-network
    volumes:
      - ./:/home/dev:cached
    labels:
      api_logging: "true"      
    
  dbms:
    image: mariadb
    restart: always
    ports:
      - "3379:3306"
    environment: 
      - MYSQL_ALLOW_EMPTY_PASSWORD=false
      - MYSQL_ROOT_PASSWORD=rootpassword
    command: [
      "--character-set-server=utf8mb4",
      "--collation-server=utf8mb4_unicode_ci",
    ]
    volumes:
      - ./dbms/dbms-data:/var/lib/mysql
      - ./dbms/mariadb.cnf:/etc/mysql/mariadb.cnf
    networks:
      - api-network

networks:
  api-network:
    driver: bridge
    name: api-network

```

Nous allons construire une image Docker similaire à `vscode_api` que nous allons déployer dans un cluster Kubernetes ! 

{% hint style="success" %}

Si vous avez suivi le [cours API](https://docs.glassworks.tech/api/mise-en-production/005-git), vous avez déployé manuellement votre application sur un serveur, en téléchargeant votre source sur Git, puis en vous connectant à votre serveur, en construisant et en déployant un conteneur Docker sur le serveur.

C'était un peu lent et répétitif. En utilisant Docker et Kubernetes, nous pouvons automatiser ce processus !

{% endhint %}