# Exercice Pratique : Création d'un Registre d'Entreprise et Autorisation d'Accès au Registre OpenShift

## Module : Publication d'Images de Conteneurs d'Entreprise

**Durée estimée :** 2 heures  
**Niveau :** Débutant à Intermédiaire  
**Type :** Travaux pratiques guidés

---

## Objectifs Pédagogiques

À l'issue de cet exercice, l'apprenant sera en mesure de :

1. **Comprendre les concepts fondamentaux** d'un registre d'images de conteneurs d'entreprise
2. **Créer et configurer** un registre d'images privé dans OpenShift
3. **Mettre en place des autorisations d'accès** appropriées pour sécuriser le registre
4. **Publier et gérer** des images de conteneurs dans le registre d'entreprise
5. **Configurer l'authentification** et les politiques de sécurité pour l'accès au registre

## Prérequis

Avant de commencer cet exercice, assurez-vous de disposer des éléments suivants :

### Connaissances Techniques
- Connaissance de base des conteneurs Docker
- Familiarité avec les concepts Kubernetes
- Compréhension des principes de base d'OpenShift
- Notions de sécurité et d'authentification

### Environnement Technique
- Accès à un cluster OpenShift (version 4.x ou supérieure)
- Outil en ligne de commande `oc` installé et configuré
- Accès administrateur ou privilèges suffisants pour créer des projets
- Docker ou Podman installé sur votre poste de travail
- Connexion Internet pour télécharger des images de base

### Ressources Nécessaires
- Une image de conteneur simple à utiliser comme exemple (nginx, httpd, ou une application personnalisée)
- Fichier Dockerfile pour construire une image personnalisée
- Accès aux documentations OpenShift et Red Hat

---

## Contexte et Enjeux

Dans un environnement d'entreprise, la gestion des images de conteneurs représente un défi majeur pour les équipes de développement et d'exploitation. Un registre d'entreprise permet de centraliser, sécuriser et contrôler la distribution des images de conteneurs au sein de l'organisation.

### Pourquoi un Registre d'Entreprise ?

Un registre d'images d'entreprise offre plusieurs avantages critiques pour les organisations modernes qui adoptent la conteneurisation. Premièrement, il permet un contrôle centralisé de toutes les images utilisées dans l'environnement de production, garantissant ainsi la cohérence et la standardisation des déploiements. Cette centralisation facilite également la mise en place de politiques de sécurité uniformes et la traçabilité des versions d'images déployées.

Deuxièmement, un registre privé améliore considérablement la sécurité en permettant de stocker des images contenant du code propriétaire ou des configurations sensibles sans les exposer sur des registres publics. Les équipes peuvent ainsi maintenir la confidentialité de leurs applications tout en bénéficiant des avantages de la conteneurisation.

Troisièmement, la performance et la disponibilité sont optimisées grâce à la proximité géographique du registre avec les clusters de déploiement, réduisant les temps de téléchargement et la dépendance aux services externes. Cette approche permet également de maintenir la continuité de service même en cas de problèmes de connectivité avec les registres publics.

### Défis de la Gestion des Autorisations

La gestion des autorisations dans un registre d'entreprise nécessite une approche structurée qui équilibre sécurité et productivité. Les équipes doivent pouvoir accéder aux images nécessaires à leur travail tout en respectant les principes de moindre privilège et de séparation des responsabilités.

Les défis incluent la définition de rôles appropriés pour différents types d'utilisateurs (développeurs, testeurs, administrateurs), la mise en place de politiques de lecture et d'écriture granulaires, et l'intégration avec les systèmes d'authentification existants de l'entreprise. Il est également crucial de maintenir un audit trail complet des accès et modifications pour répondre aux exigences de conformité et de sécurité.

---

## Partie 1 : Préparation de l'Environnement

### Étape 1.1 : Vérification de l'Accès OpenShift

Commencez par vérifier que votre environnement OpenShift est correctement configuré et accessible.

```bash
# Vérifier la connexion au cluster OpenShift
oc whoami
oc cluster-info

# Lister les projets accessibles
oc projects

# Vérifier les permissions actuelles
oc auth can-i create projects
oc auth can-i create imagestreams
```

