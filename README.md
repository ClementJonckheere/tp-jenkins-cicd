# TP : Jenkins et CI/CD avec Docker

## Partie 1 : Installation et Configuration de Jenkins

### 1. Installer Jenkins sur une machine Debian

```bash
# Mise à jour du système
sudo apt update && sudo apt upgrade -y

# Installation de Java (prérequis Jenkins)
sudo apt install -y openjdk-17-jdk

# Vérification de Java
java -version

# Ajout de la clé GPG Jenkins
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null

# Ajout du dépôt Jenkins
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null

# Installation de Jenkins
sudo apt update
sudo apt install -y jenkins

# Démarrage et activation au boot
sudo systemctl start jenkins
sudo systemctl enable jenkins

# Récupération du mot de passe initial
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

**Alternative : Installation via Docker (recommandée)**

```bash
# Installation de Docker
sudo apt install -y docker.io
sudo systemctl enable docker
sudo systemctl start docker

# Ajout de l'utilisateur au groupe docker
sudo usermod -aG docker $USER

# Lancement de Jenkins avec Docker
docker run -d \
  --name jenkins \
  -p 8080:8080 \
  -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  jenkins/jenkins:lts
```

Accès : http://localhost:8080

---

### 2. Créer un nouveau job dans Jenkins

1. Connectez-vous à Jenkins (http://localhost:8080)
2. Cliquez sur **"Nouveau Item"** (New Item)
3. Entrez un nom pour le job (ex: `mon-projet-nodejs`)
4. Sélectionnez **"Pipeline"**
5. Cliquez sur **OK**

---

### 3. Configurer le job pour utiliser un pipeline script à partir de SCM

Dans la configuration du job Pipeline :

1. Section **"Pipeline"** → Definition : **"Pipeline script from SCM"**
2. SCM : **Git**
3. Repository URL : `https://github.com/votre-utilisateur/votre-repo.git`
4. Credentials : Ajoutez vos credentials GitHub si nécessaire
5. Branch : `*/main` ou `*/master`
6. Script Path : `Jenkinsfile`
7. **Sauvegarder**

---

### 4. Installer le plugin Docker Pipeline

1. Allez dans **"Administrer Jenkins"** → **"Gérer les plugins"**
2. Onglet **"Disponibles"**
3. Recherchez : `Docker Pipeline`
4. Cochez et cliquez sur **"Installer sans redémarrer"**
5. Plugins supplémentaires recommandés :
   - Docker
   - Docker Commons
   - CloudBees Docker Build and Publish

---

### 5. Pourquoi installer le plugin Docker Pipeline ?

| Fonctionnalité | Description |
|----------------|-------------|
| **Isolation des builds** | Chaque build s'exécute dans un conteneur isolé |
| **Reproductibilité** | Environnement identique à chaque exécution |
| **Syntaxe DSL** | Utilisation de `docker.image()`, `docker.build()` dans les pipelines |
| **Agents dynamiques** | Création d'agents Docker à la volée |
| **Gestion des images** | Construction, push et pull d'images Docker |
| **Multi-environnement** | Tests sur différentes versions (Node 18, 20, 22...) |

---

## Partie 2 : Configuration Avancée

### 1. Configurer un webhook sur GitHub

**Sur GitHub :**

1. Allez dans votre repository → **Settings** → **Webhooks**
2. Cliquez sur **"Add webhook"**
3. Configuration :
   - **Payload URL** : `http://VOTRE_IP_JENKINS:8080/github-webhook/`
   - **Content type** : `application/json`
   - **Secret** : (optionnel, mais recommandé)
   - **SSL verification** : Selon votre config

**Sur Jenkins :**

1. Installez le plugin **"GitHub Integration"**
2. Dans la config du job → **"Build Triggers"**
3. Cochez **"GitHub hook trigger for GITScm polling"**

---

### 2. Événements à configurer dans le webhook

| Événement | Utilisation |
|-----------|-------------|
| **push** | Déclenche un build à chaque push (le plus courant) |
| **pull_request** | Build sur création/mise à jour de PR |
| **create** | Création de branches/tags |
| **delete** | Suppression de branches |

**Configuration recommandée :** Sélectionnez **"Just the push event"** pour commencer.

---

### 3 & 4. Pipeline Jenkins avec Jenkinsfile pour Node.js

Voir le fichier `Jenkinsfile` dans ce projet.

---

### 5 & 6. Tests unitaires Node.js

Voir les fichiers :
- `app/app.js` - Application exemple
- `app/app.test.js` - Tests unitaires avec Jest
- `app/package.json` - Configuration npm

---

### 7. Configurer Jenkins pour utiliser des agents Docker

**Méthode 1 : Dans le Jenkinsfile (recommandée)**

```groovy
pipeline {
    agent {
        docker {
            image 'node:20-alpine'
            args '-v /tmp:/tmp'
        }
    }
    // ...
}
```

**Méthode 2 : Configuration globale**

1. **Administrer Jenkins** → **Gérer les nœuds et clouds**
2. **Configurer les clouds** → **Ajouter un nouveau cloud** → **Docker**
3. Configuration :
   - Docker Host URI : `unix:///var/run/docker.sock`
   - Enabled : ✓
4. **Docker Agent templates** :
   - Labels : `docker-agent`
   - Docker Image : `jenkins/agent:latest`
   - Remote File System Root : `/home/jenkins`

---

### 8. Pourquoi utiliser Docker comme agent pour les builds ?

| Avantage | Explication |
|----------|-------------|
| **Isolation complète** | Pas de conflit entre projets |
| **Environnement propre** | Chaque build démarre from scratch |
| **Scalabilité** | Agents créés/détruits à la demande |
| **Multi-versions** | Test sur Node 18, 20, 22 simultanément |
| **Sécurité** | Conteneurs éphémères, pas de résidus |
| **Économie de ressources** | Pas de VM permanentes |
| **Reproductibilité** | Même environnement dev/CI/prod |

---

### 9. Gestion des problèmes sur les nœuds Jenkins

**Problèmes courants et solutions :**

```bash
# 1. Agent hors ligne
# Vérifier la connexion
docker ps -a | grep jenkins

# 2. Espace disque insuffisant
docker system prune -af
docker volume prune -f

# 3. Conteneur bloqué
docker stop $(docker ps -q --filter "label=jenkins")
docker rm $(docker ps -aq --filter "label=jenkins")
```

**Dashboard Jenkins :**

1. **Administrer Jenkins** → **Gérer les nœuds**
2. Vérifiez l'état de chaque agent
3. Cliquez sur un nœud → **Log** pour diagnostiquer

**Bonnes pratiques :**

- Configurez des **health checks**
- Activez le **monitoring** (plugin Prometheus)
- Définissez des **timeouts** pour les builds
- Utilisez des **labels** pour router les jobs
- Nettoyez régulièrement les workspaces

---

## Commandes utiles

```bash
# Vérifier le statut Jenkins
sudo systemctl status jenkins

# Logs Jenkins
sudo journalctl -u jenkins -f

# Redémarrer Jenkins
sudo systemctl restart jenkins

# Vérifier Docker
docker info
docker ps

# Nettoyer Docker
docker system prune -af
```
