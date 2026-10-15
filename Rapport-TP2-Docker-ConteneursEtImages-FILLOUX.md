# Rapport du TP2 de « cours en question »

Note : version actuelle en Markdown. Une conversion en PDF via LaTeX est prévue.

Environnement : VM `vmdocker`, utilisateur `dockeruser` (Docker installé via le dépôt apt).

## Sommaire

- Partie 2
- Partie 3
- Partie 4
- Partie 5

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

---

### 3.26 Identifiant de l’image

#### Commande(s) :
- `docker images --no-trunc`

#### Résultat(s) :
```text
REPOSITORY    TAG       IMAGE ID                                                                  CREATED        SIZE
hello-world   latest    sha256:11a3eb1597e864c51b3f56733cb2e8a90bf0c284bfc05f9444d81c558bc848aa   6 months ago   25.9kB
```

#### Commentaire :
L’identifiant complet de l’image `hello-world:latest` est `sha256:11a3eb1597e864c51b3f56733cb2e8a90bf0c284bfc05f9444d81c558bc848aa`. On le retrouve identique à celui observé aux Q3.8, Q3.19 et Q3.25, ce qui confirme que l’ID d’une image est stable tant que son contenu ne change pas. L’option `--no-trunc` évite l’affichage tronqué (par défaut `11a3eb1597e8`).

---

### 3.27 Création d’un conteneur sans le démarrer

#### Commande(s) :
- `docker create --name hello2 hello-world`
- `docker ps -a`

#### Résultat(s) :
```text
4df9754d5ba0bd21c1dbfdf690800d049759ee2708ed81105da20c7aeefc5f2c
CONTAINER ID   IMAGE         COMMAND    CREATED                  STATUS                     PORTS     NAMES
4df9754d5ba0   hello-world   "/hello"   Less than a second ago   Created                              hello2
08a04dd7c691   hello-world   "/hello"   6 minutes ago            Exited (0) 6 minutes ago             hello1
```

#### Commentaire :
`docker create` crée le conteneur et prépare son système de fichiers, mais ne lance pas son processus principal : le conteneur `hello2` apparaît donc avec le statut `Created`, contrairement à `hello1` qui est déjà passé en `Exited (0)`. On peut ainsi préparer un conteneur à l’avance, puis le démarrer plus tard avec `docker start`.

---

### 3.28 Démarrage de `hello2` par son nom

#### Commande(s) :
- `docker start hello2`
- `docker ps -a`

#### Résultat(s) :
```text
hello2
CONTAINER ID   IMAGE         COMMAND    CREATED          STATUS                      PORTS     NAMES
4df9754d5ba0   hello-world   "/hello"   3 minutes ago    Up Less than a second                 hello2
08a04dd7c691   hello-world   "/hello"   10 minutes ago   Exited (0) 10 minutes ago             hello1
```

#### Commentaire :
`docker start hello2` démarre le conteneur sans attacher son terminal : la commande n’affiche donc que le nom `hello2`, et non la sortie du programme `/hello`. Juste après, le conteneur apparaît en cours d’exécution (`Up Less than a second`). Comme le programme `hello-world` se termine immédiatement, le statut repasse ensuite en `Exited (0)`. Pour voir directement le message, il faudra utiliser l’option d’attachement (Q3.29).

---

### 3.29 Démarrage de `hello2` avec attachement

#### Commande(s) :
- `docker start -a hello2`

#### Résultat(s) :
```text
Hello from Docker!
This message shows that your installation appears to be working correctly.
...
```

#### Commentaire :
L’option `-a` (`--attach`) attache la sortie standard (et l’entrée standard) du conteneur au terminal courant. Contrairement à `docker start hello2` qui n’affichait que le nom, on voit donc ici la sortie du programme `/hello` défiler directement.

---

### 3.30 Démarrage de `hello1` avec attachement

#### Commande(s) :
- `docker start -a hello1`

#### Résultat(s) :
```text
Hello from Docker!
This message shows that your installation appears to be working correctly.
...
```

#### Commentaire :
La même option `-a` fonctionne sur n’importe quel conteneur arrêté, référencé par son nom (`hello1` ici). Le message `Hello from Docker!` est de nouveau affiché en direct dans le terminal, ce qui confirme l’attachement de la sortie standard.

---

### 3.31 Exécution en arrière-plan avec `-d`

#### Commande(s) :
- `docker run -d --name hello3 hello-world`
- `docker ps -a`

#### Résultat(s) :
```text
f435cc31db3392e1cfe3994fe0b2ead09e7b3817fcfd69be2b148e8f9560079d
CONTAINER ID   IMAGE         COMMAND    CREATED                  STATUS                     PORTS     NAMES
f435cc31db33   hello-world   "/hello"   Less than a second ago   Up Less than a second                hello3
4df9754d5ba0   hello-world   "/hello"   7 minutes ago            Exited (0) 2 minutes ago             hello2
08a04dd7c691   hello-world   "/hello"   14 minutes ago           Exited (0) 2 minutes ago             hello1
```

#### Commentaire :
L’option `-d` (`--detach`) exécute le conteneur en arrière-plan : la sortie standard du programme n’est pas attachée au terminal, et `docker run` n’affiche donc que l’ID du conteneur créé. `docker ps -a` montre ensuite `hello3`, d’abord en cours d’exécution (`Up Less than a second`) puis, très rapidement, en `Exited (0)` puisque le programme `/hello` se termine immédiatement.

---

### 3.32 Exécution avec suppression automatique (`--rm`)

#### Commande(s) :
- `docker run --rm --name hello4 hello-world`
- `docker ps -a`