**Questions de réflexion :**
- Quels sont vos privilèges actuels dans le cluster ?
- Avez-vous les permissions nécessaires pour créer un registre ?

### Étape 1.2 : Création du Projet pour le Registre

Créez un nouveau projet dédié au registre d'entreprise. Cette séparation logique permet une meilleure organisation et gestion des ressources.

```bash
# Créer un nouveau projet pour le registre
oc new-project enterprise-registry --display-name="Registre d'Entreprise" --description="Registre privé pour les images de conteneurs d'entreprise"

# Vérifier la création du projet
oc project enterprise-registry
oc status
```

### Étape 1.3 : Configuration des Variables d'Environnement

Définissez les variables d'environnement qui seront utilisées tout au long de l'exercice pour maintenir la cohérence et faciliter la réutilisation des commandes.

```bash
# Variables pour le registre
export REGISTRY_PROJECT="enterprise-registry"
export REGISTRY_NAME="company-registry"
export REGISTRY_DOMAIN=$(oc get route -n openshift-image-registry -o jsonpath='{.items[0].spec.host}')

# Variables pour l'authentification
export REGISTRY_USER="registry-admin"
export REGISTRY_PASSWORD="SecurePassword123!"

# Afficher les variables configurées
echo "Projet: $REGISTRY_PROJECT"
echo "Nom du registre: $REGISTRY_NAME"
echo "Domaine: $REGISTRY_DOMAIN"
```

---

## Partie 2 : Création et Configuration du Registre d'Entreprise

### Étape 2.1 : Déploiement du Registre Docker

OpenShift inclut un registre d'images intégré, mais pour cet exercice, nous allons créer un registre dédié qui simule un environnement d'entreprise avec des configurations personnalisées.

```bash
# Créer un registre Docker privé
oc new-app docker.io/registry:2 --name=$REGISTRY_NAME

# Exposer le service du registre
oc expose service/$REGISTRY_NAME

# Vérifier le déploiement
oc get pods -l app=$REGISTRY_NAME
oc get services $REGISTRY_NAME
oc get routes $REGISTRY_NAME
```

### Étape 2.2 : Configuration du Stockage Persistant

Pour un registre d'entreprise, il est essentiel de configurer un stockage persistant pour garantir la durabilité des images stockées.

```bash
# Créer un volume persistant pour le registre
cat << EOF | oc apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: registry-storage
  namespace: $REGISTRY_PROJECT
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: gp2
EOF

# Attacher le volume au déploiement du registre
oc set volume deployment/$REGISTRY_NAME --add --type=persistentVolumeClaim --claim-name=registry-storage --mount-path=/var/lib/registry

# Vérifier la configuration du volume
oc describe deployment $REGISTRY_NAME
```

### Étape 2.3 : Configuration HTTPS et Sécurité

La sécurisation du registre est cruciale pour un environnement d'entreprise. Nous allons configurer HTTPS et les certificats appropriés en créant nos propres certificats SSL/TLS.

#### Création des Certificats SSL/TLS

Pour sécuriser notre registre d'entreprise, nous devons créer des certificats SSL/TLS. Dans un environnement de production, ces certificats seraient fournis par une autorité de certification (CA) reconnue, mais pour cet exercice, nous allons créer des certificats auto-signés.

```bash
# Créer un répertoire pour les certificats
mkdir -p ~/registry-certs
cd ~/registry-certs

# Obtenir le nom d'hôte du registre pour le certificat
REGISTRY_HOSTNAME=$(oc get route $REGISTRY_NAME -o jsonpath='{.spec.host}')
echo "Nom d'hôte du registre: $REGISTRY_HOSTNAME"

# Créer une clé privée RSA de 2048 bits
openssl genrsa -out key.pem 2048

# Vérifier la création de la clé privée
ls -la key.pem
openssl rsa -in key.pem -text -noout | head -10
```

#### Configuration du Certificat avec les Bonnes Extensions

Pour que le certificat soit accepté par les clients modernes, nous devons inclure les extensions appropriées, notamment le Subject Alternative Name (SAN).

