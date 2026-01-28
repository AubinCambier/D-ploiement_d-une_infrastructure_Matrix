## 5.0 Pre-requis : configuration et interconnexion (PC de TP)

### 1. Schema d'architecture reseau (vue logique)

L'architecture repose sur une segmentation des services. Le flux entrant est capte par le reverse proxy qui redistribue les requetes vers les VMs concernees via le reseau interne.

```text
      [ PC DE TP IUT ]
              |
      (Tunnel SSH Port 9090)
              |
     _________________|_________________
    |         VM RPROXY (10.42.XX.2)    | <--- Point d'entree
    |___________________________________|
              |                |
      (Requetes Web)     (Requetes Web)
      ________|_______   _______|________
     |  VM ELEMENT    | |   VM MATRIX    |
     |  (10.42.XX.4)  | |  (10.42.XX.1)  |
     |________________| |________________|
                               |
                        (Flux Database)
                         ______|_________
                        | VM POSTGRESQL  |
                        |  (10.42.XX.3)  |
                        |________________|
```

---

### 2. Actions de configuration (contexte PC de TP)

#### A. Fixation des adresses IP (configuration statique)

Sur chaque VM, le fichier `/etc/network/interfaces` doit etre configure en statique pour garantir que les services pointent toujours vers les bonnes cibles.

Exemple pour la VM `rproxy` :

```bash
auto enp0s3
iface enp0s3 inet static
    address 10.42.XX.2
    netmask 255.255.0.0
    gateway 10.42.0.1
```

Action : redemarrer le service avec :

```bash
sudo systemctl restart networking
```

#### B. Identification (hostname)

Chaque VM doit etre identifiable pour eviter les erreurs d'administration :

```bash
sudo hostnamectl set-hostname [nom_de_la_vm]
```

#### C. Annuaire DNS local (`/etc/hosts`)

Indispensable pour la communication par noms de domaine. A copier sur les 4 VMs et sur le PC de TP :

```text
127.0.0.1       localhost
10.42.XX.2      rproxy.groseillier.iutinfo.fr   rproxy
10.42.XX.1      matrix.groseillier.iutinfo.fr   matrix
10.42.XX.3      postgresql.groseillier.iutinfo.fr   postgresql
10.42.XX.4      element.groseillier.iutinfo.fr  element
```

#### D. Securisation et acces SSH

Deploiement des cles depuis le PC de TP :

```bash
ssh-copy-id user@10.42.XX.YY
```

Verification des droits :

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

---

### 3. Section dediee aux problemes (troubleshooting)

| Probleme | Cause racine | Solution appliquee |
| --- | --- | --- |
| Destination Host Unreachable | La VM est allumee mais son interface `enp0s3` n'a pas d'IP active (confirme par `vmiut info` vide). | Redemarrer la VM ou forcer l'IP via la console graphique a l'IUT. |
| Connection refused | Le service SSH ne tourne pas sur la VM cible ou port mal configure. | Verifier avec `sudo systemctl status ssh`. |
| Port 9090 already in use | Une session SSH precedente sur le PC de TP bloque le port. | Identifier et tuer le processus : `killall ssh`. |

---

### 4. Tests de validation (recette technique)

* Test 1 (connectivite) :

```bash
ping -c 3 matrix
```

Attendu : 0% de perte de paquets.

* Test 2 (resolution) :

```bash
getent hosts element
```

Attendu : doit retourner `10.42.XX.4`.

* Test 3 (privileges) :

```bash
sudo -l
```

Attendu : l'utilisateur doit avoir les droits complets pour administrer les services.
---

- Page precedente: [Partie 4 : Reverse proxy et Element](../4-proxy/README.md)
- Page suivante: [5.1 : Configuration de la VM Matrix](config-matrix-vm.md)