#### Résultat(s) :
```text
Hello from Docker!
This message shows that your installation appears to be working correctly.
...
```
`docker ps -a` ne montre plus que les conteneurs précédents (`hello3`, `hello2`, `hello1`) et **aucune trace de `hello4`** :
```text
CONTAINER ID   IMAGE         COMMAND    CREATED          STATUS                      PORTS     NAMES
f435cc31db33   hello-world   "/hello"   36 seconds ago   Exited (0) 35 seconds ago             hello3
4df9754d5ba0   hello-world   "/hello"   8 minutes ago    Exited (0) 3 minutes ago              hello2
08a04dd7c691   hello-world   "/hello"   14 minutes ago   Exited (0) 2 minutes ago              hello1
```

#### Commentaire :
L’option `--rm` demande à Docker de supprimer automatiquement le conteneur dès qu’il s’arrête. Le conteneur `hello4` exécute donc bien le programme `hello-world`, mais il disparaît aussitôt après : il n’apparaît pas dans `docker ps -a`, contrairement aux conteneurs `hello1`, `hello2` et `hello3` conservés. C’est la façon de faire pour un conteneur « jetable », dont on ne veut garder aucune trace locale.

---

## Partie 4 – DockerHub

### 4.1 Types d’images sur Docker Hub

#### Commande(s) :
- Consultation du registre public : `https://hub.docker.com/`

#### Résultat(s) :
Docker Hub distingue principalement cinq catégories d’images, matérialisées par des badges de confiance :

1. **Docker Official Images** — images officielles, maintenues par Docker et les éditeurs concernés, publiées dans l’espace `library/` (ex. `ubuntu`, `nginx`).
2. **Docker Verified Publisher (DVP)** — images d’éditeurs commerciaux vérifiés par Docker.
3. **Docker Sponsored OSS (DSOS)** — images de projets open source sponsorisés par Docker.
4. **Docker Hardened Images (DHI)** — images durcies, minimales, orientées sécurité et entreprise.
5. **Images communautaires** — images publiées par n’importe quel utilisateur ou organisation, sans badge de confiance Docker.

À cela s’ajoutent des classifications secondaires pour naviguer : type de contenu (image, modèle IA, Helm chart, sandbox), catégorie fonctionnelle (Operating Systems, Languages & Frameworks, Databases, etc.) et architecture (amd64, arm64, etc.).

#### Commentaire :
La classification principale repose donc sur le **niveau de confiance et le mode de publication** de l’image (officielle, éditeur vérifié, open source sponsorisé, durcie, ou communautaire).

---

### 4.2 Image officielle de l’OS Ubuntu

#### Commande(s) :
- Consultation de la page officielle : `https://hub.docker.com/_/ubuntu`
- Récupération locale possible : `docker pull ubuntu`

#### Résultat(s) :
- Dépôt officiel : **https://hub.docker.com/_/ubuntu**
- Nom de l’image : `ubuntu` (namespace officiel `library/ubuntu`)
- Badge : **Docker Official Image**
- Mainteneur : **Canonical**
- Récupération : `docker pull ubuntu`
- Tags principaux visibles : `24.04` (`noble`), `26.04` (`resolute`, associé à `latest` et `rolling`), `26.10` (`stonking`), etc.

#### Commentaire :
L’image `ubuntu` est une image officielle maintenue par Canonical ; elle sert de base minimale pour de nombreux conteneurs et est disponible pour plusieurs architectures (`amd64`, `arm64`, `ppc64le`, `s390x`, etc.).

---

### 4.3 Versions proposées et distinction par tags

#### Commande(s) :
- Consultation de la liste des tags : `https://hub.docker.com/_/ubuntu/tags`

#### Résultat(s) :
Les versions sont distinguées par leur **tag**, qui sert d’étiquette/alias vers une image donnée :

- **Versions par numéro de release** : `22.04`, `24.04`, `26.04`, `26.10`, etc.
- **Versions par nom de code** : `jammy` (22.04), `noble` (24.04), `resolute` (26.04), `stonking` (26.10), etc.
- **Tags datés / figés** : `jammy-20260924.1`, `noble-20260917`, `resolute-20260927`, utiles pour la reproductibilité.
- **Alias mobiles** :
  - `latest` → pointe vers la dernière **LTS** (ici Ubuntu 26.04 « resolute »).
  - `rolling` → pointe vers la dernière release publiée, LTS ou non.
  - `devel` → pointe vers la version en développement (ici 26.10 « stonking »).

#### Commentaire :
Le nom du tag permet donc de choisir une version précise ou de suivre une version mobile. Plusieurs tags peuvent néanmoins pointer vers le **même digest** d’image (par exemple `26.04`, `resolute`, `latest` et `rolling`).

---

### 4.4 Version la plus récente et version `latest`

#### Commande(s) :
- Consultation des tags : `https://hub.docker.com/_/ubuntu/tags`

#### Résultat(s) :
- **Version la plus récente** : **Ubuntu 26.10 « stonking »**
  - Tag : `26.10` (également `stonking`, `devel` actuellement)
  - Digest/ID : `sha256:ee126c2fa0249079a7e24ae3a3d29b04783ef93ab33c868751c9a1289d3fffff`
- **Version pointée par `latest`** : **Ubuntu 26.04 LTS « resolute »**
  - Tags : `26.04`, `resolute`, `latest` (et `rolling` au moment de la consultation)
  - Digest/ID : `sha256:f144425ff09be612d6d9ad965196e9cdc23dae1f42110a8a11a3e9a8198759f7`

#### Commentaire :
Le tag `latest` ne pointe donc pas vers la version la plus récente, mais vers la dernière **LTS** (ici Ubuntu 26.04 « resolute »), choisie pour sa stabilité et son support long. La version la plus récente (26.10 « stonking ») est une version **intermédiaire**, publiée plus fréquemment mais supportée moins longtemps. Les deux versions ont des digests différents.

---

### 4.5 Vulnérabilités de la version la plus récente et de `latest`