```bash
# Créer un fichier de configuration pour le certificat
cat << EOF > registry-cert.conf
[req]
default_bits = 2048
prompt = no
default_md = sha256
distinguished_name = dn
req_extensions = v3_req

[dn]
C=FR
ST=Ile-de-France
L=Paris
O=Entreprise
OU=IT Department
CN=$REGISTRY_HOSTNAME

[v3_req]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names

[alt_names]
DNS.1 = $REGISTRY_HOSTNAME
DNS.2 = $REGISTRY_NAME
DNS.3 = $REGISTRY_NAME.$REGISTRY_PROJECT.svc.cluster.local
IP.1 = 127.0.0.1
EOF

# Afficher la configuration créée
cat registry-cert.conf
```

#### Génération du Certificat Auto-signé

```bash
# Créer une demande de certificat (CSR) et le certificat auto-signé en une seule commande
openssl req -new -x509 -key key.pem -out cert.pem -days 365 -config registry-cert.conf -extensions v3_req

# Vérifier le certificat créé
openssl x509 -in cert.pem -text -noout

# Vérifier les informations importantes du certificat
echo "=== Informations du certificat ==="
openssl x509 -in cert.pem -noout -subject
openssl x509 -in cert.pem -noout -issuer
openssl x509 -in cert.pem -noout -dates

# Vérifier les extensions SAN
echo "=== Extensions Subject Alternative Name ==="
openssl x509 -in cert.pem -noout -ext subjectAltName

# Vérifier que la clé privée correspond au certificat
echo "=== Vérification de la correspondance clé/certificat ==="
CERT_MODULUS=$(openssl x509 -noout -modulus -in cert.pem | openssl md5)
KEY_MODULUS=$(openssl rsa -noout -modulus -in key.pem | openssl md5)
echo "Certificat MD5: $CERT_MODULUS"
echo "Clé privée MD5: $KEY_MODULUS"

if [ "$CERT_MODULUS" = "$KEY_MODULUS" ]; then
    echo "✅ La clé privée correspond au certificat"
else
    echo "❌ ERREUR: La clé privée ne correspond pas au certificat"
    exit 1
fi
```

#### Création du Secret TLS dans OpenShift

```bash
# Créer le secret TLS dans OpenShift avec nos certificats
oc create secret tls registry-tls-secret \
    --cert=cert.pem \
    --key=key.pem \
    -n $REGISTRY_PROJECT

# Vérifier la création du secret
oc get secret registry-tls-secret -o yaml

# Afficher les détails du secret (sans révéler les clés)
oc describe secret registry-tls-secret
```

#### Configuration de la Route Sécurisée

```bash
# Configurer la route pour utiliser la terminaison TLS edge avec notre certificat
oc patch route $REGISTRY_NAME -p '{
    "spec": {
        "tls": {
            "termination": "edge",
            "certificate": "'$(cat cert.pem | sed ':a;N;$!ba;s/\n/\\n/g')'",
            "key": "'$(cat key.pem | sed ':a;N;$!ba;s/\n/\\n/g')'",
            "insecureEdgeTerminationPolicy": "Redirect"
        }
    }
}'

# Alternative : utiliser les certificats OpenShift par défaut si les certificats personnalisés posent problème
# oc annotate route $REGISTRY_NAME router.openshift.io/tls_termination="edge"

# Vérifier la configuration HTTPS
oc get route $REGISTRY_NAME -o jsonpath='{.spec.tls}' | jq .

# Tester la connectivité HTTPS
echo "=== Test de connectivité HTTPS ==="
REGISTRY_URL="https://$(oc get route $REGISTRY_NAME -o jsonpath='{.spec.host}')"
echo "URL du registre: $REGISTRY_URL"

# Test avec curl (accepter le certificat auto-signé pour le test)
curl -k -I $REGISTRY_URL/v2/ || echo "Le registre n'est pas encore prêt, c'est normal"
```

#### Gestion des Certificats en Production

Dans un environnement de production, vous devriez considérer les bonnes pratiques suivantes pour la gestion des certificats :

