---

## 🖥️ **Exercice CLI : Découverte des commandes oc**

### **Objectif**

Manipuler les ressources OpenShift depuis le terminal, explorer, déployer et observer le cycle de vie d’une application.

---

### **Étapes**

#### 1️⃣ **Lister les projets**

```bash
oc get projects
```

* **But :** Visualiser tous les namespaces disponibles.

---

#### 2️⃣ **Créer un projet de test(optionnel dans sandbox)**

```bash
oc new-project tp-cli
```

* **But :** Travailler dans un espace isolé.

---

#### 3️⃣ **Déployer une application de démonstration**

```bash
oc new-app openshift/hello-openshift --name=hello-openshift
```
ou

```bash
oc new-app nginxinc/nginx-unprivileged --name=web-demo
```

---

#### 4️⃣ **Lister les ressources créées**

```bash
oc get all
```

* **But :** Voir pods, services, deployment, route.

---

#### 5️⃣ **Exposer l’application en externe**

```bash
oc expose svc/web-demo
oc get route
```

* **But :** Créer une route HTTP et récupérer l’URL publique.

---

#### 6️⃣ **Consulter les logs du pod**

```bash
oc get pods
oc logs <nom-du-pod>
```

* **But :** Analyser les logs de l’application.

---

#### 7️⃣ **Redémarrer le déploiement**

```bash
oc rollout restart deployment/web-demo
```

* **But :** Voir comment relancer un déploiement sans tout recréer.

---

#### 8️⃣ **Supprimer les ressources**

```bash
oc delete all --all
```
---