#### Commande(s) :
- Consultation de l’onglet **Vulnerabilities** sur Docker Hub :
  - `https://hub.docker.com/layers/library/ubuntu/latest/images/sha256-88a381d5b5eeb2b35d3ad70925a362c37ce569daf43ede89ff818ec20e4d3794?tab=vulnerabilities`
  - `https://hub.docker.com/layers/library/ubuntu/26.10/images/sha256-b18403eb2d06baad26fe088450af83744e456d0e7c46d8319d460ce3a39e1f16?tab=vulnerabilities`

#### Résultat(s) :
Les deux versions analysées présentent le même total de **13 vulnérabilités** :

| Version | Total | Critical | High | Medium | Low | Unknown |
|---|---:|---:|---:|---:|---:|---:|
| `latest` (26.04) | 13 | 2 | 8 | 0 | 0 | 3 |
| `26.10` | 13 | 2 | 8 | 0 | 0 | 3 |

Détail des vulnérabilités :
- **Critical** : `CVE-2026-56857` (CVSS 9.8), `CVE-2026-78663` (9.1)
- **High** : `CVE-2026-94440`, `CVE-2026-78669`, `CVE-2026-78659`, `CVE-2026-78667`, `CVE-2026-78660`, `CVE-2026-56866`, `CVE-2026-94439`, `CVE-2026-97031` (toutes à 7.5)
- **Unknown** : `CVE-2026-97032`, `CVE-2026-97030`, `CVE-2026-94448`

Toutes concernent le paquet `golang / stdlib / 1.26.7` et aucune n’est indiquée comme corrigeable dans Docker Hub (`0 Fixable`).

#### Commentaire :
Les images `ubuntu:latest` (26.04 LTS) et `ubuntu:26.10` partagent ici exactement les mêmes vulnérabilités, car elles embarquent la même version de la bibliothèque Go (`golang/stdlib 1.26.7`). Docker Hub affiche le total, la répartition par sévérité et la liste des CVE pour chaque architecture.

---

### 4.6 Classement des vulnérabilités

#### Commande(s) :
- Observation de l’onglet **Vulnerabilities** sur Docker Hub

#### Résultat(s) :
Docker Hub / Docker Scout classe les vulnérabilités :
- par **sévérité** décroissante : **Critical (C)**, **High (H)**, **Medium (M)**, **Low (L)**, **Unknown (U)** ;
- avec le **score CVSS** (ex. `9.8`, `9.1`, `7.5`) ;
- dans un tableau à colonnes : `CVE ID`, `Severity`, `Fixable`, `Present in`, `Affected package(s)`.

Dans le cas observé :
- `C` = 2, `H` = 8, `M` = 0, `L` = 0, `U` = 3 ;
- aucune vulnérabilité n’est marquée `Fixable` (`0 Fixable`) ;
- les vulnérabilités sont triées par gravité, les `Critical` en tête.

#### Commentaire :
La classification repose donc principalement sur la **sévérité CVSS**, avec un tri décroissant. Des filtres permettent en plus de n’afficher que les vulnérabilités corrigeables ou d’exclure certaines catégories.

---

### 4.7 Comparaison de `latest` et `noble`

#### Commande(s) :
- Consultation des tags : `https://hub.docker.com/_/ubuntu/tags`
- Comparaison des deux images via Docker Hub (onglets Vulnérabilités et Layers)

#### Résultat(s) :
- `latest` → Ubuntu **26.04 LTS « resolute »** (la dernière LTS)
- `noble` → Ubuntu **24.04 LTS « noble »**
- Les deux sont des versions **LTS**, supportées 5 ans, mais `noble` est une LTS plus ancienne et `latest` suit la LTS courante.
- Vulnérabilités au moment de la consultation :
  - `latest` (26.04) : 13 (2 Critical, 8 High, 3 Unknown)
  - `noble` (24.04) : 21 (0 Critical, 1 High, 13 Medium, 7 Low)
- Taille de la couche de base amd64 : 26.04 ≈ 41,84 Mo ; 24.04 ≈ 29,76 Mo.

#### Commentaire :
L’intérêt est de laisser le choix entre :
- **`latest`** : rester sur la LTS la plus récente, avec des paquets plus modernes ;
- **`noble`** : figer une LTS plus ancienne, plus mature et déjà validée par l’écosystème, tout en bénéficiant encore du support long.

Cela évite aussi de subir automatiquement le changement de `latest` lors du passage à la LTS suivante : un tag comme `noble` reste attaché à la version 24.04.

---

### 4.8 Comparaison des couches de `latest` et `noble`

#### Commande(s) :
- Consultation des couches via Docker Hub :
  - `https://hub.docker.com/layers/library/ubuntu/latest/images/sha256-88a381d5b5eeb2b35d3ad70925a362c37ce569daf43ede89ff818ec20e4d3794?tab=layers`
  - `https://hub.docker.com/layers/library/ubuntu/noble/images/sha256-f610ab94648195aa356059f5b41d6085c9d4d903c072430cdd1af7bdb646106b?tab=layers`

#### Résultat(s) :
- `ubuntu:latest` (26.04) — **7 couches** :
  1. `umoci raw add-layer ... --tag 26.04` → **41,84 Mo** (couche de base)
  2. `umoci config ...` (clear entrypoint/cmd) → 0 B
  3. `umoci config ... --config.cmd /bin/bash` → 0 B
  4. `umoci config ... --config.env PATH=...` → 0 B
  5. `umoci config ... --config.label ... version=26.04` → 0 B
  6. `umoci config ... --manifest.annotation ...` → 0 B
  7. `umoci raw add-layer ... .temp_layer.control_data.tar` → 388 B