```bash
# Créer un script de renouvellement automatique des certificats
cat << 'EOF' > renew-registry-certs.sh
#!/bin/bash
# Script de renouvellement des certificats du registre

CERT_DIR="/path/to/registry-certs"
REGISTRY_PROJECT="enterprise-registry"
REGISTRY_NAME="company-registry"

# Vérifier l'expiration du certificat actuel
EXPIRY_DATE=$(openssl x509 -in $CERT_DIR/cert.pem -noout -enddate | cut -d= -f2)
EXPIRY_TIMESTAMP=$(date -d "$EXPIRY_DATE" +%s)
CURRENT_TIMESTAMP=$(date +%s)
DAYS_UNTIL_EXPIRY=$(( ($EXPIRY_TIMESTAMP - $CURRENT_TIMESTAMP) / 86400 ))

echo "Le certificat expire dans $DAYS_UNTIL_EXPIRY jours"

# Renouveler si moins de 30 jours avant expiration
if [ $DAYS_UNTIL_EXPIRY -lt 30 ]; then
    echo "Renouvellement du certificat nécessaire"
    # Ajouter ici la logique de renouvellement
fi
EOF

chmod +x renew-registry-certs.sh

# Créer une sauvegarde des certificats
mkdir -p ~/registry-certs-backup
cp cert.pem key.pem ~/registry-certs-backup/
echo "Sauvegarde des certificats créée dans ~/registry-certs-backup/"
```

#### Validation de la Configuration SSL/TLS

```bash
# Tester la configuration SSL avec différents outils
echo "=== Tests de validation SSL/TLS ==="

# Test avec OpenSSL s_client
echo "Test de connexion SSL:"
echo "Q" | openssl s_client -connect $(oc get route $REGISTRY_NAME -o jsonpath='{.spec.host}'):443 -servername $(oc get route $REGISTRY_NAME -o jsonpath='{.spec.host}') 2>/dev/null | grep -E "(subject|issuer|verify)"

# Vérifier les protocoles SSL/TLS supportés
nmap --script ssl-enum-ciphers -p 443 $(oc get route $REGISTRY_NAME -o jsonpath='{.spec.host}') 2>/dev/null | grep -A 20 "TLS"

# Afficher l'empreinte du certificat pour vérification
echo "Empreinte SHA256 du certificat:"
openssl x509 -in cert.pem -noout -fingerprint -sha256

echo "✅ Configuration HTTPS terminée avec succès"
echo "📁 Certificats sauvegardés dans: $(pwd)"
echo "🔐 Secret TLS créé: registry-tls-secret"
echo "🌐 URL sécurisée: https://$(oc get route $REGISTRY_NAME -o jsonpath='{.spec.host}')"
```

---

## Partie 3 : Gestion des Autorisations et de l'Authentification

### Étape 3.1 : Création des Comptes de Service

Les comptes de service permettent une authentification programmatique et une gestion fine des permissions.

```bash
# Créer un compte de service pour l'administration du registre
oc create serviceaccount registry-admin

# Créer un compte de service pour les utilisateurs en lecture seule
oc create serviceaccount registry-reader

# Créer un compte de service pour les développeurs
oc create serviceaccount registry-developer

# Vérifier la création des comptes de service
oc get serviceaccounts
```

### Étape 3.2 : Configuration des Rôles et Permissions

Définissez des rôles spécifiques pour différents types d'accès au registre, en suivant le principe de moindre privilège.

```bash
# Créer un rôle personnalisé pour l'administration du registre
cat << EOF | oc apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: $REGISTRY_PROJECT
  name: registry-admin-role
rules:
- apiGroups: [""]
  resources: ["pods", "services", "endpoints", "persistentvolumeclaims", "configmaps", "secrets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: ["image.openshift.io"]
  resources: ["imagestreams", "imagestreamtags"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
EOF

# Créer un rôle pour les développeurs
cat << EOF | oc apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: $REGISTRY_PROJECT
  name: registry-developer-role
rules:
- apiGroups: ["image.openshift.io"]
  resources: ["imagestreams", "imagestreamtags"]
  verbs: ["get", "list", "watch", "create", "update"]
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
EOF

# Créer un rôle en lecture seule
cat << EOF | oc apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: $REGISTRY_PROJECT
  name: registry-reader-role
rules:
- apiGroups: ["image.openshift.io"]
  resources: ["imagestreams", "imagestreamtags"]
  verbs: ["get", "list", "watch"]
EOF
```

