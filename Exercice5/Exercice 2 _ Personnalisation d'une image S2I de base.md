# Exercice 2 : Personnalisation d'une image S2I de base

## Objectif
Créer une image S2I personnalisée pour optimiser le déploiement d'eShopOnWeb avec des dépendances spécifiques et des configurations adaptées.

## Prérequis
- Exercice 1 complété
- Docker ou Podman installé
- Accès à un registre de conteneurs (interne OpenShift ou externe)

## Description
Cet exercice vous guide dans la création d'une image S2I personnalisée basée sur l'image .NET Core officielle, avec des optimisations spécifiques pour eShopOnWeb.

## Étapes

### 1. Création de la structure de l'image S2I personnalisée
```bash
mkdir -p custom-dotnet-s2i/{s2i/bin,root}
cd custom-dotnet-s2i
```

### 2. Création du Dockerfile personnalisé
```dockerfile
# Dockerfile
FROM registry.redhat.io/ubi8/dotnet-80:latest

# Métadonnées de l'image S2I
LABEL io.k8s.description="Custom .NET Core 8.0 S2I image for eShopOnWeb" \
      io.k8s.display-name="Custom .NET Core 8.0" \
      io.openshift.s2i.scripts-url="image:///usr/libexec/s2i"

# Installation d'outils supplémentaires pour eShopOnWeb
USER root
RUN dnf install -y nodejs npm && \
    dnf clean all

# Configuration spécifique pour eShopOnWeb
ENV DOTNET_STARTUP_PROJECT=src/Web/Web.csproj \
    ASPNETCORE_URLS=http://*:8080 \
    DOTNET_RUNNING_IN_CONTAINER=true

# Copie des scripts S2I personnalisés
COPY s2i/bin/ /usr/libexec/s2i/

# Permissions
RUN chmod +x /usr/libexec/s2i/*

USER 1001
```

### 3. Script assemble personnalisé
```bash
# s2i/bin/assemble
#!/bin/bash
set -e

echo "Démarrage du script assemble personnalisé pour eShopOnWeb"

# Appel du script assemble original
/usr/libexec/s2i/assemble

echo "Installation des dépendances Node.js pour eShopOnWeb"
if [ -f "src/Web/package.json" ]; then
    cd src/Web
    npm install
    cd ../..
fi

echo "Optimisation des assemblies .NET"
dotnet publish src/Web/Web.csproj -c Release -o /opt/app-root/app --no-restore

echo "Script assemble personnalisé terminé"
```

### 4. Script run personnalisé
```bash
# s2i/bin/run
#!/bin/bash
set -e

echo "Démarrage d'eShopOnWeb avec configuration personnalisée"

cd /opt/app-root/app
exec dotnet Web.dll
```

### 5. Construction de l'image personnalisée
```bash
# Rendre les scripts exécutables
chmod +x s2i/bin/*

# Construire l'image
podman build -t custom-dotnet-s2i:latest .

# Tagger pour le registre OpenShift
podman tag custom-dotnet-s2i:latest default-route-openshift-image-registry.apps.cluster.local/openshift/custom-dotnet-s2i:latest
```

### 6. Push vers le registre OpenShift
```bash
# Se connecter au registre interne
oc registry login

# Pousser l'image
podman push default-route-openshift-image-registry.apps.cluster.local/openshift/custom-dotnet-s2i:latest
```

### 7. Test de l'image personnalisée
```bash
# Créer un nouveau projet
oc new-project eshop-custom-s2i

# Déployer avec l'image personnalisée
oc new-app custom-dotnet-s2i:latest~https://github.com/dotnet-architecture/eShopOnWeb \
  --name=eshopweb-custom \
  -e ASPNETCORE_ENVIRONMENT=Production

# Exposer le service
oc expose service/eshopweb-custom
```

## Validation
- L'image personnalisée se construit sans erreur
- Le déploiement avec l'image personnalisée fonctionne
- Les optimisations sont visibles dans les logs de build
- L'application démarre plus rapidement qu'avec l'image standard

## Points d'apprentissage
- Structure d'une image S2I personnalisée
- Personnalisation des scripts assemble et run
- Intégration d'outils supplémentaires
- Optimisations spécifiques à l'application