- `ubuntu:noble` (24.04) — **5 couches** :
  1. `ARG RELEASE` → 0 B
  2. `ARG LAUNCHPAD_BUILD_ARCH` → 0 B
  3. `LABEL org.opencontainers.image.version=24.04` → 0 B
  4. `ADD file:6c214... in /` → **29,76 Mo** (couche de base)
  5. `CMD ["/bin/bash"]` → 0 B

#### Commentaire :
Les deux images sont construites différemment : `noble` (24.04) part d’un Dockerfile classique (`ARG`, `LABEL`, `ADD`, `CMD`), tandis que `latest` (26.04) est construite avec **Rockcraft/umoci** (`umoci raw add-layer`, `umoci config`). Elles diffèrent donc **dès la première couche** : la couche de base (rootfs) n’a ni le même contenu ni la même taille, et les couches de configuration suivantes sont également propres à chaque version. Aucune couche commune n’apparaît entre ces deux images amd64.

---

### 4.9 Répertoire `layerdb` des couches d’images

#### Commande(s) :
- `sudo ls -la /var/lib/docker/image/overlay2/layerdb/sha256`
- `sudo ls -la /var/lib/docker/image`
- `sudo find /var/lib/docker -maxdepth 5 -type d -name 'sha256'`
- `sudo ls -la /var/lib/containerd`
- `sudo ls -la /var/lib/containerd/io.containerd.snapshotter.v1.overlayfs`
- `sudo find /var/lib/containerd -maxdepth 5 -type d -name 'sha256'`

#### Résultat(s) :
- Le chemin demandé n’existe pas :
  ```text
  ls: cannot access '/var/lib/docker/image/overlay2/layerdb/sha256': No such file or directory
  ```
- `/var/lib/docker/image` ne contient qu’un cache :
  ```text
  total 32
  drwx------  2 root root  4096 oct.  14 09:02 .
  drwx--x--- 12 root root  4096 oct.  15 08:16 ..
  -rw-------  1 root root 32768 oct.  15 08:46 identity-cache.db
  ```
- `find` ne trouve aucun répertoire `sha256` sous `/var/lib/docker`.
- Sous `/var/lib/containerd`, on trouve bien les stockages containerd :
  ```text
  io.containerd.content.v1.content
  io.containerd.snapshotter.v1.overlayfs
  ...
  ```
  et le contenu adressable :
  ```text
  /var/lib/containerd/io.containerd.content.v1.content/blobs/sha256
  ```

#### Commentaire :
Le répertoire `overlay2/layerdb/sha256` correspond au driver de stockage historique `overlay2`. Dans cette VM, Docker utilise le driver `overlayfs` via le **snapshotter containerd** (`driver-type: io.containerd.snapshotter.v1`), ce qui explique l’absence de ce chemin. Les couches et blobs sont alors gérés par containerd : le contenu adressable se trouve dans `/var/lib/containerd/io.containerd.content.v1.content/blobs/sha256`, et les snapshots dans `/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/`.

---

### 4.10 Téléchargement de l’image `ubuntu` par défaut

#### Commande(s) :
- `docker pull ubuntu`

#### Résultat(s) :
```text
Using default tag: latest
latest: Pulling from library/ubuntu
06ad70e463aa: Pull complete
4e07a0f12b2c: Pull complete
8f70d2bfe91a: Download complete
Digest: sha256:f144425ff09be612d6d9ad965196e9cdc23dae1f42110a8a11a3e9a8198759f7
Status: Downloaded newer image for ubuntu:latest
docker.io/library/ubuntu:latest
```

#### Commentaire :
Sans tag précisé, Docker utilise le tag `latest`. Au moment de la consultation, `ubuntu:latest` pointe vers **Ubuntu 26.04 LTS « resolute »** (digest `sha256:f144425…759f7`), ce qui correspond à ce qui avait été identifié en Q4.4.

---

### 4.11 Téléchargement de la version la plus récente (`26.10`)

#### Commande(s) :
- `docker pull ubuntu:26.10`

#### Résultat(s) :
```text
26.10: Pulling from library/ubuntu
d4a7338d7947: Pull complete
ae58d958c2ee: Pull complete
6582b5695784: Download complete
Digest: sha256:ee126c2fa0249079a7e24ae3a3d29b04783ef93ab33c868751c9a1289d3fffff
Status: Downloaded newer image for ubuntu:26.10
docker.io/library/ubuntu:26.10
```

#### Commentaire :
Le tag `26.10` correspond à **Ubuntu 26.10 « stonking »**, la version la plus récente. Son digest (`sha256:ee126c…fffff`) est différent de celui de `latest`, ce qui confirme qu’il s’agit bien de deux images distinctes.

---

### 4.12 Filtre sur les images `ubuntu`

#### Commande(s) :
- `docker images --filter=reference='ubuntu*'`

#### Résultat(s) :
```text
IMAGE           ID             DISK USAGE   CONTENT SIZE   EXTRA
ubuntu:26.10    ee126c2fa024        174MB           49MB
ubuntu:latest   f144425ff09b        162MB         45.6MB
```

#### Commentaire :
Le filtre `reference=` ne conserve que les images dont le nom correspond au motif `ubuntu*`. On vérifie ainsi que seules les deux images Ubuntu sont présentes localement.

---

### 4.13 Commande lancée par défaut dans l’image `ubuntu`

#### Commande(s) :
- `docker inspect ubuntu --format '{{.Config.Cmd}}'`

#### Résultat(s) :
```text
[/bin/bash]
```

#### Commentaire :
La propriété `Config.Cmd` de l’image définit la commande lancée par défaut lorsqu’on crée un conteneur à partir de l’image `ubuntu`. Ici, il s’agit de **`/bin/bash`**. Comme ce n’est pas un processus en boucle et qu’il attend une entrée standard, un conteneur `ubuntu` lancé sans terminal interactif s’arrête immédiatement.

