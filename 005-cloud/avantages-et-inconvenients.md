# Avantages et inconvénients

{% hint style="success" %}
Concevoir une architecture dans le Cloud présente aujourd'hui de nombreux avantages :

* **simplicité** : un développeur ou une entreprise peut déployer des services en quelques clics. Aucune expertise avancée n'est requise.
* **externaliser** tout ou partie du matériel physique et des logiciels
* **accessibilité accrue** : accéder, contrôler, maintenir vos services depuis n'importe où.
* **sécurité centralisée et mutualisée** : profiter des plans de sécurité avancés du fournisseur de cloud, y compris les stratégies de sauvegarde, la tolérance aux pannes, la protection contre les pirates, etc.
* **performances accrues** : le Cloud offre la possibilité d'une **évolution horizontale**, c'est-à-dire l'ajout de nombreuses ressources plus petites et moins coûteuses, par opposition à la mise à niveau de grands services
* **redondance accrue** : le Cloud simplifie l'ajout de ressources redondantes et supprime les **points de défaillance uniques**.
* **rentabilité** : le modèle de **paiement à l'utilisation** peut réduire les coûts globaux. L'ajout de ressources uniquement en cas de besoin peut améliorer la disponibilité globale, tout en optimisant la facturation.

... et beaucoup d'autres. Vous en voyez ?
{% endhint %}

{% hint style="warning" %}
Il faut toutefois être conscient des inconvénients :

* **facturation incontrôlable** : si vous ne faites pas attention, vous risquez de vous retrouver avec des factures gonflées. Cela se produit généralement en raison d'une mauvaise programmation ou d'erreurs de l'utilisateur. Par exemple, l'exécution d'un grand nombre de transferts de données inutiles (`select * from table`) peut rapidement faire grimper votre facture de transfert de données. La plupart des fournisseurs de services Cloud vous permettent de configurer des alertes de facturation.
* **services coûteux** : certains services dans le Cloud sont très coûteux, mais uniques sur le marché. Nous sommes dépendants de ces services, mais les fournisseurs profitent de leur position pour facturer davantage.
* **lock-in** : il est facile de s'enfermer dans un fournisseur particulier, ce qui rend la migration vers un autre fournisseur très difficile, voire impossible. Cependant, la notion de **multi-cloud** prend racine aujourd'hui, résolvant ce problème.
* **breakdowns** : lorsque les choses tournent mal, elles tournent très mal. Nous faisons confiance aux fournisseurs de cloud et dépendons de leur expertise, mais dans le cas d'une panne (ce qui est rare), nous n'avons absolument aucun contrôle sur la situation, ni l'expertise pour fournir une alternative. Une fois de plus, dans ce cas, le multi-cloud et la redondance sont des solutions possibles.
{% endhint %}
