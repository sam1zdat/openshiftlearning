
## Partie 1 : Création avancée d'applications à partir de modèles OpenShift

### 1.1 Création d'un modèle multi-conteneurs complet

1. Créez un nouveau projet(pas sur Sandbox) :
   ```bash
   oc new-project tp3-templates-<votre-nom>
   ```

2. Créez un fichier three-tier-template.yaml et créez un modèle pour une application à trois tiers (frontend, backend, base de données) :
   ```bash   
   oc create -f three-tier-template.yaml
   ```

### 1.2 Déploiement à partir du modèle complexe

1. Déployez une application à partir du modèle :
   ```bash
   oc new-app --template=three-tier-app \
     -p ENVIRONMENT=dev \
     -p BACKEND_REPLICAS=1 \
     -p FRONTEND_REPLICAS=1 \
     -p LOGGING_LEVEL=DEBUG
   ```

2. Vérifiez les ressources créées :
   ```bash
   oc get all
   oc get configmap
   oc get secret
   ```

### 1.3 Paramétrage pour différents environnements

1. Créez un fichier de paramètres pour l'environnement de production `production-params.env`:
   ```bash
     BACKEND_REPLICAS=3
     FRONTEND_REPLICAS=3
     BACKEND_MEMORY_LIMIT=1Gi
     BACKEND_CPU_LIMIT=1000m
     FRONTEND_MEMORY_LIMIT=512Mi
     FRONTEND_CPU_LIMIT=500m
     LOGGING_LEVEL=WARN
     ENABLE_EXPERIMENTAL_FEATURES=false
     DATABASE_SERVICE_NAME=postgresqldemo
     DATABASE_NAME=demodb
     BACKEND_SERVICE_NAME=backenddemo
     FRONTEND_SERVICE_NAME=frontenddemo
     APP_CONFIG_MAP=app-config-demo
   ```

1. Déployez une version de production dans un nouveau projet :
   ```bash
   oc new-project tp3-templates-prod-<votre-nom>
   ou
   oc delete all -l app=three-tier-app
   oc new-app --template=three-tier-app --param-file=production-params.env
   ou
   oc new-app --template=three-tier-app --param-file=production-params.env [--labels=app=env-prod]
   
   ```
