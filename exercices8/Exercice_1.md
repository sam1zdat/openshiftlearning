---

````markdown
# TP – Migration d’une Application Monolithique vers OpenShift (Spring Petclinic)

## 🎯 Contexte

Vous travaillez dans l’équipe IT d’une PME.  
Votre application métier “**Spring Petclinic**” tourne en local ou sur une VM avec accès à une base MySQL/PostgreSQL.  
Objectif : **Migrer l’application sur OpenShift**, en suivant les bonnes pratiques de modernisation cloud.

---

## 1️⃣ **Audit de l’existant**

### a. **Récupérer le code**
- Clonez le dépôt officiel :
  ```bash
  git clone https://github.com/spring-projects/spring-petclinic.git
  cd spring-petclinic
````

### b. **Identifier les points à adapter**

* BDD : H2 (par défaut), mais peut être basculée sur MySQL/PostgreSQL.
* Configurations sensibles (`application.properties`) : à externaliser.
* Build : application Spring Boot monolithique, livrée en `.jar`.

---

## 2️⃣ **Préparation de la migration**

### a. **Créer un projet OpenShift(non disponible dans Sandbox)**

```bash
oc new-project tp-petclinic
```

### b. **Configurer la base de données**

* Déployez une base MySQL ou PostgreSQL dans le même projet (exemple pour PostgreSQL) :

  ```bash
  oc new-app --name=petclinic-db \
    -e POSTGRESQL_USER=petclinic \
    -e POSTGRESQL_PASSWORD=petclinicpwd \
    -e POSTGRESQL_DATABASE=petclinic \
    registry.redhat.io/rhscl/postgresql-12-rhel7
  ```

---

## 3️⃣ **Conteneurisation de l’application**

### a. **Construire le .jar**

```bash
./mvnw package -DskipTests
```

> Le jar sera dans `target/spring-petclinic-*.jar`

### b. **Créer un Dockerfile**

```dockerfile
FROM openshift/ubi8-openjdk-17:1.18 #registry.access.redhat.com/ubi8/openjdk-17
COPY target/*.jar /app/app.jar
WORKDIR /app
CMD ["java", "-jar", "app.jar"]
```

> Adapte la version d’OpenJDK si besoin (11/17).

### c. **Construire et publier l’image**

* Soit localement (avec Docker/podman puis `oc new-app`),
* Soit en utilisant S2I sur OpenShift :

  ```bash
  oc new-build --binary --name=petclinic-app --image-stream=ubi8-openjdk-17:latest
  oc start-build petclinic-app --from-dir=. --follow
  ```

---

## 4️⃣ **Externalisation de la configuration**

### a. **Créer un Secret pour les accès BDD**

```bash
oc create secret generic petclinic-db-secret \
  --from-literal=SPRING_DATASOURCE_USERNAME=petclinic \
  --from-literal=SPRING_DATASOURCE_PASSWORD=petclinicpwd
```

### b. **Créer une ConfigMap pour les variables de connexion**

```bash
oc create configmap petclinic-db-config \
  --from-literal=SPRING_DATASOURCE_URL=jdbc:postgresql://petclinic-db:5432/petclinic
```

### c. **Injecter Secret et ConfigMap dans le déploiement**

```bash
oc set env deployment/petclinic-app --from=secret/petclinic-db-secret
oc set env deployment/petclinic-app --from=configmap/petclinic-db-config
```

---

## 5️⃣ **Ajout de probes de santé**

```bash
oc set probe deployment/petclinic-app \
  --liveness --get-url=http://:8080/ --initial-delay-seconds=30
oc set probe deployment/petclinic-app \
  --readiness --get-url=http://:8080/ --initial-delay-seconds=5
```

---

## 6️⃣ **Exposition de l’application**

```bash
oc expose svc/petclinic-app
oc get route
```

* Accédez à l’URL générée pour tester Petclinic sur OpenShift.

---

## 7️⃣ **Validation**

* Vérifiez l’accès à l’app, la connexion BDD, l’ajout de clients.
* Surveillez les pods, logs et probes via la console OpenShift.

---

## 8️⃣ **Questions/Réflexion**

* Qu’avez-vous dû modifier pour “cloud-nativer” Petclinic ?
* Quelles étapes seraient nécessaires pour aller vers une architecture microservices ?
* Comment sécuriser encore plus la gestion des secrets ?

---

## 9️⃣ **Nettoyage**

```bash
oc delete project tp-petclinic
```

---

## 💡 **Bonus**

* Déployez une version canary pour tester la répartition de trafic.
* Ajoutez un Healthcheck avancé (`/actuator/health`).
* Testez la persistance réelle (pvc) si l’application stocke des fichiers.

---