### Étape 3.3 : Attribution des Rôles aux Comptes de Service

Liez les rôles créés aux comptes de service appropriés pour établir les permissions d'accès.

```bash
# Attribuer le rôle d'administrateur
oc create rolebinding registry-admin-binding --role=registry-admin-role --serviceaccount=$REGISTRY_PROJECT:registry-admin

# Attribuer le rôle de développeur
oc create rolebinding registry-developer-binding --role=registry-developer-role --serviceaccount=$REGISTRY_PROJECT:registry-developer

# Attribuer le rôle de lecture seule
oc create rolebinding registry-reader-binding --role=registry-reader-role --serviceaccount=$REGISTRY_PROJECT:registry-reader

# Vérifier les attributions de rôles
oc get rolebindings
oc describe rolebinding registry-admin-binding
```

### Étape 3.4 : Configuration de l'Authentification par Token

Configurez l'authentification par token pour permettre l'accès programmatique au registre.

```bash
# Obtenir le token du compte de service administrateur
ADMIN_TOKEN=$(oc serviceaccounts get-token registry-admin)

# Obtenir le token du compte de service développeur
DEVELOPER_TOKEN=$(oc serviceaccounts get-token registry-developer)

# Obtenir le token du compte de service lecteur
READER_TOKEN=$(oc serviceaccounts get-token registry-reader)

# Afficher les tokens (attention : ne pas exposer en production)
echo "Token Admin: $ADMIN_TOKEN"
echo "Token Developer: $DEVELOPER_TOKEN"
echo "Token Reader: $READER_TOKEN"

# Tester l'authentification avec le token
oc --token=$ADMIN_TOKEN get imagestreams
```

---

## Partie 4 : Publication et Gestion d'Images

### Étape 4.1 : Préparation d'une Image de Test

Créez une image simple pour tester les fonctionnalités du registre d'entreprise.

```bash
# Créer un répertoire pour l'application de test
mkdir -p ~/registry-test-app
cd ~/registry-test-app

# Créer un fichier HTML simple
cat << EOF > index.html
<!DOCTYPE html>
<html>
<head>
    <title>Application de Test - Registre d'Entreprise</title>
</head>
<body>
    <h1>Bienvenue dans l'Application de Test</h1>
    <p>Cette application démontre l'utilisation du registre d'entreprise OpenShift.</p>
    <p>Version: 1.0.0</p>
    <p>Date de build: $(date)</p>
</body>
</html>
EOF

# Créer un Dockerfile
cat << EOF > Dockerfile
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
EOF
```

### Étape 4.2 : Construction et Tag de l'Image

Construisez l'image et appliquez les tags appropriés pour le registre d'entreprise.

```bash
# Construire l'image localement
docker build -t test-app:1.0.0 .

# Obtenir l'URL du registre
REGISTRY_URL=$(oc get route $REGISTRY_NAME -o jsonpath='{.spec.host}')

# Tagger l'image pour le registre d'entreprise
docker tag test-app:1.0.0 $REGISTRY_URL/$REGISTRY_PROJECT/test-app:1.0.0
docker tag test-app:1.0.0 $REGISTRY_URL/$REGISTRY_PROJECT/test-app:latest

# Vérifier les images locales
docker images | grep test-app
```

### Étape 4.3 : Authentification et Push vers le Registre

Authentifiez-vous auprès du registre et publiez l'image.

```bash
# Se connecter au registre avec le token d'administrateur
echo $ADMIN_TOKEN | docker login $REGISTRY_URL -u registry-admin --password-stdin

# Pousser l'image vers le registre
docker push $REGISTRY_URL/$REGISTRY_PROJECT/test-app:1.0.0
docker push $REGISTRY_URL/$REGISTRY_PROJECT/test-app:latest

# Vérifier la publication dans OpenShift
oc get imagestreams
oc describe imagestream test-app
```

### Étape 4.4 : Création d'ImageStreams OpenShift

Utilisez les ImageStreams d'OpenShift pour une gestion avancée des images.

