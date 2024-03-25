# Kubernetes

Nous avons parlé de la redondance et de la mise à l'échelle horizontale, mais ce travail peut être difficile à réaliser manuellement.

Nous utilisons donc une technique appelée **orchestration** pour approvisionner les ressources du nuage et démarrer nos applications automatiquement.

<figure><img src="../graphics/Kubernetes components.png" alt=""><figcaption><p><a href="https://kubernetes.io/docs/concepts/overview/components/">Source</a></p></figcaption></figure>

Avec l'orchestration, nous spécifions l'état **désiré** de notre architecture, par exemple :

* Je veux un load-balancer
* Je veux que 4 applications API fonctionnent en parallèle, chaque requête étant redirigée vers une application différente pour équilibrer la charge.
* Je veux que 3 instances de base de données soient en cours d'exécution, chacune connectée à un volume spécifique pour le stockage des données.

Nous spécifions ces souhaits dans un format dédié lisible par une machine.

Des technologies telles que Kubernetes sont capables de lire notre configuration souhaitée et de formuler un plan d'action sur la manière de la fournir. En général, ce plan est intégralement lié à notre fournisseur de cloud (qui sait comment fournir des ressources telles que des volumes et des répartiteurs de charge).

Cela présente de nombreux avantages :

* Nous pouvons écrire **ce que nous voulons**, et non pas comment l'obtenir.
* La syntaxe formelle peut être stockée et versionnée comme du code normal.
* Dans le cas d'une erreur (une VM qui disparaît), le système d'orchestration provisionnera automatiquement une nouvelle VM sans que nous ayons à intervenir. En fait, il surveille automatiquement son déploiement et détecte si l'architecture déployée diffère de l'état souhaité. Il prend alors des mesures correctives.
* Il en va de même pour les applications : si une application tombe en panne, le système d'orchestration la redémarre automatiquement, soit sur la même VM, soit sur une nouvelle VM.
* Le système d'orchestration peut évoluer automatiquement : il surveille l'utilisation des ressources (puissance CPU, RAM) et s'il détecte une sur-utilisation, il peut automatiquement provisionner plus de VMs et lancer de nouveaux processus pour gérer la charge.
