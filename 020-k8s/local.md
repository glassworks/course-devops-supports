# Projet en local

Nous allons travailler avec un projet NodeJS, contenant la base d'un API. Vous trouverez le projet ici :

[devops-sample-main.zip](https://dev.glassworks.tech/courses/devops/devops-sample/-/archive/main/devops-sample-main.zip)

Téléchargez, décompressez, et ouvrez le projet dans VSCode, et relancez votre projet dans un Dev Container (avec Docker). Suivez les instructions dans le `README.md` afin de lancer une version fonctionnelle de l'api.

Lancez votre api en local :

```sh
npm install

npm run server
```

Vous saurez si tout fonctionne correctement si vous arrivez à consulter le chemin d'information à [http://localhost:5055/info](http://localhost:5055/info) dans un navigateur web.

```sh
curl http://127.0.0.1:5050/info

# Resultat :
{"title":"DevOps Code Samples API","host":"4c320d7f5a06","platform":"linux","type":"Linux"}
```

Jetez un coup d'œil au fichier docker-compose.dev.yml. Il s'agit de l'environnement de développement qui crée deux conteneurs, un pour VSCode (basé sur l'image node:20), et un autre pour faire tourner une base de données MariaDB.

```yml
services:
  vscode_devops_api:
    image: rg.fr-par.scw.cloud/devops-code-samples-vscode/vscode_devops:2.0.1
    command: /bin/bash -c "while sleep 1000; do :; done"
    working_dir: /home/dev
    networks:
      - api-devops-network
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
      - ./.data:/var/lib/mysql
    networks:
      - api-devops-network

networks:
  api-devops-network:
    driver: bridge
    name: api-devops-network

```

Nous allons construire une image Docker similaire à `vscode_devops_api` que nous allons déployer dans un cluster Kubernetes ! 

{% hint style="success" %}

Si vous avez suivi le [cours API](https://docs.glassworks.tech/api/mise-en-production/001-docker-image), vous avez déployé manuellement votre application en local, en mettant en marche un Container Docker.

{% endhint %}