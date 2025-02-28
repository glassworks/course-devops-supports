# Tests e2e

## Tests end-to-end

Idéalement on aura la possibilité de tester notre API, et pas juste des modules au sein de notre API.

C'est-à-dire, on envoie des requêtes HTTP et on évalue les réponses.

Cela veut dire qu'il faut lancer notre serveur et créer de requêtes HTTPS et recevoir des réponses. Comment faire ?

Grâce à une conception intelligente, nous avons créé deux fonctions `StartServer` et `StopServer` dans `src/servce_manager.ts` qui nous permettent de démarrer et d'arrêter le serveur.

Avec cette structure on peut facilement lancer un serveur à partir de nos tests.

## Outil pour lancer le serveur

Nous créons un outil qui permet de lancer et arrêter le serveur dans `test/utility/TestServer.ts` :

```ts
import { Server } from 'http';
import { StartServer, StopServer } from '../../src/server_manager';

export class TestServer {

  private static _server: Server|undefined;

  public static async Start() {    
    if (process.env.START_HOST) {
      TestServer._server = await StartServer();
    }
  }

  public static async Stop() {
    await StopServer(TestServer._server);
  }

}
```

Ce script lance simplement le serveur et lui garde sa référence.

Vous remarquerez la variable `START_HOST`. En effet, nous allons lancer le serveur seulement si cette variable est présente. Pourquoi ?

* Pour nos tests locaux, nous allons affecter une valeur à `START_HOST` pour lancer un serveur.
* A terme, nous allons compiler un _artifact_ de notre API en forme de container Docker, qui sera lancé tout seul. On aimerait lancer nos tests e2e qui vont plutôt envoyer les requêtes à ce container. Dans ce cas, pas besoin de lancer notre serveur en local.

Du coup, nous allons ajouter quelques valeurs à notre fichier `test/.env.test` :

```sh
# API 
PORT=21011
API_HOST=http://localhost:21011

# TESTING
START_HOST=true
```

Cela pour indiquer qu'il faut lancer le serveur. Aussi, nous précisons le port d'écoute du serveur, ainsi que l'adresse HTTP où on peut trouver notre API.

### Le test e2e

Ensuite, dans un test e2e pour un utilisateur `/test/e2e/suites/User.test.ts`, on peut le lancer et fermer dans les _hooks_ `before` et `after` :

```ts
import { DB } from '@orm/DB';
import axios from 'axios';
import chai from 'chai';
import chaiAsPromised from 'chai-as-promised';
import { describe } from 'mocha';
import { RootDB } from '../../utility/RootDB';
import { TestServer } from '../../utility/TestServer';

chai.use(chaiAsPromised);

describe("User CRUD", function () {
  
  before(async function() {
    // Vider la base de données de test
    await RootDB.Reset();
    // Lancer le serveur
    await TestServer.Start();
  });

  after(async function() {
    // Forcer la fermeture de la base de données
    await DB.Close();
    // Arreter le serveur
    await TestServer.Stop();    
  });

  it("Create a new user", async function () {
    const result = await axios.put(process.env.API_HOST + '/user', 
      {
        familyName: "Glass",
        givenName: "Kevin",
        email: "kevin@nguni.fr",
        balance: 0
      }, 
      {
        headers: {
          Authorization: "Bearer INSERT HERE"
        }
      }
    );

    chai.expect(result.status).to.equal(200);
    chai.expect(result.data.id).to.equal(1);    
  });

  it("Create the same user twice returns an error", async function () {

    const response = await axios.put(process.env.API_HOST + '/user', 
      {
        familyName: "Glass",
        givenName: "Kevin",
        email: "kevin@nguni.fr",
        balance: 0
      }, 
      {        
        headers: {
          Authorization: "Bearer INSERT HERE"
        },
        validateStatus: (status) => { return true }
      }
    );

    chai.expect(response.status).to.equal(400);
    chai.expect(response.data.structured).to.equal('sql/failed');
        

  });
});
```

À noter, dans chaque test, on utilise la librairie `axios` afin de créer et envoyer des requêtes HTTP. Il faut installer le package :

```sh
npm install --save-dev axios
```

Enfin, pour simplifier cette démo, nous avons désactivé la sécurité sur la route `/user`. Dans la réalité, votre test devrait d'abord se connecter avec un utilisateur que vous avez défini dans vos données de départ, puis effectuer le test e2e.  Dans `src/controllers/UserController.ts` :

```ts
// @Security('jwt')  
```


### Lancer nos tests e2e

Avant de lancer le test e2e, nous allons ajouter 2 scripts à notre `package.json` :

```json
 "scripts": {
    ...
    "e2e": "env-cmd -f ./test/.env.test npm run e2e-no-env",
    "e2e-no-env": "mocha -r ts-node/register \"test/e2e/suites/**/*.test.ts\""

```

Comme avant, on ajoute une qui utilise `env-cmd` pour charger les variables d'environnement et un autre sans.

Vous pouvez maintenant lancer les tests e2e:

```sh
npm run e2e
```

Avec le résultat :

```
  User CRUD
info: API Listening on port 21011 {"tag":"exec"}
2022-09-29T16:22:36.355Z 200 POST /auth/user 8 10.573
    ✔ Create a new user (68ms)
error: Duplicate entry 'kevin@nguni.fr' for key 'email' {"details":{"code":400,"details":{"sqlCode":"ER_DUP_ENTRY","sqlState":"23000"},"message":"Duplicate entry 'kevin@nguni.fr' for key 'email'","path":"/auth/user","structured":"sql/failed"},"tag":"exec"}
2022-09-29T16:22:36.366Z 400 POST /auth/user 175 3.291
    ✔ Create the same user twice returns an error


  2 passing (420ms)
```

Notez comment on a testé une réponse négative, et qu'un code 400 est retourné.

## La suite

Nos tests sont bien en place !

On est prêt à avancer dans la configuration de **Continuous Integration**.