```bash
# Créer un ImageStream pour l'application de test
cat << EOF | oc apply -f -
apiVersion: image.openshift.io/v1
kind: ImageStream
metadata:
  name: enterprise-test-app
  namespace: $REGISTRY_PROJECT
  labels:
    app: enterprise-test-app
    version: "1.0.0"
spec:
  lookupPolicy:
    local: true
  tags:
  - name: "1.0.0"
    from:
      kind: DockerImage
      name: $REGISTRY_URL/$REGISTRY_PROJECT/test-app:1.0.0
    importPolicy:
      scheduled: true
  - name: latest
    from:
      kind: DockerImage
      name: $REGISTRY_URL/$REGISTRY_PROJECT/test-app:latest
    importPolicy:
      scheduled: true
EOF

# Vérifier la création de l'ImageStream
oc get imagestream enterprise-test-app
oc describe imagestream enterprise-test-app
```

---

## Partie 5 : Test des Autorisations et Sécurité

### Étape 5.1 : Test des Permissions de Lecture

Testez les permissions de lecture avec différents comptes de service.

```bash
# Tester l'accès en lecture avec le compte lecteur
oc --token=$READER_TOKEN get imagestreams

# Tenter une opération d'écriture (doit échouer)
oc --token=$READER_TOKEN delete imagestream enterprise-test-app

# Vérifier les logs d'audit
oc logs -l app=$REGISTRY_NAME --tail=20
```

### Étape 5.2 : Test des Permissions de Développeur

Testez les permissions du compte développeur.

```bash
# Tester la création d'un nouvel ImageStream avec le compte développeur
oc --token=$DEVELOPER_TOKEN create imagestream dev-test-app

# Tester l'import d'une nouvelle image
oc --token=$DEVELOPER_TOKEN import-image dev-test-app:latest --from=nginx:alpine --confirm

# Vérifier les permissions sur les ressources système (doit échouer)
oc --token=$DEVELOPER_TOKEN get nodes
```

### Étape 5.3 : Audit et Monitoring

Configurez le monitoring et l'audit des accès au registre.

```bash
# Créer un ConfigMap pour la configuration d'audit
cat << EOF | oc apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: registry-audit-config
  namespace: $REGISTRY_PROJECT
data:
  audit.yaml: |
    apiVersion: audit.k8s.io/v1
    kind: Policy
    rules:
    - level: Metadata
      resources:
      - group: "image.openshift.io"
        resources: ["imagestreams", "imagestreamtags"]
    - level: Request
      verbs: ["create", "update", "patch", "delete"]
      resources:
      - group: "image.openshift.io"
        resources: ["imagestreams"]
EOF

# Vérifier les métriques du registre
oc get pods -l app=$REGISTRY_NAME -o jsonpath='{.items[0].status.containerStatuses[0].restartCount}'
```

---

## Partie 6 : Intégration avec les Applications

### Étape 6.1 : Déploiement d'une Application depuis le Registre

Déployez une application en utilisant une image du registre d'entreprise.

```bash
# Créer un nouveau projet pour l'application
oc new-project test-deployment

# Donner accès au registre d'entreprise
oc policy add-role-to-user system:image-puller system:serviceaccount:test-deployment:default -n $REGISTRY_PROJECT

# Déployer l'application depuis le registre
oc new-app --image-stream=$REGISTRY_PROJECT/enterprise-test-app:latest --name=test-app

# Exposer l'application
oc expose service test-app

# Vérifier le déploiement
oc get pods -l app=test-app
oc get route test-app
```

### Étape 6.2 : Configuration des Pull Secrets

Configurez les secrets pour l'authentification automatique lors du pull d'images.

```bash
# Créer un secret Docker pour l'authentification
oc create secret docker-registry registry-pull-secret \
  --docker-server=$REGISTRY_URL \
  --docker-username=registry-reader \
  --docker-password=$READER_TOKEN \
  --docker-email=admin@company.com

# Lier le secret au compte de service par défaut
oc secrets link default registry-pull-secret --for=pull

# Tester le déploiement avec le secret
oc new-app --image=$REGISTRY_URL/$REGISTRY_PROJECT/test-app:latest --name=secure-test-app

# Vérifier que l'image est correctement récupérée
oc describe pod -l app=secure-test-app
```

---

## Évaluation et Questions de Réflexion

### Questions Techniques

