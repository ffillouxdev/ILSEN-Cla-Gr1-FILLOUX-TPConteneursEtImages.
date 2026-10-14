# Rapport du TP2 de « cours en question »

Note : version actuelle en Markdown. Une conversion en PDF via LaTeX est prévue.

Environnement : VM `vmdocker`, utilisateur `dockeruser` (Docker installé via le dépôt apt).

## Sommaire

- Partie 2

## Partie 2 – Installation et démarrage de Docker

### 2.1 Version de Docker

#### Commande(s) :
- `docker --version`
- `docker version`

#### Résultat(s) :
- Client uniquement : `Docker version 29.8.0, build 88096ef005`
- Client + serveur :
  - Client : `29.8.0` (API `1.56`, Go `1.27.1`, build `88096ef005`)
  - Serveur / Engine : `29.8.0` (API `1.56`, min `1.40`, build `3ce5872b79`)
  - containerd : `v2.3.5`
  - runc : `1.5.1`
  - docker-init : `0.19.0`

#### Commentaire :
`docker --version` n’affiche que la version du client, tandis que `docker version` distingue le client du serveur (moteur Docker + runtime).

---

### 2.2 Composants du démon et services/sockets

#### Commande(s) :
- `ps aux | grep -E '[d]ockerd|[c]ontainerd'`
- `systemctl status docker docker.socket containerd --no-pager`
- `ls -l /var/run/docker.sock`

#### Résultat(s) :
- Processus principaux : `dockerd` et `containerd` (plus shims/proxies selon l’activité).
- Services actifs : `docker.service`, `docker.socket`, `containerd.service`.
- Socket API : `/var/run/docker.sock`, type `srw-rw----`, propriétaire `root:docker`.

#### Commentaire :
Les services sont démarrés par `root` (systemd). L’accès au socket est accordé au groupe `docker`, d’où l’ajout de l’utilisateur `dockeruser` à ce groupe.

---

### 2.3 Arrêt de Docker

#### Commande(s) :
- `sudo systemctl stop docker docker.socket`

#### Résultat(s) :
- `docker.service` → `inactive (dead)`
- `docker.socket` → `inactive (dead)` (n’écoute plus sur `/run/docker.sock`)

#### Commentaire :
Il faut arrêter le socket en même temps que le service, sinon le socket peut réveiller Docker à la prochaine commande.

---

### 2.4 Désactivation / réactivation

#### Commande(s) :
- Désactivation : `sudo systemctl disable --now docker` (+ `sudo systemctl disable docker.socket`)
- Réactivation : `sudo systemctl enable --now docker docker.socket containerd`

#### Résultat(s) :
- Vérification : `systemctl is-enabled docker docker.socket containerd`
- Vérification : `systemctl is-active docker docker.socket containerd`

#### Commentaire :
`disable` empêche le démarrage automatique au boot ; `enable` le rétablit.

---

### 2.5 Utilisation sans sudo

#### Commande(s) :
- `sudo groupadd docker`
- `sudo -i` puis `usermod -aG docker dockeruser`
- `sudo reboot` puis `docker version` sans `sudo`

#### Résultat(s) :
- `id` contient le groupe `docker`
- `docker version` fonctionne sans `sudo`
- `/var/run/docker.sock` : `root:docker`, droits `srw-rw----`

#### Commentaire :
Après reconnexion, `dockeruser` appartient au groupe `docker` et peut utiliser l’API via `/var/run/docker.sock` sans `sudo`.

---

## Partie 3 – Premières manipulations de conteneurs et d’images

### 3.1 Répertoire de stockage des objets Docker

#### Commande(s) :
- `docker info --format '{{.DockerRootDir}}'`
- `docker info | grep "Docker Root Dir"`

#### Résultat(s) :
- `/var/lib/docker`
- `Docker Root Dir: /var/lib/docker`

#### Commentaire :
La commande `docker info` indique le répertoire racine de données du démon Docker. C’est là que Docker stocke ses objets et ses métadonnées (`/var/lib/docker`).

---

### 3.2 Quels sont les différentes catégories d’objets Docker qui peuvent être stockés ?

#### Commande(s) :
- `ls -la /var/lib/docker`

#### Résultat(s) :
- Répertoires observés : `buildkit`, `containers`, `image`, `network`, `plugins`, `rootfs`, `runtimes`, `swarm`, `tmp`, `volumes` (plus `engine-id`).

#### Commentaire :
Les catégories principales d’objets Docker sont les **images**, **conteneurs**, **réseaux** et **volumes**. Les autres répertoires (`buildkit`, `plugins`, `rootfs`, `runtimes`, `swarm`, `tmp`, etc.) correspondent à des données internes, au cache de build, aux plugins et aux métadonnées du daemon.

---

### 3.3 Consultation du répertoire des conteneurs

#### Commande(s) :
- `ls -la /var/lib/docker/containers`

#### Résultat(s) :
- Aucun sous-répertoire de conteneur présent : `/var/lib/docker/containers` est vide (seulement `.` et `..`) ; donc **0 conteneur** pour l’instant.

