# Evaluation

L'évaluation se fera comme suit :

Vous avez ajouté un champ à l'endpoint `/info` :

```json
{
   ...
   "name" : "VOTRE NOM ET PRENOM"
}
```

- Vous avez créé un compte gitlab, et poussé une version de votre projet : 2 points
- Un pipeline fonctionnel avec :
  - Une étape de test unitaire : 2 points
  - Une phase de tests d'intégration : 3 points
  - Une phase de test e2e : 3 points
  - Une étape de construction : 3 points
  - Une étape de déploiement : 3 points
- Un déploiement fonctionnel sur le cluster Kubernetes, et un endpoint /info qui fonctionne correctement (affiche votre nom dans le json retourné) : 4 points
