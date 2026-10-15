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

---

### 3.12 Liste des images avec `docker images`

#### Commande(s) :
- `docker images`
- `docker image ls`

#### Résultat(s) :
```text
IMAGE                ID             DISK USAGE   CONTENT SIZE   EXTRA
hello-world:latest   11a3eb1597e8       25.9kB         9.49kB    U
```
Les deux commandes affichent la même image/table.  
Image utilisée : `hello-world:latest`, ID `11a3eb1597e8`.

#### Commentaire :
`docker images` et `docker image ls` sont équivalentes. L’identifiant court affiché est `11a3eb1597e8` ; l’ID complet avait été vu à la Q3.8 (`sha256:11a3eb1597e864c51b3f56733cb2e8a90bf0c284bfc05f9444d81c558bc848aa`).

---

### 3.13 Réexécution de `hello-world` et comparaison

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
```

#### Commentaire :
La sortie reste la même, car ce texte fait partie du programme `hello-world`. Comparé à la première exécution, cette seconde tentative est en général beaucoup plus rapide, puisque l’image est déjà présente localement; Docker ne recrée pas de téléchargement. Chaque `docker run` crée toutefois un **nouveau conteneur**, et l’ancien reste conservé tant qu’on ne le supprime pas.

---

### 3.14 Nombre de conteneurs en cours / stockés

#### Commande(s) :
- `docker ps`
- `docker ps -a`

#### Résultat(s) :
- `docker ps` : **0 conteneur en exécution** (tableau sans ligne active).
- `docker ps -a` : **3 conteneurs stockés localement**, tous au statut `Exited (0)` :
  - `0a5d6006e4c4` `hello-world` `/hello` → created 32s ago
  - `9e4e86e6f3e8` `hello-world` `/hello` → created 19 min ago
  - `657d8a16a2dc` `hello-world` `/hello` → created 20 min ago

#### Commentaire :
Les trois instances précédemment créées sont arrêtées mais restent stockées localement. Cela explique aussi pourquoi l’image `hello-world` ne peut pas être supprimée sans forcer : des conteneurs, même arrêtés, y sont encore associés.

---

### 3.15 Tentative de suppression de `hello-world`

#### Commande(s) :
- `docker rmi hello-world`

#### Résultat(s) :
`Error response from daemon: conflict: unable to delete hello-world:latest (must be forced) - container 0a5d6006e4c4 is using its referenced image 11a3eb1597e8`

#### Commentaire :
Docker **refuse** de supprimer l’image car au moins un conteneur (`0a5d6006e4c4`), même arrêté, y est encore associé. Il faut d’abord supprimer les conteneurs concernés avant de réessayer `docker rmi hello-world`. L’option `-f` n’est pas utilisée ici, conformément au sujet.

---

### 3.16 Noms des conteneurs

#### Commande(s) :
- `docker ps -a`

#### Résultat(s) :
Les trois conteneurs visibles dans la table ont les noms suivants :
- `sweet_greider` (`0a5d6006e4c4`)
- `eager_shannon` (`9e4e86e6f3e8`)
- `magical_buck` (`657d8a16a2dc`)

Tous sont de l’image `hello-world` avec le statut `Exited (0)`.

#### Commentaire :
Docker assigne automatiquement un nom aléatoire si on ne précise pas `--name`. Ces noms sont visibles dans la colonne `NAMES` et peuvent être utilisés directement dans des commandes comme `docker rm <nom>`.

---

### 3.17 Affichage non tronqué avec `--no-trunc`

#### Commande(s) :
- `docker ps -a --no-trunc`

#### Résultat(s) :
Les IDs complets deviennent visibles :
- `0a5d6006e4c4143357e98dd1776bac8e54e30767ec8db38e6935cd824bb91105` → `sweet_greider`
- `9e4e86e6f3e8f1cd444736a14ac6cda4cc26d4f72fee9403fc5b8eb117b4ca27` → `eager_shannon`
- `657d8a16a2dc77937bf2ffd99d8b20b3138f002b1b8ad111aac6885c8f92368a` → `magical_buck`

Tous concernent `hello-world`, au statut `Exited (0) About ... ago`.

#### Commentaire :
Par défaut, `docker ps` tronque les IDs et certaines commandes. L’option `--no-trunc` permet de retrouver l’ID complet, utile pour des scripts ou des suppressions précises.

---

### 3.18 Exécution puis suppression par nom

#### Commande(s) :
- `docker run hello-world`
- `docker ps -l --format '{{.Names}}'`
- `docker rm <nom_du_nouveau_conteneur>`

#### Résultat(s) :
- `docker run hello-world` crée et exécute un nouveau conteneur (avec la sortie habituelle `Hello from Docker!`).
- Nom du dernier conteneur créé : `dazzling_shamir`.
- Suppression réussie : `docker rm dazzling_shamir` affiche `dazzling_shamir`.

#### Commentaire :
Le conteneur vient d’être créé puis supprimé par son nom aléatoire. `docker ps -l` permet de récupérer facilement le dernier conteneur, et `docker rm` supprime uniquement le conteneur concerné (pas l’image).

---

### 3.19 Exécution par SHA256 avec nom de conteneur

#### Commande(s) :
- `docker run --name hello_sha sha256:11a3eb1597e864c51b3f56733cb2e8a90bf0c284bfc05f9444d81c558bc848aa`

#### Résultat(s) :
- Conteneur nommé `hello_sha`
- Sortie habituelle `Hello from Docker!`
- Étapes standard indiquées par l’image, comme pour les exécutions précédentes.

#### Commentaire :
L’image peut être référencée par son SHA256 complet et un nom de conteneur peut être attribué simultanément avec `--name`. Ce conteneur est également associé à l’image, donc il devra être supprimé avant de pouvoir supprimer l’image sans `--force`.

---

### 3.20 Suppression de l’image sans forcer

#### Commande(s) :
- `docker rm hello_sha pensive_leakey funny_dubinsky && docker rmi hello-world && docker images`
- `docker ps -a`

#### Résultat(s) :
```text
hello_sha
pensive_leakey
funny_dubinsky
Untagged: hello-world:latest
Deleted: sha256:11a3eb1597e864c51b3f56733cb2e8a90bf0c284bfc05f9444d81c558bc848aa
```
`docker images` ne liste plus aucune image :
```text
IMAGE   ID             DISK USAGE   CONTENT SIZE   EXTRA
```
`docker ps -a` ne liste plus aucun conteneur :
```text
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
```

#### Commentaire :
Docker refuse de supprimer une image tant qu’un conteneur, même arrêté, la référence encore. Il faut donc d’abord supprimer tous les conteneurs associés (`hello_sha`, `pensive_leakey`, `funny_dubinsky`), puis exécuter `docker rmi hello-world` **sans** l’option `--force`. La sortie `Untagged` retire le tag `hello-world:latest`, puis `Deleted` supprime effectivement l’ID `sha256:11a3eb…848aa`. À la fin, les listes d’images et de conteneurs sont vides.

---

### 3.21 Informations globales sur le système Docker

#### Commande(s) :
- `docker info`
- `docker info | grep -E 'Containers|Images'`

#### Résultat(s) :
Extrait de `docker info` :
```text
Server:
 Containers: 0
  Running: 0
  Paused: 0
  Stopped: 0
 Images: 0
 Server Version: 29.9.0
 Storage Driver: overlayfs
 Operating System: Ubuntu 22.04.4 LTS
 Kernel Version: 5.15.0-118-generic
 CPUs: 2
 Total Memory: 1.918GiB
 Name: vmdocker
 Docker Root Dir: /var/lib/docker