---

## Partie 5 – Interagir avec un conteneur

Remarque préliminaire : la combinaison **Ctrl-D** ferme l’entrée standard d’une commande Unix (EOF). Un shell interactif qui la reçoit se termine, et tout processus qui attend des données sur son entrée standard s’arrête lorsque celle-ci est fermée. C’est ce mécanisme qui explique pourquoi un conteneur dont la commande par défaut est un shell (comme `ubuntu`) s’arrête immédiatement s’il est lancé sans terminal interactif.

### 5.1 Lancement, statut, redémarrage et suppression d’un conteneur `ubuntu`

#### Commande(s) :
- `docker run ubuntu`
- `docker ps -a`
- `docker start -ai thirsty_cori`
- `docker rm 32b867bf56db`

#### Résultat(s) :
- La création ne produit aucune sortie et le conteneur s’arrête aussitôt : il apparaît **à l’arrêt** (`Exited (0)`) sous le nom `thirsty_cori` (ID `32b867bf56db`).
- `docker start -ai thirsty_cori` ne rend pas la main dans un shell : le conteneur se termine immédiatement.
- `docker rm 32b867bf56db` supprime le conteneur.

#### Commentaire :
`ubuntu` a pour commande par défaut `/bin/bash`. Lancé sans `-i`/`-t`, le shell n’a pas de terminal et son entrée standard est immédiatement fermée (EOF) : il se termine tout de suite, d’où le statut `Exited (0)`. Un conteneur ne reste « en exécution » que si sa commande principale continue de tourner.

---

### 5.2 Conteneur `os_ubuntu` interactif et attaché à un terminal

#### Commande(s) :
- `docker run -it --name os_ubuntu ubuntu`

#### Résultat(s) :
```text
root@01d487cc57e4:/#
```

#### Commentaire :
`-i` garde l’entrée standard ouverte et `-t` alloue un pseudo-terminal : on obtient un shell interactif attaché au terminal. Nommer le conteneur (`os_ubuntu`) évite de manipuler son ID.

---

### 5.3 Commande correspondant au processus PID 1

#### Commande(s) :
- `ps -p 1 -o pid,cmd`

#### Résultat(s) :
```text
    PID CMD
      1 /bin/bash
```

#### Commentaire :
Le PID 1 est le processus principal du conteneur : ici `/bin/bash`, hérité du `CMD` de l’image `ubuntu`. C’est lui qui reçoit les signaux ; sa terminaison arrête le conteneur.

---

### 5.4 `whoami`, `pwd`, `ls` et `hostname` dans le conteneur

#### Commande(s) :
- `whoami`
- `pwd`
- `ls`
- `hostname`

#### Résultat(s) :
```text
root
/
bin   dev  home  lib64  mnt  proc  run   srv  tmp  var
boot  etc  lib   media  opt  root  sbin  sys  usr
01d487cc57e4
```

#### Commentaire :
Dans le conteneur, l’utilisateur est `root`, le répertoire courant est `/` et le nom d’hôte est l’ID du conteneur (`01d487cc57e4`). Le conteneur possède donc sa propre racine (système de fichiers Ubuntu) et son propre hostname, distincts de ceux de la VM.

---

### 5.5 Statut observé depuis un second terminal

#### Commande(s) :
- `docker ps`

#### Résultat(s) :
- Le conteneur `os_ubuntu` (ID `01d487cc57e4`) est en **`Up`**.
- Sa commande est **`/bin/bash`**.
- La colonne **`PORTS`** est vide : **aucun port n’est publié**.

#### Commentaire :
Depuis un second terminal, `docker ps` affiche le même conteneur : celui-ci est indépendant de la session qui l’a lancé. À part le statut `Up`, on constate qu’il ne publie aucun port sur la VM (pas d’option `-p`) et qu’il exécute simplement un shell.

---

### 5.6 Déplacement dans le répertoire home

#### Commande(s) :
- `cd ~`
- `pwd`

#### Résultat(s) :
```text
/root
```

#### Commentaire :
`~` désigne le répertoire personnel de l’utilisateur courant, soit `/root` pour `root`. Le prompt devient `root@01d487cc57e4:~#`.

---

### 5.7 Sortie avec la commande Unix classique

#### Commande(s) :
- `exit`
- `docker ps -a`

#### Résultat(s) :
- Le conteneur `os_ubuntu` passe en **`Exited (0)`**.
- Il reste présent dans `docker ps -a`.

#### Commentaire :
`exit` termine le shell PID 1 ; comme c’est le processus principal, le conteneur s’arrête. Le code retour `0` indique une sortie normale. Le conteneur n’est pas supprimé, seulement arrêté.

---

### 5.8 Redémarrage interactif et répertoire courant

#### Commande(s) :
- `docker start -ai os_ubuntu`

#### Résultat(s) :
- Le prompt obtenu est :
  ```text
  root@01d487cc57e4:/#
  ```

#### Commentaire :
Le répertoire courant est **`/`**, et non `/root`. Le conteneur redémarre un **nouveau** processus bash : le répertoire courant n’est pas conservé d’un shell à l’autre (l’ancien a disparu), il repart du répertoire de travail défini par l’image (`WorkingDir`, vide ici, donc `/`). Le système de fichiers, lui, est bien conservé.

---

### 5.9 Inspection du conteneur et récupération du PID

#### Commande(s) :
- `docker inspect os_ubuntu`

#### Résultat(s) :
Extrait du JSON renvoyé :
```json
"State": {
    "Status": "running",
    "Running": true,
    "Paused": false,
    "Restarting": false,
    "OOMKilled": false,
    "Dead": false,
    "Pid": 4222,
    "ExitCode": 0,
    ...
}
```

