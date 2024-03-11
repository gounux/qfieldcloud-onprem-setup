# QFieldCloud on-premise setup

Dépôt pour installation et mise en route d'un QFieldCloud auto-hébergé sur machine linux

L'installation et la configuration peuvent être faites de deux manières : 
- soit via commandes linux dans le terminal
- soit via exécution de playbooks ansible

## Mise en route via commandes linux

### Installations

- installer les paquets `apt` nécessaires

```sh
apt install -y sudo git certbot pwgen pass keepass2
```

- installer `docker` en suivant [la documentation officielle](https://docs.docker.com/engine/install/debian/) résumée ici :

```sh
sudo apt-get install ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update

sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### Configuration de la machine linux

- créer l'utilisateur unix `qfc`

```sh
QFC_ID=666
adduser --shell /bin/bash --uid ${QFC_ID} --gid ${QFC_ID} qfc
```

- ajouter l'utilisateur `qfc` aux groupe `sudo` et `docker`

```sh
usermod -aG sudo qfc
usermod -aG docker qfc
```

- lancer une session sous l'utilisateur `qfc`

```sh
su qfc
```

- cloner le dépôt qfieldcloud

```sh
cd /opt
git clone --recurse-submodules https://github.com/opengisch/qfieldcloud.git
```

- basculer sur la branche de la dernière version dispo

```sh
cd /opt/qfieldcloud
git checkout -b v0.25.0
```

### Configuration du serveur QFieldCloud

- copier le fichier de configuration

```sh
cp .env.example .env
```

- modifier les variables d'environnement nécessaires du fichier `.env`, en utilisant `pwgen` pour générer des clés et mots de passe

- créer le fichier compose standalone

```sh
touch docker-compose.override.standalone.yml
```

- remplir ce fichier nouvellement créé avec [le contenu de ce fichier](https://github.com/opengisch/qfieldcloud/pull/844/files#diff-32a4168a7b1fcd63e4c0b12368085fea9e1aec07a103211409ad8538a27487b5), via `cat > docker-compose.override.standalone.yml` puis CTRL+C / CTRL+V en terminant la saisie via CTRL+D

### Démarrage et initialisation

- démarrer les services docker

```sh
docker compose up -d --build
```

- initialiser le serveur qfieldcloud

```sh
docker compose exec app python manage.py migrate
docker compose run app python manage.py collectstatic --noinput3
docker compose run app python manage.py createsuperuser --username admin --email admin@mon.domain
```

En veillant à changer l'adresse mail du super-user et en notant le mot de passe saisi, par exemple via `pass` ou dans un fichier [keepass](https://keepass.info/)

- vérifier l'état des services

```sh
docker compose exec app python manage.py status
```

### Certificats SSL

- éteindre le service du serveur web nginx

```sh
docker compose down nginx --remove-orphans
```

- récupérer un certificat [Let's Encrypt](https://letsencrypt.org/) valable 3 mois
```sh
source .env
certbot certonly --standalone -d ${QFIELDCLOUD_HOST}
```

- copier les certificats dans la conf nginx de qfieldcloud
```sh
sudo cp /etc/letsencrypt/live/${QFIELDCLOUD_HOST}/privkey.pem ./conf/nginx/certs/${QFIELDCLOUD_HOST}-key.pem
sudo cp /etc/letsencrypt/live/${QFIELDCLOUD_HOST}/fullchain.pem ./conf/nginx/certs/${QFIELDCLOUD_HOST}.pem
```

- réallumer le service du serveur web nginx

```sh
docker compose up nginx -d --build
```

## Installation via ansible

- installer ansible

```sh
apt install ansible
```

- ajouter l'hôte dans `ansible/env/hosts` et ses variables dans `ansible/env/host_vars/$HOST.yml`

- excéuter les playbooks ansible

```sh
ansible-playbook setup.yml --inventory env --limit $HOST --tags install -K
```
