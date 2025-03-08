# Tests unitaires

## Tests unitaires

Dans la structuration de vos projets, il est toujours intéressant d'organiser son code dans les modules qui traite un et seul sujet ou problème, sans mélanger d'autres objectifs, fonctions, ou dépendances.

Il n'est pas toujours évident _où_ mettre la logique de son code.

Est-ce qu'on la met dans les _handlers_ de notre API directement ? Et, si un jour, on voulait invoquer la fonctionnalité sans passer par une requête HTTP (via un CLI par exemple) ? On sera coincé. Du coup, on pourrait mettre la _logique business_ dans une autre fonction ou classe. Dans cette fonction/classe, est-ce qu'on intègre directement les instructions IO (vers la base de données) ? Ou serait-il mieux d'encore extraire ces fonctionnalités-là, pour rendre notre logique business _indépendant_ de la couche de stockage donnée.

Je répète, il n'est pas toujours évident de décider. L'architecture et la rédaction de son code dépend de plusieurs facteurs dont il faut trouver l'équilibre :

* Le temps et argent disponibles : il serait bien de réfléchir un design pendant 1 semaine avant de se lancer dans le code. Le design sera parfait, mais ça coûtera cher et sera peut-être en retard.
* La complexité finale de votre projet. Un projet bien avec beaucoup d'abstractions devient de plus en plus difficile à comprendre.

L'architecture du logiciel est donc le sujet de diverses études, écoles, normes, frameworks :

