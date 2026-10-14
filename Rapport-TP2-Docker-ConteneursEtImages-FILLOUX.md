# Rapport du TP2 de « cours en question »

## Sommaire

- Partie 2

Environnement : VM `vmdocker`, utilisateur `dockeruser` (Docker installé via le dépôt apt).

---

## Partie 2 – Installation et démarrage de Docker

### A. Version de Docker

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

### B. Composants du démon et services/sockets

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

### C. Arrêt de Docker

#### Commande(s) :
- `sudo systemctl stop docker docker.socket`

#### Résultat(s) :
- `docker.service` → `inactive (dead)`
- `docker.socket` → `inactive (dead)` (n’écoute plus sur `/run/docker.sock`)

#### Commentaire :
Il faut arrêter le socket en même temps que le service, sinon le socket peut réveiller Docker à la prochaine commande.

---

### D. Désactivation / réactivation

#### Commande(s) :
- Désactivation : `sudo systemctl disable --now docker` (+ `sudo systemctl disable docker.socket`)
- Réactivation : `sudo systemctl enable --now docker docker.socket containerd`

#### Résultat(s) :
- Vérification : `systemctl is-enabled docker docker.socket containerd`
- Vérification : `systemctl is-active docker docker.socket containerd`

#### Commentaire :
`disable` empêche le démarrage automatique au boot ; `enable` le rétablit.

---

### E. Utilisation sans sudo

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
