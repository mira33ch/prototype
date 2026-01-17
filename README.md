
# Portfolio App – CI/CD avec Jenkins, Docker et SonarQube

Ce projet est une application **full‑stack** de portfolio avec un pipeline CI/CD complet basé sur Jenkins, Docker et SonarQube.

- Frontend : Angular (`portfolio-frontend`)
- Backend : Spring Boot (`demo1`) avec MySQL
- Qualité de code : SonarQube (backend + frontend)
- Conteneurs : Docker (images poussées sur Docker Hub)
- Scan de sécurité : Trivy sur les images Docker

---

## Architecture

L’architecture globale est la suivante :

- **Jenkins**
  - Pipeline déclaratif (Jenkinsfile)
  - Build, tests, analyse SonarQube
  - Build & push des images Docker frontend / backend
  - Scan des images avec Trivy

- **SonarQube + PostgreSQL**
  - Lancement via Docker
  - Analyses de qualité pour le backend (Java) et le frontend (Angular)

- **MySQL**
  - Conteneur `mysql-test` lancé pendant le pipeline pour les tests du backend

- **Docker Hub**
  - Registry pour publier les images :
    - `mariem360/portfolio-app-cicd-pipeline-backend`
    - `mariem360/portfolio-app-cicd-pipeline-frontend`

---

## Prérequis

- Docker et Docker Compose installés
- Java 21, Maven 3, Node 18 (ou configurés comme **tools** dans Jenkins)
- Jenkins installé et accessible
- SonarQube accessible depuis Jenkins (webhook configuré vers Jenkins)

---

## Pipeline Jenkins

Le pipeline déclaratif (`Jenkinsfile`) exécute les étapes suivantes :

1. **Start MySQL (no password)**  
   Lancement d’un conteneur MySQL (utilisé par le backend pendant les tests).

2. **Cleanup Workspace**  
   Nettoyage de l’espace de travail Jenkins.

3. **Checkout from SCM**  
   Récupération du code source depuis GitHub (branche `main`).

4. **Build Application**
   - Frontend :
     - `npm install`
     - `npm run build --prod` dans `portfolio-frontend`
   - Backend :
     - `mvn clean package -DskipTests` dans `demo1`

5. **Test Application**
   - Backend :
     - `mvn test` dans `demo1`

6. **SonarQube Analysis Backend**
   - Analyse de la partie Java via `sonar-maven-plugin` en utilisant le serveur SonarQube configuré dans Jenkins.

7. **SonarQube Frontend Analysis**
   - Analyse de la partie Angular/TypeScript via `sonar-scanner` sur le dossier `src`.

8. **Quality Gate**
   - Attente du résultat de la Quality Gate SonarQube via `waitForQualityGate`.
   - Le pipeline continue uniquement si la Quality Gate est `OK` (ou selon la configuration choisie).

9. **Build & Push Docker Images**
   - Build de l’image backend : `mariem360/portfolio-app-cicd-pipeline-backend:<TAG>`
   - Build de l’image frontend : `mariem360/portfolio-app-cicd-pipeline-frontend:<TAG>`
   - Push des images vers Docker Hub en utilisant les credentials Jenkins.

10. **Trivy Scan**
    - Scan des images Docker backend et frontend avec Trivy pour détecter les vulnérabilités HIGH / CRITICAL.

11. **Stop MySQL & Cleanup Artifacts**
    - Arrêt et suppression du conteneur MySQL utilisé pour les tests.
    - Suppression des images Docker locales utilisées pendant le pipeline (le registre Docker Hub reste intact).

---

## Configuration des credentials (Jenkins & SonarQube)

**Important :** les mots de passe, tokens et clés **ne doivent jamais être committés** dans Git. Ils sont stockés dans Jenkins sous forme de **Credentials**.

### Jenkins Credentials

Dans **Jenkins → Manage Jenkins → Manage Credentials**, créer les credentials suivants :

- **GitHub**
  - **ID** : `github`
  - **Type** : Username with password ou Personal Access Token
  - **Usage** : utilisé dans le stage **Checkout from SCM** pour cloner le repo GitHub.

- **Docker Hub**
  - **ID** : `dockerhub`
  - **Type** : Username with password
  - **Usage** :
    - Dans le bloc `environment` du Jenkinsfile :
      - `DOCKER_AUTH = credentials('dockerhub')`
    - Dans `Build & Push Docker Images` :
      - `docker.withRegistry('https://index.docker.io/v1/', 'dockerhub') { ... }`

- **SonarQube (token Jenkins)**
  - **ID** : `jenkins-sonarqube-token`
  - **Type** : Secret text
  - **Usage** :
    - Dans la stage **Quality Gate** :
      - `waitForQualityGate abortPipeline: false, credentialsId: 'jenkins-sonarqube-token'`

Les valeurs réelles (username, mot de passe, tokens) ne doivent jamais apparaître dans le Jenkinsfile ni dans ce README.

---

## Lancement de SonarQube en local (Docker)

Exemple de lancement de SonarQube + PostgreSQL en Docker (à adapter à ton environnement) :

```bash
# Base PostgreSQL pour SonarQube
docker run -d --name sonarqube-postgres \
  -e POSTGRES_USER=sonar \
  -e POSTGRES_PASSWORD=<POSTGRES_PASSWORD> \
  -e POSTGRES_DB=sonarqube \
  -p 5432:5432 \
  postgres:15

# SonarQube
docker run -d --name sonarqube \
  -e SONAR_JDBC_URL=jdbc:postgresql://sonarqube-postgres:5432/sonarqube \
  -e SONAR_JDBC_USERNAME=sonar \
  -e SONAR_JDBC_PASSWORD=<POSTGRES_PASSWORD> \
  -p 9000:9000 \
  --link sonarqube-postgres \
  sonarqube:latest
