---

# TP – **Créez et déployez votre premier template OpenShift**

## 🎯 Objectif pédagogique

* Comprendre la structure d’un template OpenShift (YAML)
* Apprendre à paramétrer un template (variables, secret)
* Déployer en une seule commande une application multi-ressources (web + base de données)
* Réutiliser et partager le template

---

## 1️⃣ **Préparation**

1. **Créez un dossier de travail** sur votre poste :

   ```bash
   mkdir tp-template-openshift
   cd tp-template-openshift
   ```

---

## 2️⃣ **Création d’un template personnalisé**

1. **Créez le fichier `webapp-mysql-template.yaml`** avec le contenu suivant :

   ```yaml
   apiVersion: template.openshift.io/v1
   kind: Template
   metadata:
     name: webapp-mysql-example
     annotations:
       description: Application web avec MySQL (exemple de template)
   parameters:
     - name: APP_NAME
       description: Nom de l'application
       required: true
     - name: MYSQL_VERSION
       description: Version de MySQL
       value: "8.0"
     - name: DB_PASSWORD
       description: Mot de passe BDD
       generate: expression
       from: "[a-zA-Z0-9]{16}"
       required: true
   objects:
     - apiVersion: v1
       kind: Secret
       metadata:
         name: ${APP_NAME}-secret
       stringData:
         password: ${DB_PASSWORD}
     - apiVersion: v1
       kind: Service
       metadata:
         name: ${APP_NAME}
       spec:
         ports:
           - port: 8080
             targetPort: 8080
         selector:
           app: ${APP_NAME}
     - apiVersion: apps.openshift.io/v1
       kind: DeploymentConfig
       metadata:
         name: ${APP_NAME}
       spec:
         replicas: 1
         selector:
           app: ${APP_NAME}
         template:
           metadata:
             labels:
               app: ${APP_NAME}
           spec:
             containers:
               - name: web
                 image: nginxinc/nginx-unprivileged:latest
                 ports:
                   - containerPort: 8080
     - apiVersion: v1
       kind: Route
       metadata:
         name: ${APP_NAME}
       spec:
         to:
           kind: Service
           name: ${APP_NAME}
   ```

   > 💡 **Remarque** : Ce template déploie un service web (nginx) et les ressources associées.

---

## 3️⃣ **Importer et utiliser le template dans OpenShift**

1. **Importez le template dans votre projet** :

   ```bash
   oc create -f webapp-mysql-template.yaml
   ```
    vérifier la création du template: 

   ```bash
   oc get template
   ```

2. **Déployez une instance personnalisée** :

   ```bash
   oc new-app --template=webapp-mysql-example -p APP_NAME=demo-app -p DB_PASSWORD=MySuperSecret
   ```
---

## 4️⃣ **Vérification**

1. **Vérifiez que les ressources sont créées** :

   ```bash
   oc get pods
   oc get svc
   oc get route
   oc get pods,svc,route
   ```

2. **Testez l’accès à l’application** :
   Rendez-vous sur l’URL fournie par la commande `oc get route`.

## 5️⃣ **Nettoyage**

```bash
oc delete all -l app=demo-app 
oc delete template webapp-mysql-example
```