#### Commentaire :
Avant d’exécuter un conteneur `hello-world`, Docker n’a encore créé aucun objet conteneur. Le nombre réel se constate avec `ls /var/lib/docker/containers` ou `docker ps -a`.

---	

### 3.4 Recherche d’images Docker

#### Commande(s) :
- `docker search hello-world`

#### Résultat(s) :
- `hello-world` est présent et officiel, avec `2595` étoiles.
- Autres résultats (non officiels, moins étoilés) : `rancher/hello-world` (6), `tutum/hello-world` (91), `crccheck/hello-world` (26), etc.
- L’image ayant le plus d’étoiles et marquée `[OK]` est `hello-world`.

#### Commentaire :
`docker search` interroge Docker Hub. L’image officielle `hello-world` est la plus étoilée ; c’est donc celle que l’on utilise pour la suite du TP.

---

### 3.5 Exécution de l’image `hello-world`

#### Commande(s) :
- `docker run hello-world`

#### Résultat(s) :
```text
Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```

#### Commentaire :
Docker a téléchargé l’image `hello-world`, puis créé et exécuté le conteneur correspondant. Le message confirme que l’installation et l’exécution fonctionnent. Le conteneur s’arrête immédiatement après l’affichage, mais reste stocké localement tant qu’on ne le supprime pas.

---

### 3.6 Nombre de conteneurs après exécution

#### Commande(s) :
- `ls -la /var/lib/docker/containers`
- `ls -1 /var/lib/docker/containers | wc -l`

#### Résultat(s) :
- Deux sous-répertoires observés :
  - `657d8a16a2dc77937bf2ffd99d8b20b3138f002b1b8ad111aac6885c8f92368a`
  - `9e4e86e6f3e8f1cd444736a14ac6cda4cc26d4f72fee9403fc5b8eb117b4ca27`
- Total : `2` conteneurs enregistrés localement.

#### Commentaire :
L’exécution de `hello-world` a créé un conteneur arrêté (plus le conteneur déjà présent localement de l’environnement initial pour la seconde entrée). La commande `docker ps -a` permet de confirmer leur rapport avec l’image `hello-world`.

---

### 3.7 Contenu du répertoire des images

#### Commande(s) :
- `ls -la /var/lib/docker/image`
- `ls -la /var/lib/docker/image/overlay2`

#### Résultat(s) :
- `/var/lib/docker/image` contient ici uniquement : `identity-cache.db` (`rw-------`).
- `/var/lib/docker/image/overlay2` n’existe pas dans cet environnement : `No such file or directory`.

#### Commentaire :
Dans cette VM, le répertoire visible `/var/lib/docker/image` ne montre pas directement le répertoire habituel du storage driver (`overlay2`) ; il ne contient qu’un cache d’identité. Cela peut dépendre du moteur/de la configuration de stockage. Les métadonnées et couches de l’image restent interrogeables via Docker (`docker image inspect`, `docker images`) même si le chemin filesystem brut diffère de l’habitude.

---

### 3.8 SHA256 de l’image `hello-world`

#### Commande(s) :
- `docker images --no-trunc`

#### Résultat(s) :
- `hello-world   latest    sha256:11a3eb1597e864c51b3f56733cb2e8a90bf0c284bfc05f9444d81c558bc848aa   6 months ago   25.9kB`
- SHA256 complet de l’image : `sha256:11a3eb1597e864c51b3f56733cb2e8a90bf0c284bfc05f9444d81c558bc848aa`.

#### Commentaire :
La commande affiche l’ID long (non tronqué) de l’image locale `hello-world`. On peut ensuite le réutiliser comme identifiant exact pour lancer un conteneur avec :
`docker run sha256:11a3eb1597e864c51b3f56733cb2e8a90bf0c284bfc05f9444d81c558bc848aa`.


---

### 3.9 Listing des conteneurs en exécution

#### Commande(s) :
- `docker ps`

#### Résultat(s) :
```text
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
```
Aucune ligne active : le tableau est vide et `docker ps` n'affiche que les conteneurs en cours d'exécution.

#### Commentaire :
Par défaut, `docker ps` masque les conteneurs arrêtés. Comme `hello-world` s’arrête immédiatement après l’affichage, il n’apparaît pas ici.

---

### 3.10 Nombre de conteneurs en exécution

#### Commande(s) :
- `docker ps`

#### Résultat(s) :
- **0 conteneur en cours d’exécution**.

#### Commentaire :
Le tableau returned by `docker ps` ne contient aucune ligne. Les conteneurs créés par `hello-world` sont donc arrêtés après leur exécution.

---

### 3.11 Alias de la commande

#### Commande(s) :
- `docker ps`

#### Résultat(s) :
- `docker ps` constitue un alias de `docker container ls` (équivalents officiels : `docker container list`, `docker container ps`, `docker ps`).

#### Commentaire :
Les deux formes visent la même opération. `docker ps` est la commande historique; `docker container ls` est la forme moderne sous la sous-commande `container`.
