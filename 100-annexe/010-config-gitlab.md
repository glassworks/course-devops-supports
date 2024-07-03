# Configuration GitLab

Ce fichier contient les instructions d'installation pour GitLab.

## Volumes

Pour ne pas être perturbé par un problème de saturation de disque, il est conseillé de monter des volumes pour le stockage GIT et l'installation docker.

Provisionnez et monter des volumes supplémentaires sur les emplacements suivants :

```
/var/opt/gitlab
/var/lib/docker
```

Suivez les instructions [ici](https://docs.glassworks.tech/unix-shell/gestion-de-la-machine/070-gestion/disques) pour monter les volumes.

## GitLab


Installer une instance avec l'appli Gitlab chez Scaleway ou bien en suivant [ce lien](https://about.gitlab.com/install/#ubuntu). Faites une connexion via SSH.

Installer le daemon de mailing :

```bash
apt update
apt install postfix
```

Se connecter à cette instance et modifier `/etc/gitlab/gitlab.rb`

```bash
nano /etc/gitlab/gitlab.rb 
```

Modifiez les entrées suivantes :

```rb
external_url 'https://gitlab.mt.glassworks.tech'

registry_external_url 'https://registry.mt.glassworks.tech'

# Optional -- sending mail through Mailjet
# gitlab_rails['smtp_enable'] = true
# gitlab_rails['smtp_address'] = "in-v3.mailjet.com"
# gitlab_rails['smtp_port'] = 587
# gitlab_rails['smtp_user_name'] = "MAILJET API KEY"
# gitlab_rails['smtp_password'] = "MAILJET SECRET KEY"
# gitlab_rails['smtp_domain'] = "in-v3.mailjet.com"
# gitlab_rails['smtp_authentication'] = "plain"
# gitlab_rails['smtp_enable_starttls_auto'] = true
# gitlab_rails['smtp_tls'] = false
# gitlab_rails['smtp_pool'] = false
# gitlab_rails['gitlab_email_from'] = 'kevin@nguni.fr'
# gitlab_rails['gitlab_email_reply_to'] = 'kevin@nguni.fr'
```

Références :

* [SMTP Sur Gitlab](https://docs.gitlab.com/omnibus/settings/smtp.html)

Ensuite, relancez GitLab :

```bash
gitlab-ctl reconfigure
gitlab-ctl restart
```

Configure le mot de passe de l'utilisateur root :

```bash
sudo gitlab-rake "gitlab:password:reset"
```

Tester l'envoi d'email avec :

```bash
gitlab-rails console

Notify.test_email('kevin@nguni.fr', 'Test message', 'Test Message Body').deliver_now
```

## Docker

D'abord, installer docker :

```bash
# Installer docker
# Désinstaller les anciennes versions de docker
sudo apt-get remove docker docker-engine docker.io containerd runc

# Mettre à jour les indexes des packages
sudo apt-get update

# Installer les packages tiers nécessaires pour ajouter Docker parmi nos indexes
sudo apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
    
# Télécharger la clé publique de Docker qui va vérifier l'authenticité
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Installer le dépot Docker parmi les sources connues de notre distribution
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo chmod a+r /etc/apt/keyrings/docker.gpg
    
# Remettre à jour nos indexes
sudo apt-get update

# Installer Docker 
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

## Docker Registry Mirror

Ajoutez le miroir pour les Docker images :

```bash
mkdir dockercache
cd dockercache
nano docker-compose.yml
```

Coller le `docker-compose.yml` suivant :

```yml
version: '3.9'

services:
  registry:
    image: registry:2
    restart: always
    ports:
      - "6000:5000"
    environment:
      - REGISTRY_PROXY_REMOTEURL=https://registry-1.docker.io
```

Lancez le miroir :

```bash
docker compose up -d
```

Ensuite, on va dire à docker où se trouve le miroir :

```bash
nano /etc/docker/daemon.json
```

Ajoutez les lignes suivantes :

```
{
  "registry-mirrors": ["http://127.0.0.1:6000"]
}
```

Relancez docker :

```bash
service docker restart

# Afficher les infos docker, il devrait y avoir "Register Mirrors"
docker info
```

## Installer les Runners

Ensuite installons le Runner :

```bash
curl -LJO "https://gitlab-runner-downloads.s3.amazonaws.com/latest/deb/gitlab-runner_amd64.deb"
dpkg -i gitlab-runner_amd64.deb

# Do the following multiple times
gitlab-runner register
# Adress: https://gitlab.mt.glassworks.tech
# Registration token: found here: https://gitlab.mt.glassworks.tech/admin/runners
# Tag: general
# Executor: docker
# Image: node:18
gitlab-runner verify
gitlab-runner start

gitlab-runner run&

```

Ensuite, on va donner l'accès privilégié pour que le Runner puisse interagir avec Docker (notamment docker-in-docker) :

```bash
nano /etc/gitlab-runner/config.toml 
```

Fixer le nombre de runners concurrents:

```
concurrent = 4
```

Modifiez chaque entrée :

```
[runners.docker]
    privileged = true
    volumes = ["/cache", "/var/run/docker.sock:/var/run/docker.sock"]
    # ...
```

Le Runner devrait se lancer automatiquement au démarrage :

```bash
crontab -e
```

Ajoute la ligne suivante :

```bash
# Restart runners at reboot
@reboot sudo gitlab-runner run

# Clean docker at 15 minutes after the hour
15 * * * * docker system prune -af --volumes
```
