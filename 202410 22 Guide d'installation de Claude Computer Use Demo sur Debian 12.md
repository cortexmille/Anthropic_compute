# Guide d'installation de Claude Computer Use Demo sur Debian 12

> pierre.aumont@gmail.com, beercan.fr, 22 octobre 2024

## 1. Installation de Docker

```bash
# Mise à jour du système
sudo apt update
sudo apt upgrade -y

# Installation des dépendances nécessaires
sudo apt install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg \
    lsb-release

# Ajout de la clé GPG officielle de Docker
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Ajout du dépôt Docker stable
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/debian $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Mise à jour des dépôts et installation de Docker
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io

# Démarrage et activation de Docker
sudo systemctl start docker
sudo systemctl enable docker

# Ajout de l'utilisateur au groupe docker pour éviter d'utiliser sudo
sudo usermod -aG docker $USER

# Redémarrer votre session pour que les changements de groupe prennent effet
# Vous pouvez soit vous déconnecter/reconnecter, soit utiliser la commande :
newgrp docker
```

## 2. Configuration de Docker pour éviter les problèmes réseau

```bash
# Création du fichier de configuration Docker
sudo mkdir -p /etc/docker
sudo nano /etc/docker/daemon.json
```

Ajoutez ce contenu dans le fichier daemon.json :
```json
{
    "dns": ["89.2.0.1", "89.2.0.2"],
    "iptables": false,
    "ip-masq": false
}
```

```bash
# Redémarrage du service Docker pour appliquer les changements
sudo systemctl restart docker
```

## 3. Configuration de la clé API Anthropic

```bash
# Création du dossier .anthropic
mkdir -p $HOME/.anthropic

# Création du fichier de configuration avec la clé API
nano $HOME/.anthropic/config.json
```

Ajoutez ce contenu dans config.json (remplacez YOUR_API_KEY par votre vraie clé API) :
```json
{
    "api_key": "YOUR_API_KEY"
}
```

```bash
# Sécurisation du fichier de configuration
chmod 600 $HOME/.anthropic/config.json
```

## 4. Premier lancement de l'application

```bash
# Lecture de la clé API depuis le fichier de configuration
export ANTHROPIC_API_KEY=$(jq -r .api_key $HOME/.anthropic/config.json)

# Création du réseau Docker dédié
docker network create --driver bridge \
    --subnet 172.20.0.0/16 \
    --gateway 172.20.0.1 \
    --opt "com.docker.network.bridge.enable_ip_masquerade=false" \
    anthropic-network

# Lancement du conteneur
docker run \
    -e ANTHROPIC_API_KEY=$ANTHROPIC_API_KEY \
    -v $HOME/.anthropic:/home/computeruse/.anthropic \
    --network anthropic-network \
    -p 5900:5900 \
    -p 8501:8501 \
    -p 6080:6080 \
    -p 8080:8080 \
    -it ghcr.io/anthropics/anthropic-quickstarts:computer-use-demo-latest
```

## 5. Lancements ultérieurs

Pour faciliter les lancements suivants, créez un script de lancement :

```bash
# Création du script de lancement
nano $HOME/launch-claude.sh
```

Ajoutez ce contenu dans launch-claude.sh :
```bash
#!/bin/bash

# Lecture de la clé API
export ANTHROPIC_API_KEY=$(jq -r .api_key $HOME/.anthropic/config.json)

# Lancement du conteneur
docker run \
    -e ANTHROPIC_API_KEY=$ANTHROPIC_API_KEY \
    -v $HOME/.anthropic:/home/computeruse/.anthropic \
    --network anthropic-network \
    -p 5900:5900 \
    -p 8501:8501 \
    -p 6080:6080 \
    -p 8080:8080 \
    -it ghcr.io/anthropics/anthropic-quickstarts:computer-use-demo-latest
```

```bash
# Rendre le script exécutable
chmod +x $HOME/launch-claude.sh
```

Pour les lancements suivants, il suffira d'exécuter :
```bash
~/launch-claude.sh
```

## 6. Accès à l'interface

Une fois le conteneur lancé, vous pouvez accéder à l'application via :
- Interface principale : http://localhost:8080
- Interface Streamlit uniquement : http://localhost:8501
- Vue bureau uniquement : http://localhost:6080/vnc.html
- Connection VNC directe : vnc://localhost:5900

## Notes importantes

- Si vous modifiez votre clé API, modifiez simplement le fichier `$HOME/.anthropic/config.json`
- Pour arrêter proprement le conteneur, utilisez Ctrl+C dans le terminal où il est lancé
- Si vous rencontrez des problèmes de réseau, vérifiez que le réseau `anthropic-network` existe bien avec `docker network ls`
- En cas de problème avec le réseau Docker, vous pouvez le recréer avec :
  ```bash
  docker network rm anthropic-network
  docker network create --driver bridge \
      --subnet 172.20.0.0/16 \
      --gateway 172.20.0.1 \
      --opt "com.docker.network.bridge.enable_ip_masquerade=false" \
      anthropic-network
  ```