* [Model-View-Controller](https://fr.wikipedia.org/wiki/Mod%C3%A8le-vue-contr%C3%B4leur)
* [Hexagonal design](https://fr.wikipedia.org/wiki/Architecture\_hexagonale)
* Broker
* Event-bus
* ...

L'architecture peut être stricte, ou flexible, et en plein d'évolution dans le même projet.

Une technique _concrète_ qui aide à la structuration de son code et utiliser des tests pour décider comment structurer vos modules.

* Est-ce que je pourrais tester clairement et facilement une logique business ?
* Est-ce que mon test est indépendant d'autres facteurs qui n'ont rien avoir avec la logique business (surtout provenant des dépendances externes) ?

## Cas d'étude

Considérez [le code suivant](https://dev.glassworks.tech/courses/devops/devops-sample/-/blob/main/src/business/AdView.ts), pour transferer de l'argent entre un annonceur de publicité et un éditeur lorsqu'une publicité est vue sur Internet : 

```ts
export class AdView {

  constructor(private settings: IAdViewSettings) {
  }
  
  transfer(ad: IAdvert, advertiser: IAdvertiserRO, publisher: IPublisherRO, user: IUserRO) : IAdViewResult { 
    
    const advertiserBalance = advertiser.balance - ad.price;
    if (advertiserBalance < 0) {
      throw new ApiError(ErrorCode.BadRequest, 'advertiser/insufficient-credit', "Not enough balance to show the ad");
    }
    /** Ici on va forcer l'arrondie au 10ème de centime, vers le bas:
     * 1.23 € * 0.75€ = 0.9225 €
     * x 100 = 92.25
     * floor(92.25) = 92
     * 92 / 100 = 0.92 €
     */
    const publisherCredit = Math.floor(ad.price * this.settings.publisherPercentage * 100) / 100.0;

    /** Et ici on va donner le reste à l'utilisateur */
    const userCredit = ad.price - publisherCredit;

    const result: IAdViewResult = {
      updates: {
        advertiser: {
          balance: advertiserBalance
        },
        publisher: {
          balance: publisher.balance + publisherCredit
        },
        user: {
          balance: user.balance + userCredit
        }
      },
      view: {
        advertId: ad.advertId,
        advertiserId: ad.advertiserId,
        publisherId: publisher.publisherId,
        userId: user.userId,
        total: ad.price,
        advertiserDebit: ad.price,
        publisherCredit: publisherCredit,
        userCredit: userCredit
      }
    }

    return result;
  }
}
```

Comme vous voyez, cette classe n'implémente que la transaction entre l'annonceur, l'éditeur, l'utilisateur et la publicité. Vous allez remarquer aussi que :

* Dans la classe, on suppose que les données ont déjà été chargées. Cette classe ne s'en occupe pas
* Dans la classe, on ne s'occupe pas de la sauvegarde des données. En effet, la fonction `transfer` va simplement retourner un objet avec les mises à jour à apporter, puis on laisse un autre module sauvegarder ses données dans la base
* Cette classe ne se préoccupe pas du tout de QUI va l'appeler, ni COMMENT, ni QUAND.
* La classe pourrait fonctionner comme une boîte noire : il y a des entrées bien définies, et des sorties bien définies

Qu'est-ce qu'on vient de faire ?

* On a isolé la logique business
* On a enlevé les dépendances (notamment le stockage, MySQL, etc)

Cette classe devient un cas parfait pour ce qu'on appelle un _test unitaire_ qui a pour objectif de faire le suivant :

{% hint style="info" %}
Un test unitaire a pour objectif de tester toutes les combinaisons de _input_, et mesurer toutes les combinaisons de _output_, pour être sûr que le _output_ soit toujours valable, et consistent.
{% endhint %}

Via un test unitaire, on pourrait :

* Détecter un comportement inattendu si jamais on refait une passe sur l'implémentation (par exemple, pour optimiser)
* Détecter des bogues (le test plante)
* Assurer que les valeurs ne change pas (en quantité, type, format, etc)
* Tester les _edge-case_ : par exemple, utiliser des valeurs non-valides comme _input_, assurer qu'une exception d'un certain type est lancé, par exemple.

## Implémentation des tests

Retournons à notre projet d'exemple, qui vous avez téléchargé pour les chapitres précédentes :

[devops-sample-main.zip](https://dev.glassworks.tech/courses/devops/devops-sample/-/archive/main/devops-sample-main.zip)

Dans ce projet, nous avons ajouté des **tests unitaires** dans le dossier `test/unit/suites`, notamment des tests pour notre classe `Adview.ts`.

Nos tests utilisent des paquets externes :

- [Mocha](https://mochajs.org)
- [Chai](https://www.chaijs.com) 

Puisque ce sont des paquets utilisés uniquement en développement, on les aurait installé avec l'option `--save-dev` :

```sh
npm install --save-dev mocha @types/mocha
npm install --save-dev chai@^4 @types/chai@^4
```

Attention l'option `--save-dev` : on veut utiliser ces packages uniquement en développement et pas en déploiement.

Ensuite, nous avons crée nos premiers tests dans le dossier `test/unit/suites`.

Pourquoi cette organisation ?

* On range tous nos tests dans le répertoire `test` pour pouvoir facilement l'exclure de notre build de production
* On range les tests unitaires à part des autres tests à venir (intégration, e2e)
* On peut ranger encore plus par thème (`suites`)

Regardons [les tests pour AdView.ts](https://dev.glassworks.tech/courses/devops/devops-sample/-/blob/main/test/unit/suites/AdView.test.ts)

Dans `mocha`, on utilise uns structure _BDD_ (_behaviour driven development_), qui veut dire qu'on va préciser un comportement souhaité, puis chaque test va assurer ce comportement.

Dans le fichier, on commence par le mot clé `describe` qui précise le module qu'on est en train de décrire.

Ensuite, avec la fonction `it`, nous spécifions le comportement souhaité :

```ts
describe("AdView", function () {

  it("One ad view should debit publisher and user, and credit advertiser", function () {

    // Implementation du test
```

À un moment, il faut valider que le comportement est validé ou pas. Normalement, nous faisons des _assertions_ dans le négatif : si tout fonctionne bien, la fonction quitte sans erreurs. Si un problème est détecté, on devrait quitter la fonction avec une erreur.

Dans pratiquement tous les langages de programmation, on a la notion _d'assertion_ (ou la fonction `assert`), qui teste une condition, et arrête le processus avec un code d'erreur si la condition ne passe pas.

La librairie `chai` nous propose un nombre de clauses qui implémente la fonction `assert` mais dans une façon plus proche à l'anglais :

```ts
  expect(result).to.not.be.undefined;
  expect(result.view.advertId).to.equal(advert.advertId);
```

Si une condition ne passe pas, le processus (le test) s'arrête avec un code d'erreur.

On pourrait même tester que les exceptions sont bien lancées :

```ts
expect(() => {
  adview.transfer(
    advert,
    advertiser,
    publisher, 
    user
  );
}).to.throw(ApiError).with.property('structured', 'advertiser/insufficient-credit');
```

Pour exécuter notre test, on ajoute un script à [package.json](../package.json) :

```json
  "scripts": {
    ...
    "unit": "mocha -r ts-node/register -r tsconfig-paths/register \"test/unit/suites/**/*.test.ts\""
  },
```

On peut ensuite lancer nos tests avec :

```sh
npm run unit
```

Le résultat :

```
  AdView
    ✔ One ad view should debit publisher and user, and credit advertiser
    ✔ Should throw an exception if the advertiser does not have enough credit


  2 passing (6ms)
```

Essayez vous même dans votre DevContainer !

Essayez d'apporter une modification qui change le comportement du module Adview pour voir comment les tests réagissent !

### Exclure nos tests du build final

Nous avons ajouté des fichiers `.ts` à notre projet qu'on ne veut pas forcément inclure dans le build final. Si on essaye de lancer `tsc` sans la libraire `mocha` installé, il y aura une erreur.

Nous allons donc exclure notre dossier `test` des builds en production. Pour cela, il faut créer un `tsconfig.build.json` qui dérive du `tsconfig.json` de base, mais qui exclue le dossier `test` :

```json
{
  "extends": "./tsconfig",
  "exclude": [
    "test"
  ]
}
```

On modifie ensuite le script `build` dans `package.json` pour utiliser plutôt ce `tsconfig.build.json` :

```json
  ...
  "scripts": {
    "server": "nodemon",
    "compile": "tsoa -r tsconfig-paths/register spec-and-routes",
    "clean": "rimraf build",
    "build": "npm run clean && npm run compile && tsc -p tsconfig.build.json && tsc-alias -p tsconfig.build.json && copyfiles public/**/* build/",
    ...
```

### Une bonne conception des tests

Attention ! La rédaction des tests peut être assez longue.

De façon générale, je compte 1 unité de temps pour la création d'un module, et 2 ou 3 fois plus pour la conception et rédaction des tests.

Pourquoi ?

* Il faut imaginer TOUS les scenarii possibles : l'usage normal, les edge-cases, les erreurs, etc
* Il faut être capable de créer les données entrantes, et les conditions de test avant de le lancer. Cela peut être longue et ardue.

En revanche, le temps et la douleur que les tests enlèvent dans le futur fait que ça vaut le coup !!

Et s'il y a des dépendances ?

Mais, vous dites, cela est très bien, mais pour la plupart, nos plateformes vont interagir avec des dépendances externes. Notamment, une base de données.

À ce moment-là, nous élargirons l'étendu de nos tests avec des **tests d'intégration**.


