# Coolify sur le serveur privé

Ce guide concerne la machine de la maison, derrière WireGuard. Pour un VPS joignable directement depuis Internet, suis [`README-vps.md`](README-vps.md).

Ce dépôt prépare l’installation de Coolify sur le nouveau serveur de la maison. Le fichier `docker-compose.yml` décrit le Traefik qui fonctionne aujourd’hui sur l’ancien serveur. Il sert de référence. Il ne faut pas le lancer sur la nouvelle machine, ni recréer son conteneur.

Le VPS et HAProxy restent inchangés. Le jour de la bascule, la nouvelle machine reprend l’adresse WireGuard que le VPS utilise déjà.

Les domaines ci-dessous sont des exemples. Remplace-les par les tiens :

- tableau de bord Coolify : `coolify.exemple.com`
- première application : `test.exemple.com`

## Architecture mesurée

```text
Internet → VPS 137.74.116.56 → HAProxy → WireGuard → 10.10.0.2:80 et :443 → Traefik
```

Relevé sur l’ancien serveur, interface `wg0` :

| Élément | Valeur |
|---|---|
| Adresse de cette machine | `10.10.0.2/24` |
| Adresse du VPS dans le tunnel | `10.10.0.1/32` |
| IP publique du VPS, endpoint du tunnel | `137.74.116.56:51820` |
| Port d’écoute WireGuard de la machine | `44223` |
| MTU | `1420` |
| Keepalive | `25` secondes |
| Réseau local de la maison | `192.168.1.0/24` |
| Ancien serveur sur ce réseau | `192.168.1.18` |

Un paquet arrivé sur le port 80 commence par la signature du protocole PROXY version 2. HAProxy l’envoie, Traefik ne fait confiance qu’à `10.10.0.1/32`, et Docker transmet cette adresse source jusqu’au conteneur. L’IP du visiteur vient de cet en-tête.

Le port 80 reste ouvert pour le défi HTTP-01 de Let’s Encrypt. Les sites eux-mêmes répondent en HTTPS. Le dépôt ne contient aucune route TCP, UDP, ni middleware supplémentaire.

Les clés WireGuard restent dans `/etc/wireguard/wg0.conf` sur la machine. Ce fichier ne va pas dans Git. `acme.json` non plus : il contient les clés privées des certificats.

## Ce que Coolify configure seul

Une fois un domaine en `https://` enregistré, Coolify :

- démarre le conteneur `coolify-proxy` ;
- ouvre les entrypoints `http` (`:80`) et `https` (`:443`) ;
- demande les certificats avec le défi HTTP-01 et le resolver `letsencrypt` ;
- range les certificats dans `/data/coolify/proxy/acme.json` ;
- crée la route, la redirection HTTP vers HTTPS de ce domaine, et le réseau Docker de l’application ;
- relie Traefik à ce réseau.

Les noms de l’ancien fichier ne se recopient pas : `web`, `websecure`, `myresolver` et le réseau Docker `proxy` appartiennent à l’ancien conteneur. Coolify utilise `http`, `https`, `letsencrypt` et le réseau `coolify`.

La redirection globale de tout le port 80 ne se recopie pas non plus. Coolify sonde `http://localhost:80/ping` pour savoir si le proxy est sain. Une redirection de tout le port 80 enverrait cette sonde vers HTTPS. La redirection utile est celle que Coolify pose sur chaque domaine déclaré en `https://`, plus le réglage du tableau de bord décrit plus bas.

Le fichier généré publie aussi `443/udp` et le port `8080`, et active HTTP/3. L’ancien Traefik ne le fait pas. Coolify réécrit ces lignes quand il régénère le proxy : les retirer ne tient pas. Le chemin public reste le TCP du VPS, qui ne transporte pas l’UDP 443. Il ne faut pas ouvrir 8080, 80 ou 443 sur la box Internet.

## Préparer Ubuntu

