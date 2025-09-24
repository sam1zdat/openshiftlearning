Ce guide vous aidera à utiliser des pipelines CI/CD dans OpenShift, spécifiquement pour construire et déployer une application Spring Boot.

## 1. Configurer un projet OpenShift

1. **Créer un nouveau projet(non disponible dans sandbox)** :
   ```sh
   oc new-project ci-cd-demo
   ```

## 2. Installer Tekton

Tekton est généralement préinstallé dans OpenShift. Si ce n'est pas le cas, vous pouvez l'installer en suivant ces étapes :

1. **Installer Tekton Pipelines(non disponible dans sandbox)** :
   ```sh
   oc apply -f https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml
   ```

2. **Vérifier l'installation** :
   ```sh
   oc get pods -n tekton-pipelines
   ```

## 3. Créer des ressources Tekton

### Créer une Task pour cloner un dépôt Git avec nettoyage

1. **Créer un fichier YAML pour la Task** (`git-clone-task.yaml`) :
   ```yaml
   apiVersion: tekton.dev/v1
   kind: Task
   metadata:
     name: git-clone
     namespace: <remplacer par votre namespace>
   spec:
     workspaces:
     - name: output
     params:
     - name: url
       type: string
     - name: revision
       type: string
       default: "main"
     steps:
     - name: clean
       image: alpine
       script: |
         rm -rf $(workspaces.output.path)/*
     - name: clone
       image: alpine/git
       script: |
         git clone $(params.url) $(workspaces.output.path)
         cd $(workspaces.output.path)
         git checkout $(params.revision)
   ```

   **Description** : Cette Task clone un dépôt Git dans un répertoire de travail et nettoie le répertoire avant de cloner pour éviter les conflits.

2. **Appliquer la Task** :
   ```sh
   oc apply -f git-clone-task.yaml
   ```

### Créer une Task pour construire et déployer une application Spring Boot

1. **Créer un fichier YAML pour la Task** (`build-deploy-task.yaml`) :
   ```yaml
   apiVersion: tekton.dev/v1
   kind: Task
   metadata:
     name: build-deploy
     namespace: <remplacer par votre namespace>
   spec:
     workspaces:
     - name: source
     steps:
     - name: build
       image: docker.io/library/maven:latest
       script: |
         cd $(workspaces.source.path)
         mvn clean package
     - name: deploy
       image: docker.io/library/alpine:latest
       script: |
         echo "Déploiement de l'application Spring Boot"
   ```

   **Description** : Cette Task utilise Maven pour construire une application Spring Boot et affiche un message de déploiement.

2. **Appliquer la Task** :
   ```sh
   oc apply -f build-deploy-task.yaml
   ```

### Créer un Pipeline

1. **Créer un fichier YAML pour le Pipeline** (`simple-pipeline.yaml`) :
   ```yaml
   apiVersion: tekton.dev/v1
   kind: Pipeline
   metadata:
     name: simple-pipeline
     namespace: <remplacer par votre namespace>
   spec:
     workspaces:
     - name: shared-data
     params:
     - name: git-url
       type: string
     tasks:
     - name: clone
       taskRef:
         name: git-clone
       params:
       - name: url
         value: $(params.git-url)
       workspaces:
       - name: output
         workspace: shared-data
     - name: build-deploy
       runAfter: ["clone"]
       taskRef:
         name: build-deploy
       workspaces:
       - name: source
         workspace: shared-data
   ```

   **Description** : Ce Pipeline orchestres les Tasks pour cloner un dépôt Git et construire/déployer une application Spring Boot.

3. **Appliquer le Pipeline** :
   ```sh
   oc apply -f simple-pipeline.yaml
   ```

## 4. Configurer un dépôt Git

1. **Créer un dépôt Git simple** :
   - Utilisez le dépôt Spring PetClinic : `https://github.com/spring-projects/spring-petclinic.git`.

## 5. Créer et exécuter un PipelineRun

1. **Créer un fichier YAML pour le PipelineRun** (`pipeline-run.yaml`) :
   ```yaml
   apiVersion: tekton.dev/v1
   kind: PipelineRun
   metadata:
     name: simple-pipeline-run
     namespace: <remplacer par votre namespace>
   spec:
     pipelineRef:
       name: simple-pipeline
     params:
     - name: git-url
       value: "https://github.com/spring-projects/spring-petclinic.git"
     workspaces:
     - name: shared-data
       volumeClaimTemplate:
         spec:
           accessModes: ["ReadWriteOnce"]
           resources:
             requests:
               storage: 1Gi
   ```

   **Description** : Ce PipelineRun exécute le Pipeline avec le dépôt Spring PetClinic et utilise un volume persistant pour le stockage.

2. **Appliquer le PipelineRun** :
   ```sh
   oc apply -f pipeline-run.yaml
   ```

## 6. Vérifier les résultats

1. **Vérifier les logs du PipelineRun** :
   ```sh
   oc logs -f simple-pipeline-run-build-deploy-pod -n ci-cd-demo
   ```

2. **Vérifier les ressources créées** :
   ```sh
   oc get pods -n ci-cd-demo
   ```

## Conclusion

Vous avez maintenant configuré un lab simple pour utiliser des pipelines CI/CD dans OpenShift, spécifiquement pour construire et déployer une application Spring Boot. 
Ce lab inclut une Task pour cloner un dépôt Git, une Task pour construire et déployer une application Spring Boot, et un Pipeline pour orchestrer ces Tasks.
