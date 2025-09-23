# Exercice 3 : Déploiement avancé avec S2I et intégration base de données

## Objectif
Configurer un déploiement complet d'eShopOnWeb avec S2I personnalisé, incluant la gestion multi-environnements et l'intégration avec une base de données PostgreSQL.

## Prérequis
- Exercices 1 et 2 complétés
- Image S2I personnalisée disponible
- Connaissances de base en SQL

## Description
Cet exercice avancé démontre un déploiement production-ready d'eShopOnWeb avec configuration de base de données, secrets, et variables d'environnement.

## Étapes

### 1. Préparation de l'environnement
```bash
# Créer le projet pour l'environnement de production
oc new-project eshop-production

# Créer les labels pour l'organisation
oc label namespace eshop-production environment=production app=eshopweb
```

### 2. Déploiement de PostgreSQL
```bash
# Déployer PostgreSQL avec stockage persistant
oc new-app postgresql-persistent \
  --param=POSTGRESQL_USER=eshopuser \
  --param=POSTGRESQL_PASSWORD=eshoppass \
  --param=POSTGRESQL_DATABASE=eshopdb \
  --param=VOLUME_CAPACITY=5Gi \
  --name=eshop-db
```

### 3. Création des secrets et ConfigMaps
```bash
# Secret pour la base de données
oc create secret generic eshop-db-secret \
  --from-literal=username=eshopuser \
  --from-literal=password=eshoppass \
  --from-literal=database=eshopdb

# ConfigMap pour les configurations d'application
oc create configmap eshop-config \
  --from-literal=ASPNETCORE_ENVIRONMENT=Production \
  --from-literal=UseOnlyInMemoryDatabase=false \
  --from-literal=ASPNETCORE_URLS=http://*:8080
```

### 4. Template de déploiement avec S2I personnalisé
```yaml
# eshop-template.yaml
apiVersion: template.openshift.io/v1
kind: Template
metadata:
  name: eshopweb-s2i-template
parameters:
- name: SOURCE_REPOSITORY_URL
  value: https://github.com/dotnet-architecture/eShopOnWeb
- name: DATABASE_SERVICE_NAME
  value: eshop-db
objects:
- apiVersion: apps.openshift.io/v1
  kind: DeploymentConfig
  metadata:
    name: eshopweb
  spec:
    replicas: 2
    selector:
      app: eshopweb
    template:
      metadata:
        labels:
          app: eshopweb
      spec:
        containers:
        - name: eshopweb
          image: eshopweb:latest
          ports:
          - containerPort: 8080
          env:
          - name: ConnectionStrings__CatalogConnection
            value: "Host=${DATABASE_SERVICE_NAME};Database=eshopdb;Username=eshopuser;Password=eshoppass"
          envFrom:
          - configMapRef:
              name: eshop-config
          - secretRef:
              name: eshop-db-secret
    triggers:
    - type: ImageChange
      imageChangeParams:
        automatic: true
        containerNames:
        - eshopweb
        from:
          kind: ImageStreamTag
          name: eshopweb:latest
```

### 5. Déploiement avec BuildConfig personnalisé
```bash
# Créer le BuildConfig avec l'image S2I personnalisée
oc new-build custom-dotnet-s2i:latest~${SOURCE_REPOSITORY_URL} \
  --name=eshopweb \
  --context-dir=src/Web \
  --env=DOTNET_STARTUP_PROJECT=Web.csproj

# Appliquer le template
oc process -f eshop-template.yaml | oc apply -f -
```

### 6. Configuration du monitoring et health checks
```bash
# Ajouter des health checks
oc set probe dc/eshopweb \
  --readiness --get-url=http://:8080/health/ready \
  --initial-delay-seconds=30 \
  --period-seconds=10

oc set probe dc/eshopweb \
  --liveness --get-url=http://:8080/health/live \
  --initial-delay-seconds=60 \
  --period-seconds=30
```

### 7. Exposition et configuration du routage
```bash
# Créer la route avec TLS
oc create route edge eshopweb \
  --service=eshopweb \
  --hostname=eshop.apps.cluster.local \
  --insecure-policy=Redirect
```

### 8. Configuration de l'autoscaling
```bash
# Configurer l'autoscaling horizontal
oc autoscale dc/eshopweb --min=2 --max=5 --cpu-percent=70
```

### 9. Initialisation de la base de données
```bash
# Job pour initialiser la base de données
oc create job eshop-db-init --image=eshopweb:latest -- \
  dotnet ef database update --project Web.csproj
```

## Validation
- PostgreSQL est déployé et accessible
- L'application eShopOnWeb se connecte à la base de données
- Les health checks fonctionnent correctement
- L'autoscaling est configuré
- L'application est accessible via HTTPS

## Points d'apprentissage
- Intégration S2I avec base de données externe
- Gestion des secrets et configurations
- Health checks et monitoring
- Déploiement production-ready
- Templates OpenShift pour la réutilisabilité