Utilise une version Ubuntu LTS acceptée par le script officiel au moment de l’installation. La page [Installation Coolify](https://coolify.io/docs/get-started/installation) indique les versions prises en charge. Le script actuel vise les LTS 20.04, 22.04 et 24.04. Sur une LTS plus récente refusée par le script, suis l’installation manuelle de cette même page.

La machine est neuve, sur le réseau de la maison, avec un accès SSH et une sortie Internet. Elle n’a pas encore WireGuard : le tunnel de l’ancien serveur doit rester seul à porter `10.10.0.2` jusqu’à la bascule.

En root, ou avec un compte qui peut utiliser `sudo` :

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl ca-certificates
docker --version
snap list docker
ip -4 addr show
```

`docker --version` peut répondre que Docker est absent : le script Coolify l’installe. Si `snap list docker` montre un Docker installé par Snap, retire-le avant de continuer, avec la méthode indiquée sur la page d’installation Coolify.

Note l’adresse `192.168.1.x` de la nouvelle machine. Elle sera différente de `192.168.1.18`.

## Installer Coolify

La méthode officielle recommandée est le script automatique, lancé en root :

```bash
curl -fsSL https://cdn.coollabs.io/coolify/install.sh | sudo bash
```

À la fin, le terminal affiche l’adresse du tableau de bord. Ouvre-la depuis la maison :

```text
http://192.168.1.x:8000
```

Crée le compte administrateur tout de suite. Tant que cette page est ouverte, la première personne qui l’atteint devient administrateur.

Copie ensuite le fichier qui contient les secrets de l’instance, dont `APP_KEY`, vers un endroit hors de ce dépôt Git :

```bash
sudo cp /data/coolify/source/.env /root/coolify-source.env
sudo chmod 600 /root/coolify-source.env
```

Garde une copie supplémentaire sur un disque qui n’est pas le serveur. Sans ce fichier, une restauration future ne peut pas relire la base Coolify.

Coolify tourne dans un conteneur. Pour agir sur la machine, ce conteneur ouvre une session SSH vers elle, sur `host.docker.internal`. Dans l’assistant, choisis **This machine**. Le serveur à garder est `localhost`. **New Server** sert à en ajouter un autre.

Le port est celui que SSH écoute vraiment sur cette machine. L’utilisateur est `root`. Sur le VPS exposé, le compte `ubuntu` a laissé le proxy bloqué sur **Starting** alors que l’écran de connexion était déjà vert. La clé publique générée par Coolify est `/data/coolify/ssh/keys/id.root@host.docker.internal.pub`. Elle doit être une ligne de `/root/.ssh/authorized_keys`, sans effacer la clé de ta session.

L’écran **Create "My First Project"** crée un projet vide. Rien n’est déployé. **Attention required** sur `localhost` veut dire que le proxy n’est pas encore démarré. **Sources**, **Destinations**, **S3 Storage** et **Shared Variables** restent vides.

## Régler le proxy

1. Dans Coolify, ouvre **Servers**, puis le serveur local, puis **Proxy**. Le type reste **Traefik**. **Switch Proxy** sert à le remplacer : Coolify répond `The running proxy must be stopped before switching` si tu cliques dessus pendant qu’il tourne. Reste sur Traefik.
2. Démarre le proxy s’il est arrêté. Coolify écrit alors `/data/coolify/proxy/docker-compose.yml`. S’il reste sur **Starting** sans journal, vérifie que `localhost` utilise `root`, puis lance `sudo docker compose -f /data/coolify/proxy/docker-compose.yml up -d`.
3. Ouvre **Configuration**. Conserve tout ce que Coolify a généré.
4. Ajoute, à la fin de la liste `command`, les deux lignes de [`examples/coolify-proxy-command.example.yml`](examples/coolify-proxy-command.example.yml) :

```yaml
- "--entrypoints.http.proxyProtocol.trustedIPs=10.10.0.1/32"
- "--entrypoints.https.proxyProtocol.trustedIPs=10.10.0.1/32"
```

5. Enregistre, puis **Restart Proxy**.
6. Rouvre la configuration et vérifie que les deux lignes sont présentes.

Ces lignes survivent à une régénération du proxy. Après une mise à jour de Coolify, revérifie-les quand même. Le bouton qui réinitialise le proxy les efface : ne l’utilise pas.

Le resolver généré s’appelle `letsencrypt` et utilise le défi HTTP sur l’entrypoint `http`. Il n’embarque pas l’adresse de contact écrite dans l’ancien `docker-compose.yml`. L’émission des certificats fonctionne sans cette ligne. Une ligne `--certificatesresolvers...email` ajoutée à la main est retirée si Coolify régénère le fichier, parce que Coolify réserve ce préfixe à sa propre configuration.

Si le proxy reste mauvais après le redémarrage :

```bash
sudo chmod 600 /data/coolify/proxy/acme.json
```

Les journaux sont dans **Servers → Proxy → Logs**.

## Préparer les domaines, avant la bascule

Les enregistrements DNS continuent de viser le VPS `137.74.116.56`. Ils ne doivent pas viser `192.168.1.18` ni l’adresse locale de la nouvelle machine.

Chez le registrar, `@` est seulement le domaine sans préfixe. Il ne crée pas `coolify.exemple.com`. Il faut une entrée A pour ce nom, ou une entrée `*` qui envoie tous les sous-domaines d’un seul niveau vers `137.74.116.56`. L’étoile ne remplace pas `@`, `www`, les MX ni les autres lignes déjà présentes. Traefik choisit le conteneur d’après le nom. Coolify refuse l’URL tant que le nom ne résout pas vers cette adresse : le message cite l’enregistrement A attendu. `Instance settings updated successfully` confirme ce DNS. Le site peut encore échouer tant que le proxy n’écoute pas sur 80 et 443.

Vérifie-le depuis la maison :

```bash
dig +short coolify.exemple.com
dig +short test.exemple.com
```

Les deux commandes doivent afficher `137.74.116.56`. S’il existe aussi un enregistrement `AAAA`, le défi HTTP peut échouer le jour où l’IPv6 du VPS ne transmet pas le port 80. Dans ce cas, retire l’`AAAA` ou fais suivre l’IPv6 par le même chemin que l’IPv4.

Dans **Settings → Configuration → General** :

- **URL** : `https://coolify.exemple.com`
- **Redirect HTTP to HTTPS** : activé
- **Instance public IPv4** : `137.74.116.56`

Cette IPv4 sert parce que la machine de la maison se présente avec l’adresse de la box, alors que les domaines pointent vers le VPS. Enregistre.

Le certificat public ne peut pas être émis tant que le tunnel arrive encore sur l’ancien serveur. Garde `http://192.168.1.x:8000` comme accès de secours. Ne redirige pas le port 8000 depuis la box.

Ce port 8000 sert le tableau de bord en HTTP, à côté de Traefik. Quand le domaine HTTPS répond, tu peux le limiter à la machine avec [`examples/coolify-localhost-port.example.yml`](examples/coolify-localhost-port.example.yml), comme sur le VPS. Les ports 80 et 443 restent nécessaires : une visite de l’adresse IP seule n’envoie pas le nom du domaine, donc Traefik ne sert pas le tableau de bord pour cette visite.

Si Coolify refuse encore le domaine à cause du contrôle DNS, ouvre **Settings → Configuration → Advanced** et désactive **DNS Validation**. Le contrôle compare le DNS à l’adresse qu’il croit être celle du serveur ; ici le chemin valide est le VPS.

## Basculer WireGuard

Fais cette étape seulement quand le compte admin existe, que les deux lignes PROXY sont enregistrées, et que le proxy est démarré. Une seule machine à la fois doit porter `10.10.0.2` et la clé du tunnel.

Sur l’ancien serveur, le fichier à reprendre est `/etc/wireguard/wg0.conf`. Il doit contenir l’adresse `10.10.0.2/24`, le port `44223`, le MTU `1420`, le peer `10.10.0.1/32`, l’endpoint `137.74.116.56:51820` et le keepalive `25`. Copie-le vers la nouvelle machine par le réseau local, par exemple avec `scp`. Ne le recrée pas à la main et ne le mets pas dans Git.

Sur la nouvelle machine :

```bash
sudo apt install -y wireguard
sudo install -m 600 /chemin/de/la/copie/wg0.conf /etc/wireguard/wg0.conf
```

Fenêtre de bascule, dans cet ordre :

```bash
# Ancien serveur : le tunnel s'arrête, Traefik ancien continue de tourner
sudo wg-quick down wg0

# Nouvelle machine
sudo wg-quick up wg0
sudo wg show
ip -4 addr show dev wg0
```

`wg show` doit afficher un handshake récent, le peer autorisé `10.10.0.1/32`, et l’endpoint `137.74.116.56:51820`. L’interface doit porter `10.10.0.2/24` avec un MTU de `1420`.

Pour que le tunnel revienne au démarrage de la nouvelle machine :

```bash
sudo systemctl enable wg-quick@wg0
```

Active ce service seulement après un `wg show` réussi. Sur l’ancien serveur, laisse le service WireGuard désactivé tant que la nouvelle machine garde l’adresse :

```bash
sudo systemctl disable wg-quick@wg0
```

## Vérifier le chemin complet

Depuis la nouvelle machine, le proxy doit écouter :

```bash
sudo ss -lntup | grep -E ':80|:443'
sudo ss -tnp 'sport = :80 or sport = :443'
```

Pendant une visite, l’adresse distante attendue est `10.10.0.1`.

Depuis un ordinateur extérieur au réseau de la maison :

```bash
curl -sI http://coolify.exemple.com
curl -sI https://coolify.exemple.com
```

La première commande doit renvoyer vers `https://coolify.exemple.com`. La seconde doit présenter un certificat Let’s Encrypt valide, après quelques instants. Le premier essai peut encore montrer un certificat provisoire le temps que le défi HTTP aboutisse. Si ça ne part pas, regarde **Proxy → Logs** et relance une fois le proxy. Évite les redémarrages en boucle : Let’s Encrypt limite les demandes répétées.

Dans ces journaux, l’adresse du visiteur doit être son IP publique. Si toutes les visites apparaissent comme `10.10.0.1`, les deux lignes PROXY manquent ou le proxy n’a pas été redémarré après leur ajout.

Contrôle aussi que le port 80 du défi arrive bien jusqu’à la nouvelle machine :

```bash
sudo timeout 20 tcpdump -ni wg0 -A -s 160 'tcp port 80 and src host 10.10.0.1'
```

Le début utile du paquet contient `QUIT` : c’est la signature PROXY version 2, la même que sur l’ancien serveur.

## Essayer une première application

Choisis un nom qui n’est utilisé par aucun service de l’ancien serveur, par exemple `test.exemple.com`. Son enregistrement `A` vaut `137.74.116.56`.

Dans Coolify, crée un projet et déploie une application dont le domaine est :

```text
https://test.exemple.com
```

Le port écrit dans ce domaine est le port interne du conteneur. Les visiteurs passent par le port 443 du proxy.

Attends le certificat, puis :

```bash
curl -sI https://test.exemple.com
```

Ouvre ensuite le site dans un navigateur. Quand cette application répond, les autres services peuvent être recréés un par un dans Coolify, chacun avec son domaine déjà pointé vers le VPS. Chaque ancien service reste sur l’ancien serveur tant que son nom n’a pas été ajouté ici et que son certificat a été émis.

## Revenir en arrière

Le VPS n’a pas changé. Le retour consiste à rendre `10.10.0.2` à l’ancien serveur. L’ancien Traefik et son dossier `letsencrypt` sont toujours en place.

```bash
# Nouvelle machine
sudo systemctl disable --now wg-quick@wg0
sudo wg-quick down wg0

# Ancien serveur
sudo systemctl enable --now wg-quick@wg0
sudo wg show
```

Le handshake doit revenir sur l’ancien serveur, avec l’adresse `10.10.0.2/24`. Les domaines repassent par l’ancien Traefik sans modification DNS.

Coolify peut rester installé sur la nouvelle machine. Sans le tunnel, il n’est plus sur le chemin public. L’accès de secours reste `http://192.168.1.x:8000` depuis la maison.
