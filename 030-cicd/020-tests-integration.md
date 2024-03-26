# Tests d'intégration

Les tests unitaires sont bien, mais il est assez rare que les modules de notre plateforme fonctionnent dans une vide.

La plupart du temps, il y a une interaction avec, au moins, une base de données.

Comment on pourrait tester notre code en sachant qu'il faudrait lancer une appli ou un module externe, même avant de tourner nos tests ?

## Le cas de la base de données

La base de données présente un challenge pour nos tests automatiques.

On pourrait juste tester contre notre base de développement, mais il faut que les tests soient répétables. Si on modifie le schéma en dev, ou on ajoute des données supplémentaires, on risque de casser nos tests.

Idéalement, on utilise une base de données _uniquement dédiée à nos tests_ :

* Avant de lancer nos tests, on supprime l'ancienne base (si elle existe)
* On recrée le schéma
* On crée un utilisateur de test (qui aura les mêmes droits que notre API ou module en question)
* On préremplit la base avec des données (qu'on appelle les données _seed_)

Cette procédure assure que nos tests restent répétables.

{% hint style="success" %}
Il y a encore un avantage : dans l'esprit de CI/CD, cette procédure pourrait être répétée n'importe où (y compris sur notre serveur de développement)
{% endhint %}

## Docker au secours

Docker est donc idéal pour des tests d'intégration. Avec un `docker-compose.yml` bien écrit, on pourrait créer sans effort l'environnement de dépendances nécessaires pour nos tests (par exemple, lancer un MariaDB, Redis, ... ) puis lancer nos tests automatiques.

Pour notre environnement de développement, on a déjà une instance de MariaDB qui est créée. Super ! Quand on lance notre API en développement, on parle par défaut avec le service `dbms` et la database qui s'appelle `school`.

Nous n'avons qu'à ajouter une deuxième **database**, qu'on appellera `school_test`. Nous allons interagir avec cette base via l'utilisateur `api-test`.

Dans l'esprit de l'environnement de développement, nous allons créer la **database** et l'utilisateur dans un fichier `dbms/ddl/init-test.sql`

```sql
/* On supprime notre base pour chaque test, afin de recommoncer à zero */
drop database IF EXISTS school_test;

/* On recrée la base */
create database IF NOT EXISTS school_test;

/* Créer l'utilisateur API */
create user IF NOT EXISTS 'api-test'@'%.%.%.%' identified by 'testpassword';
grant select, update, insert, delete on school_test.* to 'api-test'@'%.%.%.%';
flush privileges;
```

{% hint style="success" %}
Jusqu'au présent, nous avons utilisé **`use school;`** dans le DDL. Si on veut réutiliser le même DDL pour nos tests, **on sera obligé d'enlever cette ligne du DD**L.

Pour mettre à jour le schéma de la base de données, nous serions obligés de désormais préciser le nom de la **database** sur la ligne de commande mycli : `mycli -h dbms -u root school < ./dbms/ddl/ddl.sql`
{% endhint %}

## Outil pour réinitialiser notre base de données

On aura besoin d'un outil qui permet de remettre à zéro notre base de données (récréer la base `school_test` et recréer son schéma).

Cette opération est en dehors de l'utilisation de notre API de base, parce qu'il y aura des opérations normalement interdites :

* `drop database`
* `create table`
* etc.

Pour nos tests uniquement, nous allons se connecter d'abord en tant que l'utilisateur `root`, effectuer ces opérations, puis laisser l'utilisateur de l'API reprendre la main.

Pour cela, j'ai créé une classe utilitaire qui s'appelle `test/utility/RootDB.ts`. Ce fichier est dans le dossier `test` pour ne pas l'inclure lors de notre build en production.

```ts
import { readFile } from 'fs/promises';
import mysql from 'mysql2/promise';
import { PoolOptions } from 'mysql2/typings/mysql';
import { join } from 'path';

/** Class utilitaire pour réinitialiser la base de données 
 * - On DROP la base existant, si elle existe, 
 * - on en crée une nouvelle, 
 * - on importe le DDL
 * - (optionnelle) on fait executer les instructions SEED (remplir la base avec les données de test)
*/
export class RootDB {
  static async Reset() {

    const database = process.env.DB_DATABASE || "school_test";

    const config: PoolOptions = {         
      host: process.env.DB_HOST || "dbms",
      user: process.env.DB_ROOT_USER || "root",      
      password: process.env.DB_ROOT_PASSWORD || "rootpassword",
      multipleStatements: true
    };
    const POOL = mysql.createPool(config);

    const setup = await readFile(join('dbms', 'ddl', 'init-test.sql'), { encoding: 'utf-8'});   
    console.log(config);
    console.log(setup); 
    await POOL.query(setup);

    const ddl = await readFile(join('dbms', 'ddl', 'ddl.sql'), { encoding: 'utf-8'});
    await POOL.query(`use ${database}; ${ddl}`);

    await POOL.end();
  }
}
```

## Test d'intégration

Nous allons utiliser une librairie de plus, `chai-as-promised` qui permet d'exprimer nos assertions qui concernent des Promises (des opérations `async`).

```bash
npm install --save-dev chai-as-promised @types/chai-as-promised
```

On pourrait, par exemple, tester une opération CRUD pour l'ajout d'un utilisateur (`test/integration/suites/User.integration.ts`)

```ts
import chai, { expect } from 'chai';
import chaiAsPromised from 'chai-as-promised';
import { describe } from 'mocha';
import { RootDB } from '../utility/RootDB';
import { DB } from '../../src/utility/DB';
import { UserController } from '../../src/routes/UserController';

chai.use(chaiAsPromised);

describe("User CRUD", function () {
  
  before(async function() {
    // Vider la base de données de test
    await RootDB.Reset();
  });

  after(async function() {
    // Forcer la fermeture de la base de données
    await DB.Close();
  });

  it("Create a new user", async function () {
    const user = new UserController();
    const result = await user.createUser({
      familyName: "Glass",
      givenName: "Kevin",
      email: "kevin@nguni.fr",
      balance: 0
    });

    expect(result.id).to.equal(1);
  });

  it("Create the same user twice throws an exception", async function () {
    const user = new UserController();

    await expect(user.createUser({
      familyName: "Glass",
      givenName: "Kevin",
      email: "kevin@nguni.fr",
      balance: 0
    })).to.be.rejected;
      
  });

});
```

Note bien l'utilisation du _hook_ `before` et `after`. Ce sont les fonctions appelées avant tous les tests de ce fichier et après tous les tests. Cela permet d'initialiser la base de données, et aussi fermer la connexion à la fin de tous les tests.

&#x20;Il faut donc ajouter la fonction `Close()` à la classe `src/utility/DB.ts`:

<pre class="language-typescript"><code class="lang-typescript">export class DB {
  private static POOL: Pool|undefined;  // Ajouter |undefined
  ...
<strong>  static async Close() {    
</strong>    if (this.POOL) {      
      await this.POOL.end();      
      this.POOL = undefined;    
    }
  }
}
</code></pre>

## Lancer les test d'intégration

Il faut maintenant lancer nos tests. Par contre, on aura besoin de bien préciser les valeurs pour nos variables d'environnement. Souvenez qu'on utilise au moins :

* `DB_HOST` : normalement `dbms` (selon notre `docker-compose.yml`)
* `DB_DATABASE`: le nom de la base à utiliser. Pour le dev, c'est `school`, mais pour nos tests, on va plutôt utiliser `school_test`
* `DB_USER`: le nom d'utilisateur
* `DB_PASSWORD`: le mot de passe
* `DB_ROOT_USER`: le nom d'utilisateur `root`
* `DB_ROOT_PASSWORD` : le mot de passe de l'utilisateur `root`

On devrait donc fournir un `.env` qui va fixer toutes ses variables uniquement pour nos tests.

Moi, j'ai créé un fichier `test/.env.test` qui reprend tous les variables nécessaires pour notre base de test :

```bash
# DATABASE
DB_HOST=dbms
DB_USER=api-test
DB_PASSWORD=testpassword
DB_DATABASE=school_test

DB_ROOT_USER=root
DB_ROOT_PASSWORD=rootpassword
```

Ensuite, nous créons des scripts dans `package.json` pour lancer nos tests d'intégration :

```json
  "scripts": {
    "integration": "env-cmd -f ./test/.env.test npm run integration-no-env",
    "integration-no-env": "mocha -r ts-node/register \"test/integration/suites/**/*.test.ts\"",
  },
```

Notez qu'on a crée 2 scripts :

* `integration-no-env`: qui, comme `unit` lance mocha normalement
* `integration`: qui va commencer par charger les variables d'environnement de `./test/.env.test` avant de lancer le script `integration-no-env`

On sépare ses deux scripts parce qu'à terme, on va pouvoir préciser ces variables d'environnement dans un fichier externe (un `docker-composer.yml` par exemple).

Pour lancer le test en local, on va devoir d'abord installer le package `env-cmd`:

```bash
npm install --save-dev env-cmd
```

On est enfin prêt à lancer notre test d'intégration :

```sh
npm run integration
```

... qui donnera le résultat suivant :

```
  User CRUD
    ✔ Create a new user
    ✔ Create the same user twice throws an exception


  2 passing (191ms)
```

## Considérations

Les tests d'intégration, surtout avec une base de données, peuvent-être assez compliqué à mettre en place :

* Qu'elles sont les données à charger (préconditions) avant l'exécution de mon test ? Parfois, elles en sont nombreuses. Pour l'exemple de publicité, il faut d'abord un annonceur, un éditeur, un utilisateur, une publicité. Il faut créer toutes ces données, et les importer dans votre base avant de lancer le test. Ceci pourrait être :
  * dans les scripts d'initialisation par exemple (`before` hook de mocha)
  * via des modules utilitaires qui permettent de créer tout le scenario
  * une combinaison des deux
* Occasionnellement, on aimerait interroger directement la base de données pour valider que les bonnes données y sont mises. Il faudrait peut-être ajouter à la classe `RootDB.ts` des fonctions utiles pour ce faire.
* Attention au _TEMPS_ et aux _DATES_ ! Si votre application utilise la notion de temps, il faut bien concevoir vos tests pour se passer à un moment fixe, sinon vos tests ne fonctionneront plus dans le futur. En revanche, cela veut dire qu'il y ait la possibilité de paramétrer la date/temps de votre plateforme de façon globale.
* Performance : attention à ne pas importer toute une base de production avant chaque test. On ne veut pas que les tests soient trop longs !

## Code coverage

On aimerait savoir si on a testé _toutes_ les lignes de code dans notre projet.

Et, si on oublie une condition particulière, et on n'a pas un test pour cela ?

Heureusement il y a des outils qui permettent de nous indiquer si nos tests ont bien couvert toutes les différentes branches possibles de notre projet.

Nous allons utiliser le package `istanbul` (ou `nyc`) :

```
npm install --save-dev nyc source-map-support
```

Nous allons mettre à jour notre `package.json` afin d'invoquer cet outil et le paramétrer :

```json
 "scripts": {
    ...
    "unit": "nyc --report-dir ./coverage/unit mocha -r ts-node/register -r source-map-support/register --recursive \"test/unit/suites/**/*.test.ts\"",
    "integration": "env-cmd -f ./test/.env.test npm run integration-no-env",
    "integration-no-env": "nyc --report-dir ./coverage/integration mocha -r ts-node/register -r source-map-support/register --recursive \"test/integration/suites/**/*.test.ts\""
  },
```

À noter, nous avons ajouté le prefixe `nyc --report-dir ./coverage/[DOSSIER]` ainsi que l'option `-r source-map-support/register --recursive` à nos 2 lignes de test.

A la fin du fichier `package.json`, on ajoute une section dédiée à `nyc` :

```json
  "nyc": {
    "extension": [
      ".ts",
      ".tsx"
    ],
    "exclude": [
      "**/*.d.ts"
    ],    
    "all": false,
    "reporter": ["text", "text-summary", "cobertura"]
  }
```

On précise de regarder uniquement les fichiers `.ts`, et de nous générer 3 types de rapport :

* `text`: En texte pour chaque fichier
* `text-summary`: Un résumé de tous les tests
* `cobertura`: Un rapport en XML qu'on va utiliser plus tard pour nos processus de CI/CD

Si on relance `npm run unit` on aura le résultat :

```sh
  AdView
    ✔ One ad view should debit publisher and user, and credit advertiser
    ✔ Should throw an exception if the advertiser does not have enough credit


  2 passing (7ms)

------------------|---------|----------|---------|---------|-------------------
File              | % Stmts | % Branch | % Funcs | % Lines | Uncovered Line #s 
------------------|---------|----------|---------|---------|-------------------
All files         |   66.66 |    66.66 |   35.71 |   65.62 |                   
 Business/AdViews |     100 |      100 |     100 |     100 |                   
  AdView.ts       |     100 |      100 |     100 |     100 |                   
 Errors           |      52 |       50 |      25 |   47.61 |                   
  ApiError.ts     |   36.84 |        0 |   18.18 |   26.66 | 15-30,38-58       
  ErrorCode.ts    |     100 |      100 |     100 |     100 |                   
------------------|---------|----------|---------|---------|-------------------
```

On voit qu'il y a des fichiers dont on n'a pas forcément touché à toutes les lignes de code dans nos tests.

Essayez avec `npm run integration`.

Est-ce que vous avez remarqué qu'on n'a pas forcément testé les comportements de l'API ? C'est-à-dire, tester qu'on récupère les bons codes HTTP dans nos réponses, etc. Nous nous en occupons dans l'étape suivante !