1. **Architecture et Sécurité**
   - Expliquez les avantages d'un registre d'entreprise par rapport à l'utilisation de registres publics
   - Quels sont les risques de sécurité associés à un registre mal configuré ?
   - Comment implémenteriez-vous une stratégie de sauvegarde pour le registre ?

2. **Gestion des Permissions**
   - Décrivez le principe de moindre privilège appliqué aux autorisations du registre
   - Comment gérer les permissions pour une équipe de 50 développeurs répartis en 5 projets ?
   - Quelles sont les meilleures pratiques pour la rotation des tokens d'authentification ?

3. **Intégration et Automatisation**
   - Comment intégrer ce registre dans un pipeline CI/CD ?
   - Quelles métriques surveilleriez-vous pour assurer la performance du registre ?
   - Comment automatiser la gestion des versions d'images obsolètes ?

### Exercices d'Approfondissement

1. **Mise en Place d'une Politique de Rétention**
   - Configurez une politique automatique de suppression des images anciennes
   - Implémentez un système de tags sémantiques pour les versions

2. **Intégration avec un Système d'Authentification Externe**
   - Configurez l'intégration avec LDAP ou Active Directory
   - Mettez en place l'authentification SSO pour le registre

3. **Monitoring et Alerting**
   - Configurez Prometheus pour surveiller les métriques du registre
   - Créez des alertes pour les événements critiques (espace disque, échecs d'authentification)

### Critères d'Évaluation

| Critère | Excellent (4/4) | Bien (3/4) | Satisfaisant (2/4) | Insuffisant (1/4) |
|---------|-----------------|-------------|--------------------|--------------------|
| **Configuration du Registre** | Registre entièrement fonctionnel avec HTTPS et stockage persistant | Registre fonctionnel avec quelques éléments manquants | Registre de base opérationnel | Registre non fonctionnel |
| **Gestion des Autorisations** | Rôles granulaires et sécurisés correctement implémentés | Rôles de base avec quelques permissions manquantes | Autorisations basiques fonctionnelles | Autorisations mal configurées ou non fonctionnelles |
| **Publication d'Images** | Images publiées avec tags appropriés et ImageStreams | Images publiées avec tags de base | Images publiées sans organisation | Échec de publication |
| **Tests de Sécurité** | Tests complets des permissions et audit configuré | Tests de base des permissions | Tests partiels | Aucun test de sécurité |
| **Documentation** | Documentation complète et claire des procédures | Documentation de base avec étapes principales | Documentation minimale | Aucune documentation |

---

## Ressources Complémentaires

### Documentation Officielle
- [OpenShift Container Registry Documentation](https://docs.openshift.com/container-platform/latest/registry/index.html)
- [Red Hat Quay Documentation](https://docs.projectquay.io/)
- [Kubernetes RBAC Documentation](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)

### Outils et Utilitaires
- [Skopeo](https://github.com/containers/skopeo) - Outil pour la gestion d'images de conteneurs
- [Crane](https://github.com/google/go-containerregistry/tree/main/cmd/crane) - Outil en ligne de commande pour les registres
- [Harbor](https://goharbor.io/) - Registre d'entreprise open source

### Bonnes Pratiques
- [NIST Container Security Guide](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-190.pdf)
- [CIS Kubernetes Benchmark](https://www.cisecurity.org/benchmark/kubernetes)
- [OpenShift Security Best Practices](https://docs.openshift.com/container-platform/latest/security/index.html)

---

## Conclusion

Cet exercice vous a permis de découvrir les aspects essentiels de la création et de la gestion d'un registre d'entreprise dans OpenShift. Vous avez appris à configurer les autorisations d'accès, à publier des images de manière sécurisée, et à intégrer le registre dans un workflow de développement.

Les compétences acquises dans cet exercice sont directement applicables dans un environnement de production et constituent une base solide pour la gestion d'images de conteneurs à l'échelle de l'entreprise. La maîtrise de ces concepts est essentielle pour tout professionnel travaillant avec OpenShift et les technologies de conteneurisation.

**Prochaines étapes recommandées :**
- Approfondissement des politiques de sécurité avancées
- Intégration avec des outils de scan de vulnérabilités
- Mise en place de pipelines CI/CD automatisés
- Configuration de la haute disponibilité pour le registre

---

