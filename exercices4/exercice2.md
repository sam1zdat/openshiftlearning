---

## 🏋️‍♂️ **Exercice S2I Avancé : Déployer une appli Java avec base PostgreSQL**

### **Objectif**

Déployer une application Java (ex : Spring Boot ou Quarkus) sur OpenShift **via S2I**, la connecter à une base de données PostgreSQL gérée par OpenShift, et paramétrer l’accès via des variables d’environnement.

---

### **Étapes**

#### 1. **Cloner une application Java compatible base de données**

Par exemple :

```bash
git clone https://github.com/spring-projects/spring-petclinic.git
cd spring-petclinic
```

#### 2. **Déployer une base PostgreSQL sur OpenShift**

```bash
oc new-app --name=petclinic-db \
  -e POSTGRESQL_USER=petclinic \
  -e POSTGRESQL_PASSWORD=petclinicpwd \
  -e POSTGRESQL_DATABASE=petclinic \
  registry.redhat.io/rhscl/postgresql-12-rhel7
```

#### 3. **Créer le build S2I pour l’appli Java**

```bash
oc new-build --binary --name=petclinic-app --image-stream=openshift/ubi8-openjdk-17:1.18

oc start-build petclinic-app --from-dir=. --follow
```

#### 4. **Déployer l’application Java**

```bash
oc new-app petclinic-app
```

#### 5. **Configurer les variables d’environnement pour la connexion BDD**
-- Lister les deploiements et afficher les détails du deploiement petclinic-app:

```bash
oc get deployment
oc descibe deployment petclinic-app
```

```bash
oc set env deployment/petclinic-app \
  SPRING_DATASOURCE_URL=jdbc:postgresql://petclinic-db:5432/petclinic \
  SPRING_DATASOURCE_USERNAME=petclinic \
  SPRING_DATASOURCE_PASSWORD=petclinicpwd \
  SPRING_JPA_HIBERNATE_DDL_AUTO=update
```


#### 6. **Exposer l’application**

```bash
oc expose svc/petclinic-app
oc get route
```

#### 7. **Tester l’application**

Ouvre l’URL et vérifie que l’application Java fonctionne **avec la base de données PostgreSQL** (ajoute un client, vérifie la persistance…).

### Pour tester la connectivité (optionnel) 
Tu peux lancer un pod temporaire pour tester la connexion :

```bash
oc run testpg --rm -ti --image=registry.redhat.io/rhscl/postgresql-12-rhel7 -- bash
# puis dans le pod :
psql -h petclinic-db -U petclinic petclinic
# (le mot de passe sera petclinicpwd si tu as suivi l’exemple)
```

---

### **Pour aller plus loin :**

* **Ajoute un secret pour la gestion des mots de passe** (au lieu de stocker le mot de passe en clair)

  ```bash
  oc create secret generic petclinic-db-secret --from-literal=POSTGRESQL_PASSWORD=petclinicpwd

  oc set env deployment/petclinic-db --from=secret/petclinic-db-secret
  
  oc set env deployment/petclinic-app \
  --from=secret/petclinic-db-secret

  ```
* **Scale** l’application à 3 répliques :

  ```bash
  oc scale deployment/petclinic-app --replicas=3
  ```
* **Simule un rolling update** en modifiant le code et en relançant un build S2I
* **Ajoute un HealthCheck** avec `oc set probe`

---

## **Ce que tu apprends avec cet exercice :**

* Déployer des applications multi-conteneurs sur OpenShift
* Utiliser S2I avec des paramètres personnalisés
* Configurer des connexions BDD via variables d’environnement
* Gérer le cycle de vie de l’application dans un environnement cloud

---