#### Commentaire :
`docker inspect` renvoie l’ensemble des métadonnées du conteneur. `State.Running` vaut `true` (le conteneur tourne) et `State.Pid` donne le **PID du processus principal vu depuis la VM** : ici **4222**.

---

### 5.10 Extraction du PID avec `jq`

#### Commande(s) :
- `docker inspect os_ubuntu | jq '.[0].State.Pid'`

#### Résultat(s) :
```text
4222
```

#### Commentaire :
`docker inspect` produit un tableau JSON (`[ {...} ]`). Le filtre `jq` `.[0].State.Pid` extrait directement le PID, sans avoir à lire tout le JSON.

---

### 5.11 Retrouver le processus correspondant sur la VM

#### Commande(s) :
- `ps -p 4222 -o pid,ppid,cmd`

#### Résultat(s) :
```text
    PID    PPID CMD
   4222    4198 /bin/bash
```

#### Commentaire :
Le PID 4222 est bien un processus **visible depuis la VM** : c’est le PID 1 du conteneur (le `/bin/bash`). Son parent (PPID `4198`) est le processus intermédiaire (shim/runc) qui gère le conteneur. La conteneurisation n’est donc pas une machine virtuelle : ce sont des processus ordinaires, isolés par les namespaces et limités par les cgroups.

---

### 5.12 Retour dans le home

#### Commande(s) :
- `cd ~`
- `pwd`

#### Résultat(s) :
```text
/root
```

#### Commentaire :
Le shell (toujours le même processus) revient dans `/root`.

---

### 5.13 Détachement avec Ctrl-p Ctrl-q

#### Commande(s) :
- `Ctrl-p Ctrl-q`
- `docker ps`

#### Résultat(s) :
```text
CONTAINER ID   IMAGE     COMMAND       CREATED         STATUS         PORTS     NAMES
01d487cc57e4   ubuntu    "/bin/bash"   5 minutes ago   Up 2 minutes             os_ubuntu
```

#### Commentaire :
`Ctrl-p Ctrl-q` **détache** le terminal du conteneur sans l’arrêter : on revient au prompt de la VM et le conteneur reste **`Up`**. C’est la différence avec `exit`, qui termine le shell donc le conteneur.

---

### 5.14 Rattachement et ajout d’un fichier

#### Commande(s) :
- `docker attach os_ubuntu`
- `pwd`
- `echo "test depuis os_ubuntu" > fichier_test.txt`
- `ls -l`
- `cat fichier_test.txt`

#### Résultat(s) :
```text
root@01d487cc57e4:~# pwd
/root
root@01d487cc57e4:/# cd /root
pwd
echo "test depuis os_ubuntu" > fichier_test.txt
ls -l
cat fichier_test.txt
/root
total 4
-rw-r--r-- 1 root root 22 Oct 15 09:34 fichier_test.txt
test depuis os_ubuntu
```

#### Commentaire :
`docker attach` rebranche le terminal sur le shell existant : on retrouve le même bash et donc le répertoire courant **`/root`**. Le fichier `fichier_test.txt` est créé dans le système de fichiers **du conteneur** (et non de la VM), ce que confirmera `docker diff`.

---

### 5.15 Différences dans le système de fichiers du conteneur

#### Commande(s) :
- `Ctrl-p Ctrl-q`
- `docker diff os_ubuntu`

#### Résultat(s) :
```text
C /root
A /root/.bash_history
A /root/fichier_test.txt
```

#### Commentaire :
`docker diff` compare la couche modifiable (au-dessus de l’image) avec l’image d’origine. `A` = fichier ajouté, `C` = modifié. On retrouve le fichier créé (`A /root/fichier_test.txt`) et l’historique du shell (`A /root/.bash_history`), ainsi que le répertoire `/root` modifié (`C`). Les modifications faites dans un conteneur restent donc locales à sa couche d’écriture.

---

### 5.16 Exécuter `hostname` sans foreground

#### Commande(s) :
- `docker exec -d os_ubuntu hostname`
- `docker exec os_ubuntu hostname`

#### Résultat(s) :
- `docker exec -d …` : **aucune sortie** (commande lancée en arrière-plan).
- `docker exec os_ubuntu hostname` :
  ```text
  01d487cc57e4
  ```

#### Commentaire :
`docker exec` exécute une commande **dans un conteneur déjà démarré**, sans l’arrêter. L’option `-d` (detached) la lance en arrière-plan ; sans `-d`, on voit sa sortie. Le `hostname` confirme l’ID du conteneur.

---

### 5.17 Exécuter `bash` en interactif et compter les processus

#### Commande(s) :
- `docker exec -it os_ubuntu bash`
- `ps -ef | grep bash`

#### Résultat(s) :
```text
root           1       0  0 09:34 pts/0    00:00:00 /bin/bash
root          36       0  0 09:35 pts/1    00:00:00 bash
root          43      36  0 09:36 pts/1    00:00:00 grep --color=auto bash
```

#### Commentaire :
Deux **bash** tournent dans le conteneur : le PID 1 (`/bin/bash`, le shell d’origine) et le PID 36 (`bash`, celui lancé par `docker exec -it`). La troisième ligne est le `grep` lui-même. On peut donc ouvrir plusieurs shells dans un même conteneur avec `exec`.

---

### 5.18 Quitter puis afficher les processus du conteneur

#### Commande(s) :
- `exit`
- `docker top os_ubuntu`

#### Résultat(s) :
```text
UID                 PID                 PPID                C                   STIME               TTY                 TIME                CMD
root                4450                4426                0                   09:34               pts/0               00:00:00            /bin/bash
```

#### Commentaire :
`exit` quitte le bash du `docker exec` **sans arrêter** le conteneur (le PID 1 continue). `docker top` affiche les processus du conteneur **tels que vus par la VM** ; il ne reste alors que le PID 1 `/bin/bash`.

---

