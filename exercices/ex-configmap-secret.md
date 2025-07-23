---

## 🌟 **Exercice : Utiliser des ConfigMaps et des Secrets dans OpenShift**

### **Objectif :**

* Créer un ConfigMap et un Secret
* Les utiliser dans un déploiement pour injecter des variables d’environnement
* Vérifier que ton application lit bien ces valeurs

---

### **Étape 1 : Créer un ConfigMap**

Supposons que tu veux stocker des paramètres d’application.

```bash
oc create configmap app-config \
  --from-literal=APP_MODE=production \
  --from-literal=APP_DEBUG=false
```

---

### **Étape 2 : Créer un Secret**

Supposons que tu veux stocker un mot de passe de base de données.

```bash
oc create secret generic db-secret \
  --from-literal=DB_USER=myuser \
  --from-literal=DB_PASSWORD=myS3cretP@ss
```

---

### **Étape 3 : Déployer une application qui utilise ces valeurs**

Voici un exemple de déploiement YAML pour **Nginx** qui affiche les variables d’environnement :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-demo
  template:
    metadata:
      labels:
        app: nginx-demo
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        envFrom:
          - configMapRef:
              name: app-config
          - secretRef:
              name: db-secret
        env:
          - name: PRINT_ENV
            value: "true"
        command: ["sh", "-c"]
        args:
          - |
            if [ "$PRINT_ENV" = "true" ]; then
              env && sleep 3600
            else
              sleep 3600
            fi
```

1. Copie ce YAML dans un fichier `nginx-demo.yaml`
2. Applique-le :

   ```bash
   oc apply -f nginx-demo.yaml
   ```

---

### **Étape 4 : Vérifier que les variables sont bien injectées**

1. Liste les pods :

   ```bash
   oc get pods
   ```
2. Entre dans le pod :

   ```bash
   oc exec -it <nom_du_pod> -- sh
   ```
3. Dans le shell du pod, affiche les variables :

   ```bash
   env | grep APP_
   env | grep DB_
   ```

---

### **Modifier la valeur d’un ConfigMap**

Modifie la valeur dans le ConfigMap :

```bash
oc edit configmap app-config
```

* Change `APP_MODE` par exemple en `development`.
* Redémarre le pod pour voir la prise en compte :

  ```bash
  oc delete pod <nom_du_pod>
  ```

---