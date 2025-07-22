---

## 🏋️‍♂️ **Exercice 3 : Customiser S2I sur Spring Petclinic**

### **Objectif**

Personnaliser le build S2I de Petclinic pour :

* Ajouter une étape custom (ex : écrire un fichier `info-build.txt` qui indique la date du build)
* Utiliser `.s2i/bin/assemble` pour compléter le build Maven standard

---

### **Étapes détaillées**

#### 1️⃣ **Cloner et préparer le projet**

```bash
git clone https://github.com/spring-projects/spring-petclinic.git
cd spring-petclinic
```

#### 2️⃣ **Créer le dossier `.s2i/bin`**

```bash
mkdir -p .s2i/bin
```

#### 3️⃣ **Créer le script custom `.s2i/bin/assemble`**

Dans `.s2i/bin/assemble` :

```bash
#!/bin/bash
echo "=== [Custom S2I] Build Petclinic - $(date) ===" > /tmp/info-build.txt

# Par exemple, tu pourrais injecter un secret, télécharger un fichier, modifier un .properties ici

# Lance le build Maven standard (S2I Java attend ce script !)
exec /usr/local/s2i/assemble
```

Lancer un pod temporaire pour vérifier l'emplacement de s2i: 

```bash
oc run s2i-debug --rm -ti --image=registry.redhat.io/ubi8/openjdk-17 -- bash

ls /usr/local/s2i/
```

Rends-le exécutable :

```bash
dos2unix .s2i/bin/assemble

chmod +x .s2i/bin/assemble
```

#### 4️⃣ **Lancer le build S2I comme d’habitude**

```bash
oc new-build --binary --name=petclinic-s2i-custom --image-stream=openshift/ubi8-openjdk-17:1.18
oc start-build petclinic-s2i-custom --from-dir=. --follow
```

#### 5️⃣ **Vérifier la prise en compte de ton hook**

Dans les logs du build, tu dois voir la ligne :

```
=== [Custom S2I] Build Petclinic - <date> ===
```

> Le fichier `/tmp/info-build.txt` sera présent dans l’image finale (dans `/tmp`).

#### 6️⃣ **Déployer et exposer l’application**

```bash
oc new-app petclinic-s2i-custom
```

#### Vérifier le contenu du fichier `/tmp/info-build.txt`

```bash
oc get pods
oc rsh petclinic-s2i-custom-7cdf76595f-j8thp
cat /tmp/info-build.txt
```

```bash
oc expose svc/petclinic-s2i-custom
oc get route
```

---

## **Bonus**

* Tu peux customiser plus loin :
  → Ajouter des étapes, copier des fichiers, modifier des variables d’env, etc dans `.s2i/bin/assemble`.
* Même logique possible avec `.s2i/bin/run` pour changer le comportement de lancement (démarrage debug, options JVM dynamiques…).

---

## **Ce que tu retires de l’exercice**

* Tu maîtrises la **customisation du process S2I** sur un vrai projet Java/Spring
* Tu peux injecter des hooks dans le cycle de build sans changer le Dockerfile
* **Idéal pour des pipelines CI/CD industriels !**

---
