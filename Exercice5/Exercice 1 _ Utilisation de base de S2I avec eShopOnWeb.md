# Exercice 1 : Utilisation de base de S2I avec eShopOnWeb

## Objectif
Déployer l'application eShopOnWeb en utilisant l'image S2I .NET Core standard d'OpenShift pour comprendre le processus de base de Source-to-Image.

## Prérequis
- Accès à un cluster OpenShift
- CLI OpenShift (oc) installé et configuré
- Connexion au cluster avec les droits appropriés

## Description
Dans cet exercice, vous allez déployer l'application de référence eShopOnWeb directement depuis son dépôt GitHub en utilisant la fonctionnalité S2I intégrée d'OpenShift.

## Étapes

### 1. Création d'un nouveau projet
```bash
oc new-project eshop-s2i-base
```

### 2. Déploiement avec S2I
```bash
oc new-app dotnet:8.0~https://github.com/dotnet-architecture/eShopOnWeb \
  --context-dir=src/Web \
  --name=eshopweb \
  -e ASPNETCORE_ENVIRONMENT=Development
```

### 3. Exposition du service
```bash
oc expose service/eshopweb
```

### 4. Vérification du déploiement
```bash
# Vérifier le statut du build
oc get builds

# Vérifier le statut des pods
oc get pods

# Obtenir l'URL de l'application
oc get route eshopweb
```

### 5. Test de l'application
Accédez à l'URL obtenue et vérifiez que l'application eShopOnWeb fonctionne correctement.

## Points clés à observer
- Le processus de build automatique avec S2I
- La détection automatique du framework .NET Core
- La création automatique des ressources OpenShift
- Le temps de build et de déploiement

## Validation
- L'application est accessible via l'URL de la route
- La page d'accueil d'eShopOnWeb s'affiche correctement
- Les logs du build montrent les étapes S2I

## Nettoyage
```bash
oc delete project eshop-s2i-base
```

## Points d'apprentissage
- Simplicité du déploiement avec S2I
- Détection automatique du type d'application
- Intégration native avec OpenShift
- Limitations de l'approche standard

