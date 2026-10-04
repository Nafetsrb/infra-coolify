# Coolify sur un VPS exposé à Internet

Ce guide installe Coolify sur un VPS dont les ports 80 et 443 reçoivent Internet directement. Il n’y a pas de WireGuard, pas de HAProxy, et pas de protocole PROXY.

Le fichier `docker-compose.yml` reste une référence de comportement. Il ne faut pas le lancer : Coolify fournit son propre Traefik. Le guide de la machine de la maison est [`README.md`](README.md). Les deux lignes de `examples/coolify-proxy-command.example.yml` ne s’appliquent pas ici.

Installation déjà faite sur ce VPS :

- adresse publique : `141.94.22.25`
- zone DNS : `ftcg.fr`, chez OVH
- tableau de bord : `https://coolify.ftcg.fr`
- port SSH du VPS : `56764`
- utilisateur SSH utilisé par Coolify : `root`

## Ce qu’on reprend du Traefik actuel

Le compose de référence fait quatre choses qui restent valables quand le trafic arrive en direct :

- écouter le TCP 80 et le TCP 443 ;
- laisser le port 80 ouvert pour le défi HTTP-01 de Let’s Encrypt ;
- servir les sites en HTTPS, avec redirection depuis HTTP ;
- ignorer les conteneurs Docker qui ne se déclarent pas au proxy (`exposedbydefault=false`).

Coolify fait déjà cela. Ses entrypoints s’appellent `http` et `https`, son resolver s’appelle `letsencrypt`, et son réseau Docker s’appelle `coolify`. Les noms `web`, `websecure`, `myresolver` et le réseau `proxy` appartiennent à l’ancien conteneur.

## Ce qu’on ne reprend pas

Ces réglages n’existent que pour le chemin VPS → WireGuard → maison :

- `--entrypoints.web.proxyProtocol.trustedIPs=10.10.0.1/32`
- `--entrypoints.websecure.proxyProtocol.trustedIPs=10.10.0.1/32`
- l’adresse `10.10.0.2`, le MTU, le keepalive et le fichier `wg0.conf`

Sur un VPS exposé, la connexion TCP vient du visiteur. Son adresse est déjà la bonne. Ajouter une IP de confiance PROXY ferait lire un en-tête qui n’est pas envoyé.

La redirection globale de tout le port 80, écrite dans l’ancien compose, ne se recopie pas. Coolify sonde `http://localhost:80/ping`. Une redirection de tout le port 80 enverrait cette sonde vers HTTPS. Chaque domaine déclaré en `https://` est redirigé par Coolify, et le tableau de bord l’est par son propre réglage.

Le compose de référence n’ouvre pas l’UDP 443. Coolify, lui, active HTTP/3 et publie `443/udp`. Sur un VPS joignable directement, ce réglage peut rester. Coolify le réécrit lorsqu’il régénère le proxy.

Coolify publie aussi le port `8080` pour son proxy, avec l’API Traefik fermée (`api.insecure=false`). Rien ne doit pointer le public vers ce port.

## Préparer Ubuntu

