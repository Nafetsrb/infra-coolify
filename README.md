# Coolify sur Requiem

Ce guide concerne Requiem, la machine de la maison. Coolify s’y installe à côté de l’ancien Traefik. Le VPS ne reçoit rien de nouveau : il reste la porte d’entrée, avec HAProxy et WireGuard déjà en place. Il n’y a pas de seconde machine et pas de bascule du tunnel.

Le fichier `docker-compose.yml` décrit le conteneur Traefik qui tourne déjà ici. Il sert de référence pour le proxy de Coolify. Une fois le nouveau proxy vérifié, ce conteneur s’arrête. Pour un Coolify installé directement sur un VPS public, suis [`README-vps.md`](README-vps.md).

Les domaines ci-dessous sont des exemples. Remplace-les par les tiens :

- tableau de bord Coolify : `coolify.exemple.com`
- première application : `test.exemple.com`

## Architecture mesurée

```text
Internet → VPS 137.74.116.56 → HAProxy → WireGuard → Requiem 10.10.0.2:80 et :443 → Traefik
```

Relevé sur Requiem, interface `wg0` :

| Élément | Valeur |
|---|---|
| Adresse WireGuard de cette machine | `10.10.0.2/24` |
| Adresse du VPS dans le tunnel | `10.10.0.1/32` |
| IP publique du VPS, endpoint du tunnel | `137.74.116.56:51820` |
| Port d’écoute WireGuard | `44223` |
| MTU | `1420` |
| Keepalive | `25` secondes |
| Réseau local | `192.168.1.0/24` |
| Cette machine sur le réseau local | `192.168.1.18` |

WireGuard ne se copie pas et ne se redémarre pas pour Coolify. `/etc/wireguard/wg0.conf` reste sur cette machine et ne va pas dans Git.

Un paquet arrivé sur le port 80 commence par la signature du protocole PROXY version 2. HAProxy l’envoie, Traefik ne fait confiance qu’à `10.10.0.1/32`, et Docker transmet cette adresse source jusqu’au conteneur. L’IP du visiteur vient de cet en-tête.

Le port 80 reste ouvert pour le défi HTTP-01 de Let’s Encrypt. Les sites eux-mêmes répondent en HTTPS. Le dépôt ne contient aucune route TCP, UDP, ni middleware supplémentaire. `acme.json` ne va pas dans Git : il contient les clés privées des certificats.

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

Les deux Traefik veulent les ports 80 et 443. Un seul peut les tenir. L’ancien conteneur reste démarré tant que le proxy Coolify n’est pas prêt à prendre sa place.

## Installer Coolify