### 5.19 Revenir dans le conteneur et l’arrêter en le quittant

#### Commande(s) :
- `docker attach os_ubuntu`
- `exit`
- `docker ps -a`

#### Résultat(s) :
- `os_ubuntu` apparaît en **`Exited (0)`**.

#### Commentaire :
Cette fois, `exit` termine le processus **PID 1** : le conteneur s’arrête (code `0`). Il reste stocké et pourra être relancé.

---

### 5.20 Conteneur `ll` avec la commande `ls -l`

#### Commande(s) :
- `docker run --name ll ubuntu ls -l`

#### Résultat(s) :
```text
total 48
lrwxrwxrwx   1 root root    7 Apr 20 08:46 bin -> usr/bin
drwxr-xr-x   2 root root 4096 Apr 20 08:46 boot
drwxr-xr-x   5 root root  340 Oct 15 09:38 dev
drwxr-xr-x   1 root root 4096 Oct 15 09:38 etc
drwxr-xr-x   3 root root 4096 Sep 27 10:25 home
lrwxrwxrwx   1 root root    7 Apr 20 08:46 lib -> usr/lib
lrwxrwxrwx   1 root root    9 Apr 20 08:46 lib64 -> usr/lib64
drwxr-xr-x   2 root root 4096 Sep 27 10:24 media
drwxr-xr-x   2 root root 4096 Sep 27 10:24 mnt
drwxr-xr-x   2 root root 4096 Sep 27 10:24 opt
dr-xr-xr-x 187 root root    0 Oct 15 09:38 proc
drwx------   2 root root 4096 Sep 27 10:25 root
drwxr-xr-x   4 root root 4096 Sep 27 10:25 run
lrwxrwxrwx   1 root root    8 Apr 20 08:46 sbin -> usr/sbin
drwxr-xr-x   2 root root 4096 Sep 27 10:24 srv
dr-xr-xr-x  13 root root    0 Oct 15 09:38 sys
drwxrwxrwt   2 root root 4096 Sep 27 10:25 tmp
drwxr-xr-x  12 root root 4096 Sep 27 10:24 usr
drwxr-xr-x  11 root root 4096 Sep 27 10:25 var
```

#### Commentaire :
La commande placée après le nom de l’image **remplace** le `CMD` par défaut (`/bin/bash`) : le conteneur exécute `ls -l` puis se termine (`Exited (0)`). Le conteneur est nommé `ll`.

---

### 5.21 Redémarrer `ll` pour le même affichage

#### Commande(s) :
- `docker start -ai ll`

#### Résultat(s) :
Le même listing qu’en 5.20 — seuls quelques horodatages de `/dev`, `/proc` et `/sys` changent, car ces pseudo-systèmes de fichiers sont regénérés à chaque démarrage :
```text
total 48
lrwxrwxrwx   1 root root    7 Apr 20 08:46 bin -> usr/bin
drwxr-xr-x   2 root root 4096 Apr 20 08:46 boot
drwxr-xr-x   5 root root  340 Oct 15 09:39 dev
...
```

#### Commentaire :
Redémarrer un conteneur **rejoue sa commande** (`ls -l`) et réaffiche donc la même chose. Le conteneur `ll` conserve sa configuration ; il relance simplement son processus.

---

### 5.22 Conteneur `ps` avec `ps aux` et suppression automatique

#### Commande(s) :
- `docker run --rm --name ps ubuntu ps aux`
- `docker ps -a`

#### Résultat(s) :
```text
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1 40.0  0.1   6752  3504 ?        Rs   09:39   0:00 ps aux
```
`docker ps -a` ne montre **pas** de conteneur `ps` (il ne reste que `ll` et `os_ubuntu`) :
```text
CONTAINER ID   IMAGE     COMMAND       CREATED          STATUS                     PORTS     NAMES
7720d582f5b4   ubuntu    "ls -l"       34 seconds ago   Exited (0) 21 seconds ago             ll
01d487cc57e4   ubuntu    "/bin/bash"   16 minutes ago   Exited (0) About a minute ago             os_ubuntu
```

#### Commentaire :
Dans le conteneur, `ps aux` est bien le processus **PID 1** : c’est la commande principale. Grâce à `--rm`, le conteneur est **automatiquement supprimé** dès qu’il s’arrête : il n’apparaît donc pas dans `docker ps -a`.

---

### 5.23 Conteneur `salut` avec `echo Bonjour`, puis relance

#### Commande(s) :
- `docker run --name salut ubuntu echo Bonjour`
- `docker start -ai salut`

#### Résultat(s) :
```text
Bonjour
```
puis, au redémarrage :
```text
Bonjour
```

#### Commentaire :
`echo Bonjour` s’exécute une fois, puis le conteneur s’arrête. `docker start -ai salut` **rejoue** la même commande et réaffiche `Bonjour`. Un conteneur arrêté n’est pas supprimé : il peut être relancé autant de fois que voulu.

---

### 5.24 Conteneur à commande infinie, puis suppression

#### Commande(s) :
- `docker run -d --name boucle ubuntu sh -c "while true ; do sleep 3600 ; done"`
- `docker ps`
- `docker rm -f boucle`

#### Résultat(s) :
```text
8e94a4e9114f1fd11bd43e10d2001c152e56c88e25f4da3e088fc80dbdbd5600
```
```text
CONTAINER ID   IMAGE     COMMAND                  CREATED         STATUS         PORTS     NAMES
8e94a4e9114f   ubuntu    "sh -c 'while true ;…"   6 seconds ago   Up 5 seconds             boucle
```
```text
boucle
```

#### Commentaire :
La commande `while true ; do sleep 3600 ; done` ne se termine jamais : il faut donc la lancer en **arrière-plan** (`-d`), sinon le terminal resterait bloqué. Comme le conteneur tourne, sa suppression nécessite de **forcer** l’arrêt avec `docker rm -f`.