Utilise une version Ubuntu LTS acceptée par le script officiel. La page [Installation Coolify](https://coolify.io/docs/get-started/installation) indique les versions prises en charge. Le script actuel vise les LTS 20.04, 22.04 et 24.04. Sur une LTS plus récente refusée par le script, suis l’installation manuelle de cette même page.

Le VPS est neuf, joignable en SSH, avec une sortie Internet. Avant d’installer, prends un instantané chez l’hébergeur si tu en as un : c’est le retour arrière le plus simple.

En root, ou avec un compte qui peut utiliser `sudo` :

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl ca-certificates
docker --version
snap list docker
```

`docker --version` peut répondre que Docker est absent : le script Coolify l’installe. Si `snap list docker` montre un Docker installé par Snap, retire-le avant de continuer, avec la méthode indiquée sur la page d’installation Coolify.

Repère l’adresse publique :

```bash
curl -4 ifconfig.me
echo
```

Sur ce VPS, l’adresse attendue est `141.94.22.25`. Si la commande affiche une autre adresse que celle de la fiche hébergeur, retiens l’adresse de la fiche : c’est celle que le DNS devra viser.

## Pare-feu

Ce VPS OVH n’a pas de pare-feu réseau activé. Aucun port n’a été ouvert à la main, et 80, 443 comme 8000 répondent quand même : OVH laisse passer le trafic vers les ports qu’un programme écoute.

UFW n’a pas été activé non plus. S’il l’est un jour, ouvre d’abord le port SSH réel, ici `56764`, avant `ufw enable`. Une règle UFW ne suffit pas toujours pour un port publié par Docker : Docker ajoute ses propres règles iptables.

Le port à retirer plus tard est `8000`. La méthode est dans la section [Garder le tableau de bord sur le HTTPS](#garder-le-tableau-de-bord-sur-le-https), pas dans une règle OVH.

## Changer le port SSH

OVH, dans le guide « How to secure a VPS », fait quitter le port `22` parce que les scans automatiques le visent en premier. Le port de remplacement se choisit entre `49152` et `65535`. Sur ce VPS, c’est `56764`.

Fais ce changement avant d’ouvrir Coolify. Une connexion par clé SSH doit déjà fonctionner. Garde cette session ouverte jusqu’au bout : si le nouveau port ne répond pas, elle permet encore de revenir en arrière. OVH indique le mode rescue du VPS si cette session a été fermée trop tôt.

Le guide fait éditer deux fichiers, puis relancer le socket. Les deux fichiers doivent contenir `56764`. Coolify a affiché « Server is not reachable » quand son formulaire ne visait pas le port réellement ouvert.

### 1. Dire à sshd d’utiliser le nouveau port

```bash
sudo nano /etc/ssh/sshd_config
```

Repère ces lignes :

```text
#Port 22
#AddressFamily any
#ListenAddress 0.0.0.0
```

Retire le `#` devant `Port` et remplace `22` :

```text
Port 56764
#AddressFamily any
#ListenAddress 0.0.0.0
```

Enregistre et quitte. Ce fichier est celui que lit sshd. Sur Ubuntu 24.04, il ne suffit pas : le port est ouvert par systemd avant que sshd démarre.

### 2. Dire au socket systemd d’ouvrir ce port

```bash
sudo nano /lib/systemd/system/ssh.socket
```

Le bloc `[Socket]` ressemble à l’un de ces deux modèles. Dans les deux, chaque `ListenStream` doit porter `56764`. L’exemple OVH laisse parfois `[::]:22` : cette ligne laisserait l’IPv6 sur l’ancien port.

```text
[Socket]
ListenStream=56764
Accept=no
```

```text
[Socket]
ListenStream=0.0.0.0:56764
ListenStream=[::]:56764
BindIPv6Only=ipv6-only
Accept=no
FreeBind=yes
```

Enregistre, recharge systemd, puis redémarre le socket. Ce sont les commandes du guide pour Ubuntu 24.04 :

```bash
sudo systemctl daemon-reload
sudo systemctl restart ssh.socket
```

`systemctl restart ssh` ou `systemctl restart sshd`, sans le `daemon-reload` et sans le redémarrage de `ssh.socket`, laisse l’écoute sur `22`. C’est pour ça que sshd peut annoncer un port et que la connexion continue d’arriver sur l’autre.

Si UFW est actif, OVH fait autoriser le nouveau port avant ce redémarrage. Sur ce VPS, UFW n’est pas activé : rien à ouvrir.

### 3. Vérifier sans fermer l’ancienne session

Dans un second terminal, depuis ta machine :

```bash
ssh -p 56764 ubuntu@141.94.22.25
```

Cette connexion doit aboutir. Ensuite seulement, ferme l’ancienne session qui passait par le port `22`.

Sur le VPS, les deux contrôles doivent afficher `56764` :

```bash
sudo sshd -T | grep -i '^port '
sudo ss -tlnp | grep ssh
```

`sshd -T` montre le port lu dans `sshd_config`. `ss` montre le port ouvert par `ssh.socket`. S’ils diffèrent, corrige le fichier qui ne contient pas `56764`, relance `daemon-reload` et `restart ssh.socket`, puis reteste.

Si Fail2ban est installé, OVH fait remplacer `port = ssh` par le numéro réel dans `/etc/fail2ban/jail.local`, section `[sshd]`, sinon il surveille encore le port `22` :

```text
[sshd]
enabled = true
port = 56764
```

Puis :

```bash
sudo systemctl restart fail2ban
```

## Installer Coolify

La méthode officielle recommandée est le script automatique, lancé en root :

```bash
curl -fsSL https://cdn.coollabs.io/coolify/install.sh | sudo bash
```

À la fin, le terminal affiche l’adresse du tableau de bord. Ouvre-la depuis ta machine :

```text
http://141.94.22.25:8000
```

Crée le compte administrateur tout de suite. Tant que cette page est ouverte, la première personne qui l’atteint devient administrateur.

Copie le fichier qui contient les secrets de l’instance, dont `APP_KEY`, hors de ce dépôt Git :

```bash
sudo cp /data/coolify/source/.env /root/coolify-source.env
sudo chmod 600 /root/coolify-source.env
```

Garde une seconde copie ailleurs que sur le VPS. Sans ce fichier, une restauration ne peut pas relire la base Coolify.

## Dire à Coolify de piloter cette machine

Coolify tourne dans un conteneur. Ce conteneur n’a pas le shell d’Ubuntu. Pour lancer Traefik et déployer, il ouvre une session SSH vers la machine qui l’héberge, sur `host.docker.internal`. Ce n’est pas une connexion qui vient d’Internet. La clé autorise seulement ce conteneur.

Dans l’assistant, choisis **This machine**. Le bouton **New Server** sert à ajouter une autre machine. `localhost`, dans **Servers**, est déjà ce VPS.

Renseigne :

- le port `56764`, celui écrit dans `/etc/ssh/sshd_config` et dans `/lib/systemd/system/ssh.socket` ;
- l’utilisateur `root`.

Si le formulaire garde le port `22`, Coolify dit que le serveur n’est pas atteignable : sshd n’écoute plus là.

Le compte `ubuntu` est proposé par l’image du VPS, et Coolify le marque comme expérimental. Avec cet utilisateur, l’écran peut afficher le serveur comme joignable, puis le proxy reste bloqué sur **Starting**. Le démarrage de Traefik est passé en mettant `root`.

La clé publique que Coolify a générée est dans :

```text
/data/coolify/ssh/keys/id.root@host.docker.internal.pub
```

Elle doit être une ligne de `/root/.ssh/authorized_keys`. Ajoute-la sans effacer la clé qui sert à ta propre session SSH :

```bash
sudo mkdir -p /root/.ssh
sudo chmod 700 /root/.ssh
sudo cat /data/coolify/ssh/keys/id.root@host.docker.internal.pub | sudo tee -a /root/.ssh/authorized_keys
sudo chmod 600 /root/.ssh/authorized_keys
```

Garde la session SSH ouverte tant que Coolify n’a pas validé la connexion.

L’écran **Create "My First Project"** crée seulement un projet vide. Rien n’est déployé. **Skip setup** termine l’assistant sans ce projet. Le créer tout de suite suffit.

Sur **Servers**, la ligne `localhost` peut afficher **Attention required** tant que le proxy n’est pas démarré. **Sources**, **Destinations**, **S3 Storage** et **Shared Variables** restent vides à ce stade.

## Laisser le proxy tel que Coolify l’écrit

Ouvre **Servers → localhost → Proxy**. Le type reste **Traefik**.

**Switch Proxy** sert à remplacer Traefik par un autre proxy. Coolify répond `The running proxy must be stopped before switching` si tu cliques dessus pendant que Traefik tourne ou démarre. Ferme ce message et reste sur Traefik.

La configuration générée est le bon réglage pour un trafic direct. N’y ajoute pas les lignes `proxyProtocol.trustedIPs`. N’y remets pas les noms `web` et `websecure`.

Vérifie seulement que ces éléments sont présents :

- les ports `80:80` et `443:443` ;
- `--entrypoints.http.address=:80` ;
- `--entrypoints.https.address=:443` ;
- `--certificatesresolvers.letsencrypt.acme.httpchallenge=true` ;
- `--certificatesresolvers.letsencrypt.acme.httpchallenge.entrypoint=http` ;
- `--providers.docker.exposedbydefault=false`.

Le resolver généré n’embarque pas l’adresse de contact écrite dans l’ancien `docker-compose.yml`. Les certificats sont émis quand même. Une ligne `--certificatesresolvers...email` ajoutée à la main est retirée si Coolify régénère le fichier.

Si le statut reste sur **Starting** sans nouveau journal, le conteneur n’a pas été créé. Avec l’utilisateur `root` déjà en place sur `localhost`, tu peux le démarrer depuis la session SSH :

```bash
sudo docker compose -f /data/coolify/proxy/docker-compose.yml up -d
sudo docker ps --filter name=coolify-proxy --format '{{.Names}} {{.Status}} {{.Ports}}'
sudo ss -lntup | grep -E ':80|:443'
```

`coolify-proxy` doit être `Up`, et `ss` doit montrer 80 et 443. Les journaux de l’interface sont dans **Servers → Proxy → Logs**.

Si un certificat ne s’écrit pas :

```bash
sudo chmod 600 /data/coolify/proxy/acme.json
```

## Raccorder le domaine

Chez OVH, l’entrée `@` de type A ne concerne que `ftcg.fr`, le nom sans préfixe. `www` est une autre ligne. Aucune des deux ne crée `coolify.ftcg.fr`.

Deux façons d’envoyer les noms vers `141.94.22.25` :

- une entrée dont le sous-domaine est `coolify`, type A ;
- une entrée `*`, type A, qui envoie tous les sous-domaines d’un seul niveau vers la même adresse.

L’étoile est celle qui a été ajoutée. Traefik choisit ensuite le conteneur d’après le nom demandé. Un nom absent de Coolify arrive sur le proxy, et aucun conteneur ne répond pour lui. L’étoile ne remplace pas `@`, `www`, les MX ni le FTP.

Coolify refuse l’URL tant que le nom ne résout pas. Le message est `Validating DNS failed. Required DNS record type A pointing to 141.94.22.25`. `Instance settings updated successfully` veut dire que ce contrôle DNS a réussi. Le site peut encore afficher « La connexion a échoué » : à ce moment-là, les ports 80 et 443 n’acceptaient aucune connexion, parce que Traefik n’était pas démarré.

Vérifie le nom avant d’enregistrer :

```bash
nslookup coolify.ftcg.fr 1.1.1.1
```

La réponse doit être `141.94.22.25`. N’ajoute un `AAAA` que si ce VPS a vraiment une IPv6 dont le port 80 est ouvert. Un `AAAA` vers une adresse qui ne répond pas fait échouer le défi Let’s Encrypt.

Dans **Settings → Configuration → General** :

- **URL** : `https://coolify.ftcg.fr`
- **Redirect HTTP to HTTPS** : activé

Laisse **Instance public IPv4** vide si Coolify a détecté `141.94.22.25`. Remplis ce champ seulement quand la détection affiche une autre adresse.

Enregistre une fois Traefik démarré. Le certificat se demande alors tout seul.

## Vérifier

Depuis ta machine :

```bash
curl -sI http://coolify.ftcg.fr
curl -sI https://coolify.ftcg.fr
```

La première commande doit renvoyer vers `https://coolify.ftcg.fr` avec un code `307`. La seconde doit répondre `302` vers `/login`, avec un certificat Let’s Encrypt. Le premier essai peut encore montrer un certificat provisoire le temps du défi HTTP. S’il ne part pas, regarde **Proxy → Logs**. Évite les redémarrages en boucle : Let’s Encrypt limite les demandes répétées.

Sur le VPS :

```bash
sudo ss -lntup | grep -E ':80|:443'
```

Dans les journaux du proxy, l’adresse d’une visite faite depuis ta connexion doit être ton IP publique, pas `10.10.0.1` et pas une adresse de réseau Docker.

## Garder le tableau de bord sur le HTTPS

`http://coolify.ftcg.fr` est déjà renvoyé vers `https://coolify.ftcg.fr`.

`http://141.94.22.25:8000` ouvre encore le même tableau de bord, en HTTP, parce que Coolify publie le port `8000` sur toutes les interfaces. L’interface n’a pas de bouton pour arrêter cette publication. Le pare-feu OVH n’y change rien tant qu’il n’est pas activé.

Pour que ce port ne réponde plus que sur la machine, crée `/data/coolify/source/docker-compose.custom.yml` avec le contenu de [`examples/coolify-localhost-port.example.yml`](examples/coolify-localhost-port.example.yml) :

```yaml
services:
  coolify:
    ports:
      - "127.0.0.1:8000:8080"
```

Puis, en gardant la session SSH ouverte :

```bash
sudo bash /data/coolify/source/upgrade.sh
```

Après la commande, `https://coolify.ftcg.fr` doit s’ouvrir. `http://141.94.22.25:8000` ne doit plus répondre depuis l’extérieur. Sur le VPS, `http://127.0.0.1:8000` reste un accès de secours si Traefik est arrêté.

Les ports 80 et 443 restent ouverts : c’est par eux que le domaine arrive. Une visite de l’adresse IP seule n’envoie pas le nom `coolify.ftcg.fr`, donc Traefik ne sert pas le tableau de bord pour cette visite.

## Essayer une première application

Avec l’entrée `*`, un nouveau nom comme `test.ftcg.fr` pointe déjà vers `141.94.22.25`. Sans cette étoile, il faut une entrée A pour ce nom-là.

Dans le projet vide créé au début, déploie une application dont le domaine est :

```text
https://test.ftcg.fr
```

Le port écrit dans ce domaine est le port interne du conteneur. Les visiteurs passent par le port 443 du proxy.

Attends le certificat, puis :

```bash
curl -sI https://test.ftcg.fr
```

Ouvre le site dans un navigateur. Quand il répond, les autres services peuvent être ajoutés un par un, chacun avec son domaine en `https://`.

## Revenir en arrière

Sur un VPS neuf, l’instantané pris avant l’installation remet la machine à vide.

Si un nom de domaine a déjà été déplacé vers ce VPS et que le résultat ne convient pas, remets son enregistrement `A` vers l’ancienne adresse. Le trafic quitte ce Coolify dès que le DNS est repris par les visiteurs. Coolify et ses certificats peuvent rester en place le temps de comprendre l’échec. Le fichier `/root/coolify-source.env` sert si tu réinstalles Coolify et que tu dois relire l’ancienne base.

Pour rendre le port `8000` public à nouveau, retire `docker-compose.custom.yml` et relance `upgrade.sh`.