Requiem a déjà Ubuntu, Docker, UFW et WireGuard. Le script officiel s’ajoute là-dessus. La page [Installation Coolify](https://coolify.io/docs/get-started/installation) indique les versions d’Ubuntu prises en charge. Le script actuel vise les LTS 20.04, 22.04 et 24.04.

```bash
curl -fsSL https://cdn.coollabs.io/coolify/install.sh | sudo bash
```

À la fin, ouvre le tableau de bord depuis la maison :

```text
http://192.168.1.18:8000
```

Crée le compte administrateur tout de suite. Tant que cette page est ouverte, la première personne qui l’atteint devient administrateur.

Copie le fichier qui contient les secrets de l’instance, dont `APP_KEY`, vers un endroit hors de ce dépôt Git :

```bash
sudo cp /data/coolify/source/.env /root/coolify-source.env
sudo chmod 600 /root/coolify-source.env
```

Garde une copie supplémentaire hors de la machine. Sans ce fichier, une restauration future ne peut pas relire la base Coolify.

Coolify tourne dans un conteneur. Pour agir sur Requiem, ce conteneur ouvre une session SSH vers la machine, sur `host.docker.internal`. Dans l’assistant, choisis **This machine**. Le serveur à garder est `localhost`. **New Server** sert à en ajouter un autre.

Sur Requiem, sshd écoute le port `22` sur toutes les interfaces, et `root` entre seulement avec une clé (`PermitRootLogin without-password`). Dans l’assistant, le port est donc `22` et l’utilisateur `root`. La clé affichée par Coolify, aussi présente dans `/data/coolify/ssh/keys/id.root@host.docker.internal.pub`, doit être une ligne de `/root/.ssh/authorized_keys`, sans effacer la clé de ta session.

UFW est actif. Il laisse passer toute la maison (`192.168.0.0/16`) et, depuis le VPS `10.10.0.1`, seulement les ports 80, 443 et 25565. Le conteneur Coolify n’est dans aucune de ces plages : son réseau Docker est `10.0.1.0/24`. Sans règle pour ce réseau, la connexion vers `host.docker.internal` port 22 expire.

La règle qui débloque Coolify :

```bash
sudo ufw allow from 10.0.1.0/24 to any port 22 proto tcp
```

Si le conteneur a changé d’adresse après une réinstallation, retrouve-la avant d’adapter la règle :

```bash
sudo docker inspect coolify -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}'
```

Les essais sur `172.16.0.0/12` n’ont servi à rien. Les retirer :

```bash
sudo ufw delete allow from 172.16.0.0/12 to any port 22 proto tcp
sudo ufw route delete allow from 172.16.0.0/12 to any port 22 proto tcp
```

`ufw status` doit garder `22/tcp ALLOW 10.0.1.0/24` et ne plus montrer `172.16.0.0/12`.

L’écran **Create "My First Project"** crée un projet vide. Rien n’est déployé. **Attention required** sur `localhost` veut dire que le proxy n’est pas encore démarré. **Sources**, **Destinations**, **S3 Storage** et **Shared Variables** restent vides.

## Régler le proxy, puis arrêter l’ancien Traefik

1. Dans Coolify, ouvre **Servers**, puis le serveur local, puis **Proxy**. Le type reste **Traefik**. **Switch Proxy** sert à le remplacer : Coolify répond `The running proxy must be stopped before switching` si tu cliques dessus pendant qu’il tourne. Reste sur Traefik.
2. Ouvre **Configuration**. Conserve tout ce que Coolify a généré.
3. Ajoute, à la fin de la liste `command`, les deux lignes de [`examples/coolify-proxy-command.example.yml`](examples/coolify-proxy-command.example.yml) :

```yaml
- "--entrypoints.http.proxyProtocol.trustedIPs=10.10.0.1/32"
- "--entrypoints.https.proxyProtocol.trustedIPs=10.10.0.1/32"
```

4. Enregistre. Ne démarre pas encore le proxy : l’ancien conteneur `traefik` occupe déjà les ports 80 et 443.
5. Arrête seulement ce conteneur. Le tunnel et les autres conteneurs restent démarrés.

```bash
sudo docker stop traefik
```

6. Dans Coolify, démarre le proxy. S’il reste sur **Starting** sans journal :

```bash
sudo docker compose -f /data/coolify/proxy/docker-compose.yml up -d
```

7. Rouvre la configuration et vérifie que les deux lignes PROXY sont présentes.

Ces lignes survivent à une régénération du proxy. Après une mise à jour de Coolify, revérifie-les quand même. Le bouton qui réinitialise le proxy les efface : ne l’utilise pas.

Le resolver généré s’appelle `letsencrypt` et utilise le défi HTTP sur l’entrypoint `http`. Il n’embarque pas l’adresse de contact écrite dans l’ancien `docker-compose.yml`. L’émission des certificats fonctionne sans cette ligne. Une ligne `--certificatesresolvers...email` ajoutée à la main est retirée si Coolify régénère le fichier, parce que Coolify réserve ce préfixe à sa propre configuration.

Si un certificat ne s’écrit pas :

```bash
sudo chmod 600 /data/coolify/proxy/acme.json
```

Les journaux sont dans **Servers → Proxy → Logs**.

Garde le conteneur `traefik` arrêté, sans le supprimer, tant que le nouveau chemin n’est pas vérifié. Son dossier `letsencrypt` reste la copie de secours des anciens certificats.

## Domaines

Les enregistrements DNS continuent de viser le VPS `137.74.116.56`. Ils ne changent pas, et ils ne visent pas `192.168.1.18`.

Chez le registrar, `@` est seulement le domaine sans préfixe. Il ne crée pas `coolify.exemple.com`. Il faut une entrée A pour ce nom, ou une entrée `*` qui envoie tous les sous-domaines d’un seul niveau vers `137.74.116.56`. L’étoile ne remplace pas `@`, `www`, les MX ni les autres lignes déjà présentes. Traefik choisit le conteneur d’après le nom. Coolify refuse l’URL tant que le nom ne résout pas vers cette adresse : le message cite l’enregistrement A attendu. `Instance settings updated successfully` confirme ce DNS. Le site peut encore échouer tant que `coolify-proxy` n’écoute pas sur 80 et 443.

Vérifie depuis la maison :

```bash
dig +short coolify.exemple.com
dig +short test.exemple.com
```

Les deux commandes doivent afficher `137.74.116.56`. S’il existe aussi un enregistrement `AAAA`, le défi HTTP peut échouer quand l’IPv6 du VPS ne transmet pas le port 80. Dans ce cas, retire l’`AAAA` ou fais suivre l’IPv6 par le même chemin que l’IPv4.

Dans **Settings → Configuration → General** :

- **URL** : `https://coolify.exemple.com`
- **Redirect HTTP to HTTPS** : activé
- **Instance public IPv4** : `137.74.116.56`

Cette IPv4 sert parce que Requiem se présente avec l’adresse de la box, alors que les domaines pointent vers le VPS. Enregistre.

Garde `http://192.168.1.18:8000` comme accès depuis la maison. Ne redirige pas le port 8000 depuis la box.

Ce port 8000 sert le tableau de bord en HTTP, à côté de Traefik. Quand le domaine HTTPS répond, tu peux le limiter à la machine avec [`examples/coolify-localhost-port.example.yml`](examples/coolify-localhost-port.example.yml). Les ports 80 et 443 restent nécessaires : une visite de l’adresse IP seule n’envoie pas le nom du domaine, donc Traefik ne sert pas le tableau de bord pour cette visite.

Si Coolify refuse encore le domaine à cause du contrôle DNS, ouvre **Settings → Configuration → Advanced** et désactive **DNS Validation**. Le contrôle compare le DNS à l’adresse qu’il croit être celle du serveur. Ici le chemin valide passe par le VPS.

## Vérifier le chemin

Le tunnel ne bouge pas. `wg show` doit continuer d’afficher un handshake récent avec `10.10.0.1/32`.

Le proxy qui écoute est `coolify-proxy`, plus `traefik` :

```bash
sudo docker ps --format '{{.Names}} {{.Ports}}' | grep -E 'traefik|coolify-proxy'
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

Contrôle aussi que le port 80 du défi arrive toujours par le tunnel :

```bash
sudo timeout 20 tcpdump -ni wg0 -A -s 160 'tcp port 80 and src host 10.10.0.1'
```

Le début utile du paquet contient `QUIT` : c’est la signature PROXY version 2.

## Essayer une première application

Choisis un nom qui n’est utilisé par aucun conteneur encore servi par l’ancien Traefik, par exemple `test.exemple.com`. Son enregistrement `A` vaut déjà `137.74.116.56` si l’étoile DNS est en place. Sinon, ajoute cette entrée A.

Dans Coolify, déploie une application dont le domaine est :

```text
https://test.exemple.com
```

Le port écrit dans ce domaine est le port interne du conteneur. Les visiteurs passent par le port 443 du proxy.

Attends le certificat, puis :

```bash
curl -sI https://test.exemple.com
```

Ouvre ensuite le site dans un navigateur. Quand cette application répond, les autres services peuvent être recréés un par un dans Coolify, chacun avec son domaine déjà pointé vers le VPS.

## Brancher Discord

Coolify envoie des messages dans un salon Discord par un webhook entrant. Le détail officiel est sur [Discord | Coolify Docs](https://coolify.io/docs/core/notifications/channels/discord). L’adresse du webhook ne va pas dans Git : quiconque l’a peut écrire dans le salon.

Dans Discord, ouvre le serveur qui doit recevoir les alertes. Il faut le droit de gérer les webhooks. Crée un salon texte, par exemple `coolify-notifications`, ou choisis-en un qui existe déjà.

1. Ouvre les paramètres du serveur Discord, puis **Intégrations**, puis **Webhooks**.
2. **Nouveau webhook**.
3. Donne-lui un nom reconnaissable, par exemple `Coolify`.
4. Choisis le salon qui doit recevoir les messages.
5. **Copier l’URL du webhook**.

Dans Coolify :

1. Ouvre **Notifications**, puis **Discord**.
2. Colle l’URL dans **Webhook** et enregistre.
3. Active **Enabled**.
4. **Ping Enabled** est facultatif. Il ajoute `@here` aux événements critiques. Discord ne prévient alors que les membres en ligne qui voient ce salon.
5. Dans **Notification Settings**, coche les événements à envoyer. L’écran propose notamment le succès et l’échec des déploiements, le changement d’état d’un conteneur (arrêt et redémarrage), le succès et l’échec des sauvegardes, les tâches planifiées, le nettoyage Docker, l’espace disque, le serveur joignable ou non, les mises à jour du serveur et un Traefik obsolète.
6. **Send Test Notification**.

Le message de test doit apparaître dans le salon avant d’élargir la liste des événements.

Si rien n’arrive : le webhook existe encore dans **Intégrations → Webhooks**, il vise bien ce salon, et l’URL enregistrée dans Coolify est celle qui vient d’être copiée. Un webhook recréé change de jeton : il faut recoller la nouvelle URL, enregistrer, puis renvoyer le test.

## Brancher Coolidash

[Coolidash](https://n4th4n.dev/coolidash) est l’application iPhone qui parle directement à l’API de cette instance. Elle n’utilise pas le mot de passe du compte. Le jeton ne va pas dans Git. Coolify ne l’affiche qu’une fois. La procédure de l’application est sur [Support Coolidash](https://n4th4n.dev/coolidash/support).

Hors de la maison, le téléphone passe par le VPS, en HTTPS. `http://192.168.1.18:8000` ne répond que sur le réseau local : Coolidash refuse d’envoyer le jeton en HTTP vers une adresse publique.

1. Dans Coolify, ouvre **Settings**, puis **Configuration**, puis **Advanced**.
2. Active **API Access**.
3. Laisse **Allowed IPs for API Access** vide. En 4G, l’adresse du téléphone change. Une liste limitée à la maison le bloque dès qu’il n’est plus sur ce réseau.
4. Ouvre **Keys & Tokens**, puis **API Tokens**. L’équipe active au moment de la création est celle du jeton.
5. Description, par exemple `coolidash`.
6. Coche `read` et `deploy`. Ajoute `read:sensitive` pour les logs de build. Ne coche pas `root`.
7. **Create**, puis copie le jeton tout de suite, numéro et barre verticale compris.

Dans Coolidash, touche **Connecter un serveur** :

1. Protocole **HTTPS**.
2. Domaine `coolify.exemple.com`, sans `https://` et sans port.
3. Colle le jeton en entier.
4. Valide. L’application teste la connexion avant d’enregistrer.

Si Coolidash dit que le jeton n’a pas les permissions nécessaires, recrée-le avec `read` et `deploy`. Un certificat refusé veut dire que Let’s Encrypt n’est pas encore en place sur ce nom : le certificat provisoire ou expiré est rejeté par iOS. Un jeton perdu se révoque dans **API Tokens**, puis on en crée un autre. Changer le mot de passe du compte révoque aussi ses jetons.

## Retirer l’ancien conteneur

Quand `coolify-proxy` répond et que l’application d’essai a son certificat, l’ancien conteneur n’a plus à redémarrer tout seul. Le projet Compose qui le lançait est `~/Traefik`.

```bash
cd ~/Traefik
sudo docker compose stop traefik
```

`docker stop traefik` suffit si ce Compose n’est plus le moyen de le démarrer. Ne supprime pas tout de suite le dossier `letsencrypt` : il sert au retour arrière.

## Revenir en arrière

Le VPS, HAProxy et WireGuard n’ont pas changé. Le retour consiste à rendre les ports 80 et 443 à l’ancien conteneur.

```bash
sudo docker stop coolify-proxy
sudo docker start traefik
```

Les domaines repassent par l’ancien Traefik, avec les certificats de son dossier `letsencrypt`. Coolify peut rester installé. Sans le port 80 et le port 443, il n’est plus sur le chemin public. L’accès depuis la maison reste `http://192.168.1.18:8000`.