---

### 5.25 `os_ubuntu` en interactif avec un affichage périodique

#### Commande(s) :
- `docker start -ai os_ubuntu`
- `while true; do echo salut; sleep 3; done`

#### Résultat(s) :
```text
salut
salut
salut
...
```
`salut` s’affiche **toutes les 3 secondes**.

#### Commentaire :
La boucle tourne dans le shell du conteneur et écrit sur la sortie standard, qui est capturée dans le journal du conteneur (voir 5.28).

---

### 5.26 Rattachement depuis le second terminal et observation

#### Commande(s) :
- `docker attach os_ubuntu`
- `Ctrl-C`
- `ls`

#### Résultat(s) :
- Le second terminal affiche les mêmes `salut` : les deux terminaux sont **synchronisés** (même TTY).
- `Ctrl-C` envoie un signal d’interruption au processus au premier plan et **stoppe la boucle** ; le prompt `root@01d487cc57e4:/#` revient dans les deux terminaux.
- La commande `ls` lancée dans un terminal s’affiche **aussi dans l’autre**.

#### Commentaire :
`docker attach` branche un second client sur l’entrée/sortie du conteneur : les deux terminaux partagent donc le même terminal (mêmes entrées, mêmes sorties). Cela permet d’interagir depuis l’extérieur avec un conteneur lancé dans un autre terminal.

---

### 5.27 Sortie avec un code retour non nul

#### Commande(s) :
- `exit 1`
- `docker ps -a`

#### Résultat(s) :
```text
CONTAINER ID   IMAGE     COMMAND          CREATED          STATUS                     PORTS     NAMES
463690b0e46d   ubuntu    "echo Bonjour"   2 minutes ago    Exited (0) 2 minutes ago             salut
7720d582f5b4   ubuntu    "ls -l"          3 minutes ago    Exited (0) 3 minutes ago             ll
01d487cc57e4   ubuntu    "/bin/bash"      19 minutes ago   Exited (1) 9 seconds ago             os_ubuntu
```

#### Commentaire :
`exit 1` termine le shell **PID 1** avec le code de retour `1`. Ce code est **propagé au conteneur**, qui apparaît donc en `Exited (1)` : un code non nul signale une terminaison en erreur.

---

### 5.28 Voir les logs du conteneur

#### Commande(s) :
- `docker logs os_ubuntu`

#### Résultat(s) :
Extrait du journal (sortie standard du conteneur) :
```text
root@01d487cc57e4:/# ps -p 1 -o pid,cmd
    PID CMD
      1 /bin/bash
root@01d487cc57e4:/# whoami
...
root@01d487cc57e4:/# while true; do echo salut; sleep 3; done
salut
salut
salut
...
^C
root@01d487cc57e4:/# exit 1
```

#### Commentaire :
`docker logs` affiche la sortie standard et la sortie d’erreur du **PID 1**, conservées après l’arrêt du conteneur. Elles incluent l’écho du terminal (dû à `-t`) et les `salut` de la boucle.

---

### 5.29 Retrouver le fichier de journal

#### Commande(s) :
- `docker inspect os_ubuntu | jq '.[0].LogPath'`
- `sudo cat <chemin>`

#### Résultat(s) :
```text
"/var/lib/docker/containers/01d487cc57e404c93e0692a14782f5e19282d0fff13d82fc2e029038967c0db0/01d487cc57e404c93e0692a14782f5e19282d0fff13d82fc2e029038967c0db0-json.log"
```
Le fichier contient une ligne JSON par évènement :
```json
{"log":"\u001b[?2004l\rsalut\r\n","stream":"stdout","time":"2026-10-15T09:40:54.739716396Z"}
{"log":"salut\r\n","stream":"stdout","time":"2026-10-15T09:40:57.746935657Z"}
...
```

#### Commentaire :
Le champ `LogPath` de `docker inspect` donne le chemin du fichier où Docker stocke les logs du conteneur, au format JSON (une ligne par écriture, avec le flux et l’horodatage). C’est ce fichier que `docker logs` lit et remet en forme.

---

### 5.30 Supprimer tous les conteneurs arrêtés

#### Commande(s) :
- `docker container prune`
- `docker ps -a`

#### Résultat(s) :
```text
WARNING! This will remove all stopped containers.
Are you sure you want to continue? [y/N] y
Deleted Containers:
463690b0e46dd585c0475e78a5da43a91ad3c21f7326a0453b0fe98993b05c4a
7720d582f5b44b5c7395a6dc6f3dcbc9765bb59af6216d52e03034abfa5e5d72
01d487cc57e404c93e0692a14782f5e19282d0fff13d82fc2e029038967c0db0

Total reclaimed space: 24.58kB
```
`docker ps -a` ne liste plus aucun conteneur.

#### Commentaire :
`docker container prune` supprime **tous** les conteneurs **arrêtés** (avec confirmation), ce qui libère l’espace de leurs couches modifiables. Les conteneurs en cours d’exécution ne sont pas touchés.

---

### 5.31 Clôture : arrêter et supprimer tous les conteneurs actifs en une seule commande

#### Commande(s) :
- `docker rm -f $(docker ps -aq)`

#### Résultat(s) :
```text
docker: 'docker rm' requires at least 1 argument

Usage:  docker rm [OPTIONS] CONTAINER [CONTAINER...]

See 'docker rm --help' for more information
```

#### Commentaire :
La substitution `$(docker ps -aq)` renvoie la liste de tous les ID de conteneurs ; ici elle est **vide** (tous avaient déjà été supprimés en 5.30), donc `docker rm -f` n’a plus d’argument. Prise dans le cas général, cette commande unique **arrête (`-f`) puis supprime** tous les conteneurs existants.

---