```
Résultat filtré :
```text
 Containers: 0
 Images: 0
```

#### Commentaire :
`docker info` affiche un état global du moteur Docker : versions client/serveur, pilote de stockage, runtime, noyau, ressources machines, plugins, etc. Les lignes `Containers` et `Images` donnent ici respectivement le nombre de conteneurs et d’images connus localement. Après la Q3.20, plus aucun conteneur ni image n’est stocké, d’où les compteurs à `0`. La variante avec `grep` permet de ne conserver que les deux informations demandées.

---

### 3.22 Affichage des IDs de conteneurs uniquement

#### Commande(s) :
- `docker ps -aq`
- Équivalents possibles : `docker ps -q` (conteneurs en cours d’exécution uniquement), `docker container ls -aq`

#### Résultat(s) :
- `-q` / `--quiet` n’affiche que les identifiants, un par ligne.
- `-a` / `--all` inclut les conteneurs arrêtés, pas seulement ceux en cours d’exécution.
- Dans l’état actuel (aucun conteneur stocké), la commande ne retourne donc aucune ligne.

#### Commentaire :
`docker ps -aq` combine deux options : `-a` liste tous les conteneurs et `-q` ne conserve que leurs IDs. C’est la forme la plus utile pour la suite, car elle permet de récupérer en une seule sortie les identifiants de tous les conteneurs, y compris les conteneurs arrêtés, afin de les transmettre ensuite à `docker rm` (Q3.23).

---

### 3.23 Suppression de tous les conteneurs en une seule commande

#### Commande(s) :
- `docker run hello-world` (×2, pour recréer des conteneurs de test)
- `docker ps -a`
- `docker rm $(docker ps -aq)`
- `docker ps -a`

#### Résultat(s) :
Après les deux exécutions, deux conteneurs arrêtés sont présents :
```text
CONTAINER ID   IMAGE         COMMAND    CREATED                  STATUS                              PORTS     NAMES
caaabea6788a   hello-world   "/hello"   Less than a second ago   Exited (0) Less than a second ago             epic_galois
ed367948e1fc   hello-world   "/hello"   Less than a second ago   Exited (0) Less than a second ago             blissful_ramanujan
```
`docker rm $(docker ps -aq)` supprime les deux conteneurs et affiche leurs IDs :
```text
caaabea6788a
ed367948e1fc
```
Le `docker ps -a` final ne liste plus aucun conteneur :
```text
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
```

#### Commentaire :
`$(docker ps -aq)` est une substitution de commande bash : Docker fournit d’abord la liste des identifiants de tous les conteneurs (`-a` pour inclure les conteneurs arrêtés, `-q` pour n’afficher que les IDs), puis `docker rm` récupère cette liste comme arguments et les supprime en une seule commande. Sans conteneur existant, la substitution est vide et `docker rm` renvoie l’erreur *« requires at least 1 argument »* ; des conteneurs de test ont donc été recréés ici pour démontrer la commande.  

---

### 3.24 Suppression de l’image après recréation des conteneurs

#### Commande(s) :
- `docker rmi hello-world`
- `docker images`

#### Résultat(s) :
```text
Untagged: hello-world:latest
Deleted: sha256:11a3eb1597e864c51b3f56733cb2e8a90bf0c284bfc05f9444d81c558bc848aa
```
`docker images` ne liste plus aucune image :
```text
IMAGE   ID             DISK USAGE   CONTENT SIZE   EXTRA
```

#### Commentaire :
L’image avait été retéléchargée automatiquement lors des `docker run` de la Q3.23. Une fois les conteneurs supprimés, `docker rmi hello-world` peut de nouveau retirer le tag (`Untagged`) puis supprimer l’image (`Deleted`) **sans** l’option `--force`. Le second `docker images` confirme qu’il ne reste plus aucune image locale.

---

### 3.25 Nouvelle exécution après suppression de l’image

#### Commande(s) :
- `docker run --name hello1 hello-world`

#### Résultat(s) :
```text
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
4f55086f7dd0: Pull complete
d5e71e642bf5: Download complete
Digest: sha256:11a3eb1597e864c51b3f56733cb2e8a90bf0c284bfc05f9444d81c558bc848aa
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.
...
```

#### Commentaire :
L’image `hello-world` avait été supprimée à la Q3.24. Docker ne la trouve donc pas localement (*Unable to find image*) et la **retélécharge automatiquement** depuis Docker Hub avant de créer puis d’exécuter le conteneur nommé `hello1`. Le digest récupéré est identique à l’ancien ID de l’image : `sha256:11a3eb1597e864c51b3f56733cb2e8a90bf0c284bfc05f9444d81c558bc848aa`. C’est le comportement attendu : un `docker run` référence toujours l’image par son nom, et Docker la rapatrie si nécessaire.

