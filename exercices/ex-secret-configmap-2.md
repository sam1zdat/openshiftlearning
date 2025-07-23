---

## 🚀 Exercice Complet : Manipuler ConfigMaps et Secrets dans OpenShift

### **Objectifs :**

* Créer et modifier ConfigMaps/Secrets à partir de fichiers et de valeurs littérales
* Les monter dans des pods (volumes et variables d’environnement)
* Mettre à jour leur contenu et observer le comportement côté pod
* Sécuriser l’accès aux secrets
* Nettoyer les ressources après usage

---

### **Étape 1 : Préparer les fichiers sources**

```bash
echo "application_name=demo-app" > app.properties
echo "host=prod-db.example.com" > db.properties
echo "MySuperPassword2025" > dbpassword.txt
echo "apikey=ZXhhbXBsZUFQSTI1Ng==" > apikey.txt
```

---

### **Étape 2 : Créer un ConfigMap avec plusieurs fichiers et variables**

```bash
oc create configmap app-config \
  --from-file=app.properties \
  --from-file=db.properties \
  --from-literal=LOG_LEVEL=INFO \
  --from-literal=FEATURE_FLAG=true
```

---

### **Étape 3 : Créer un Secret à partir de fichiers et de variables**

```bash
oc create secret generic app-secret \
  --from-file=dbpassword.txt \
  --from-file=apikey.txt \
  --from-literal=ADMIN_USER=admin \
  --from-literal=ADMIN_PASSWORD=admin@2025
```

---

### **Étape 4 : Déployer un pod utilisant les deux méthodes (volumes + variables)**

Crée un fichier `app-demo-pod.yaml` :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-demo
spec:
  containers:
    - name: alpine
      image: alpine
      command: ["sh", "-c", "sleep 3600"]
      envFrom:
        - configMapRef:
            name: app-config
        - secretRef:
            name: app-secret
      volumeMounts:
        - name: config-vol
          mountPath: /etc/app-config
        - name: secret-vol
          mountPath: /etc/app-secret
  volumes:
    - name: config-vol
      configMap:
        name: app-config
    - name: secret-vol
      secret:
        secretName: app-secret
```

Applique le pod :

```bash
oc apply -f app-demo-pod.yaml
```

---

### **Étape 5 : Vérifier le contenu dans le pod**

1. **Liste des variables d’environnement** :

   ```bash
   oc exec -it app-demo -- env | grep -E 'LOG_LEVEL|FEATURE_FLAG|ADMIN_'
   ```

2. **Vérifier les fichiers montés** :

   ```bash
   oc exec -it app-demo -- ls /etc/app-config
   oc exec -it app-demo -- cat /etc/app-config/app.properties
   oc exec -it app-demo -- ls /etc/app-secret
   oc exec -it app-demo -- cat /etc/app-secret/dbpassword.txt
   ```

---

### **Étape 6 : Mettre à jour le ConfigMap et observer**

1. Modifie `app.properties` localement :

   ```bash
   echo "application_name=demo-app-v2" > app.properties
   ```

2. Mets à jour le ConfigMap (attention, il faut supprimer et recréer) :

   ```bash
   oc create configmap app-config \
     --from-file=app.properties \
     --from-file=db.properties \
     --from-literal=LOG_LEVEL=DEBUG \
     --from-literal=FEATURE_FLAG=false \
     --dry-run=client -o yaml | oc apply -f -
   ```

3. Supprime puis relance le pod pour voir la prise en compte du changement :

   ```bash
   oc delete pod app-demo
   oc apply -f app-demo-pod.yaml
   ```

4. Vérifie le nouveau contenu dans le pod comme à l’étape précédente.

---

### **Étape 7 : Sécuriser et afficher les secrets**

* **Voir le contenu encodé du secret** :

  ```bash
  oc get secret app-secret -o yaml
  ```

* **Décoder un champ secret** (exemple avec la commande `base64`) :

  ```bash
  oc get secret app-secret -o jsonpath="{.data.dbpassword.txt}" | base64 -d
  ```

* **Vérifie les droits d’accès sur les fichiers montés** :

  ```bash
  oc exec -it app-demo -- ls -l /etc/app-secret
  ```

---

### **Étape 8 : Nettoyage**

```bash
oc delete pod app-demo
oc delete configmap app-config
oc delete secret app-secret
```

---