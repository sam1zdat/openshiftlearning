
---

## 🏋️‍♂️ **Exercice d’introduction à S2I avec OpenShift**

### **Objectif**

Découvrir la construction d’images d’application à partir du code source, sans Dockerfile, en utilisant la fonctionnalité **Source-to-Image (S2I)** d’OpenShift.

---

### **Scénario**

Vous êtes développeur et vous souhaitez déployer une petite application web Java sur OpenShift, sans écrire de Dockerfile.
OpenShift vous permet d’automatiser la création de l’image grâce à S2I : il télécharge votre code source, compile et assemble automatiquement l’image exécutable.

---

### **Étapes**

#### 1. **Récupérer un projet d’exemple Java (Quarkus ou Spring Boot)**

```bash
git clone https://github.com/spring-projects/spring-petclinic.git
cd spring-petclinic
```

> (Ou utilisez un autre projet Java avec un `pom.xml` à la racine)

---

#### 2. **Créer le build S2I dans OpenShift**

```bash
oc new-build --binary --name=spring-petclinic --image-stream=openshift/ubi8-openjdk-17:1.18
```
---

### **Ce que la commande fait concrètement**

* Crée un objet **BuildConfig** nommé `spring-petclinic` : définit comment sera construit le code.
* Crée un objet **ImageStream** nommé `spring-petclinic` : référencera l’image générée.
* Se base sur le builder S2I `ubi8-openjdk-17` pour la construction.
* Attend que tu fournisses le code source via une commande suivante (`oc start-build ...`).

---

#### 3. **Lancer la construction de l'image spring-petclinic de votre application spring-petclinic à partir du code source**

```bash
oc start-build spring-petclinic --from-dir=. --follow
```

* **Démarre un build** pour le BuildConfig `spring-petclinic`.
* **--from-dir=.** : Envoie le code source du dossier courant.
* **--follow** : Affiche les logs en temps réel.

### **Objets OpenShift impliqués** :

* **BuildConfig** : (déjà créé) définit comment construire l’image.
* **Build** : nouvel objet créé à chaque `start-build` (ex : `build/spring-petclinic-1`).
* **ImageStream** : (déjà créé) reçoit l’image générée à la fin du build.

### en cas d'erreur, lancer 
```bash
oc logs build/spring-petclinic-1 | less
```

---

#### 4. **Déployer l’application à partir de l'image spring-petclinic**

```bash
oc new-app spring-petclinic
oc expose svc/spring-petclinic
```

> Récupérez l’URL avec :

```bash
oc get route
```

---

#### 5. **Questions pour bien comprendre**

* Quelle est la différence entre S2I et la création d’image via Dockerfile ?
* Quels sont les avantages d’utiliser S2I pour les développeurs ?
* Comment OpenShift détecte-t-il le langage et le type de build à effectuer ?

---

### **À la fin de l’exercice, vous saurez :**

* Utiliser la commande S2I pour créer une image applicative à partir du code source
* Déployer une application en quelques commandes, sans Dockerfile

---

**Variante** : Faites le même exercice avec une application Node.js ou Python pour montrer la polyvalence de S2I.

---
