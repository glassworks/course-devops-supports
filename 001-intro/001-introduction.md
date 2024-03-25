# Introduction

Qu'est-ce que DevOps ?

DevOps, ou abréviation de "development and operations" englobe de multiples responsabilités au sein d'un même profil :

- Conception, développement d'une plateforme en ligne
- Conception et création d'une infrastructure de déploiement (serveurs, load-balancers, ...).
- Déploiement d'une application sur l'infrastructure de déploiement
- Maintenance (opérations) de l'infrastructure de déploiement, y compris le monitoring, la réponse aux incidents, l'accroissement, ....
- Automatisation des actions de déploiement : construction, test et déploiement automatiques.

Un employé recruté dans le rôle de DevOps aura tout ou partie de ces responsabilités. 

Dans une startup, le développeur s'occupera également du déploiement et des opérations. 

Dans une grande entreprise, l'équipe de développement est totalement séparée de l'équipe de production. L'équipe de production est uniquement responsable du déploiement des nouvelles versions, ainsi que de la conception et de la maintenance de l'architecture de déploiement.

**Cependant, le terme DevOps concerne toujours le déploiement d'une version *live*, prête pour la production, d'une application.**

Le rôle d'un DevOps nécessite des connaissances dans de multiples domaines :

- les infrastructures de déploiement, notamment les serveurs qui hébergeront l'application. Aujourd'hui, cela se fait principalement dans le Cloud
- les bonnes pratiques de développement : versioning, testing, ....
- automatisation des tâches
- surveillance : logging, monitoring, ....


Au cours de cette formation, vous serez donc exposé aux éléments suivants :

- Une introduction au Cloud, son vocabulaire et ses composants, et comment concevoir une architecture pour une API simple.
- Déploiement manuel d'une API simple dans le cloud à l'aide de Kubernetes.
- Automatisation fiable du processus de déploiement (également connu sous le nom de CI/CD) :
   - Utilisation de git pour gérer les versions (tags)
   - Déclencher des builds automatiques dans Git
   - Déclencher des tests automatiques dans Git
   - Déclencher un déploiement automatique avec Git