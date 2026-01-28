# 5.3 Mise en place de PostgreSQL (VM `db`)


- Réseau virtuel IUT : **10.42.0.0/16**
- `XX` = identifiant (3ᵉ octet) : `10.42.XX.3`
- Adressage :

  - `matrix` : `10.42.XX.1/16`
  - `rproxy` : `10.42.XX.2/16`
  - `db` : `10.42.XX.3/16`
  - `element` : `10.42.XX.4/16`

## 1) Schéma d’architecture

```
VM matrix (10.42.XX.1)  ---- TCP/5432 ---->  VM db (10.42.XX.3)
Réseau : 10.42.0.0/16
```

---

## 2) Procédure pas à pas (sudo)

### A. Création et premier accès à la VM (depuis le PC physique)

1. Se connecter à la machine de virtualisation :

```bash
ssh virt
```

Pourquoi : `vmiut` se lance sur la machine de virtualisation.

2. Créer la VM (le nom interne est libre) :

```bash
vmiut creer db
```

Pourquoi : crée une VM Debian vierge.

3. Démarrer la VM :

```bash
vmiut demarrer db
```

Pourquoi : la VM doit tourner pour être configurée.

4. Ouvrir la console :

```bash
vmiut console db
```

Pourquoi : au début, c’est le moyen le plus fiable pour configurer le réseau, surtout quand on va couper/remonter l’interface.

5. Se connecter dans la VM (identifiants du modèle) :

- login : `user`
- mot de passe : `user`

---

### B. Pré-requis minimaux imposés (sudo, hostname, DNS local)

1. Mettre à jour + installer outils :

```bash
sudo apt update
sudo apt full-upgrade -y
sudo apt install -y sudo vim
```

Pourquoi : VM saine + `sudo` requis par le sujet.

2. Hostname :

```bash
sudo hostnamectl set-hostname db-XX
```

Pourquoi : identification claire de la VM.

3. Résolution locale (obligatoire) :

```bash
sudo nano /etc/hosts
```

Ajouter :

```text
10.42.XX.1  matrix
10.42.XX.2  rproxy
10.42.XX.3  db
10.42.XX.4  element
```

Pourquoi : le sujet impose que les VMs se joignent par nom (DNS local via `/etc/hosts`).

---

### C. Passage en IP statique (étape clé avant toute connexion SSH)

> Important : on effectue cette étape en console `vmiut`, car `ifdown` peut couper la connectivité.

1. (Recommandé) Descendre l’interface avant de modifier (selon consignes de votre cours) :

```bash
sudo ifdown enp0s3
```

Pourquoi : appliquer proprement le changement de DHCP vers statique.

2. Modifier la configuration réseau :

```bash
sudo nano /etc/network/interfaces
```

Pourquoi : c’est le fichier Debian classique pour définir l’IP statique.

Remplacer le bloc DHCP 

```text
# The primary network interface
allow-hotplug enp0s3
iface enp0s3 inet dhcp
```

Par :

```text
auto enp0s3
iface enp0s3 inet static
    address 10.42.XX.3/16
    gateway 10.42.0.1
```

3. Remonter l’interface :

```bash
sudo ifup enp0s3
```

Pourquoi : appliquer immédiatement l’IP statique.

4. Vérifier l’IP :

```bash
ip addr show enp0s3
```

Résultat attendu : `inet 10.42.XX.3/16`.

> À partir de ce moment seulement, l’IP est stable : on peut quitter la console.

---

### D. Connexion SSH vers la VM `db` (uniquement après IP statique)

#### 1) Connexion directe (sans config SSH)

Depuis le PC physique :

```bash
ssh -J virt user@10.42.XX.3
```

Pourquoi : `-J virt` fait un saut SSH via la machine de virtualisation vers le réseau privé.

Si vous n’avez pas l’alias `virt` :

```bash
ssh -J user@dattier.iutinfo.fr user@10.42.XX.3
```

#### 2) Configuration permanente dans `~/.ssh/config` (recommandé)

Sur le PC physique :

```bash
nano ~/.ssh/config
```

Ajouter :

```sshconfig
Host postgresql
    HostName 10.42.XX.3
    User user
    ProxyJump virt
```

Pourquoi : vous pourrez ensuite faire simplement :

```bash
ssh postgresql
```

---

### E. Déploiement de la clé SSH (obligatoire)

Depuis le PC physique :

```bash
ssh-copy-id -J virt user@10.42.XX.3
```

Pourquoi : installer la clé publique pour éviter le mot de passe et respecter la contrainte “déploiement de clés SSH”.

---

### F. Installation et configuration PostgreSQL

1. Installer PostgreSQL :

```bash
sudo apt install -y postgresql
```

Pourquoi : installer le serveur DB.

2. Repérer la version active :

```bash
pg_lsclusters
```

Pourquoi : connaître `<VER>` (souvent 15) pour modifier les bons fichiers.

3. Autoriser l’écoute réseau (sur l’IP de la VM db) :

```bash
sudo nano /etc/postgresql/<VER>/main/postgresql.conf
```

Mettre :

```text
listen_addresses = 'localhost,10.42.XX.3'
```

Pourquoi : accepter les connexions réseau, sans exposer inutilement sur toutes les interfaces.

4. Autoriser uniquement la VM `matrix` dans `pg_hba.conf` :

```bash
sudo nano /etc/postgresql/<VER>/main/pg_hba.conf
```

Ajouter :

```text
host    synapse   synapse_user   10.42.XX.1/32   scram-sha-256
```

Pourquoi : seul `matrix` (Synapse) peut se connecter. `/32` = une seule IP.

5. Redémarrer PostgreSQL :

```bash
sudo systemctl restart postgresql
```

Pourquoi : appliquer la config.

---

### G. Création DB + user Synapse

1. Entrer dans psql en admin :

```bash
sudo -u postgres psql
```

Pourquoi : administrer PostgreSQL via l’utilisateur `postgres`.

2. Créer DB + user :

```sql
CREATE USER synapse_user WITH ENCRYPTED PASSWORD 'CHANGEZ_MOI';
CREATE DATABASE synapse OWNER synapse_user;
GRANT ALL PRIVILEGES ON DATABASE synapse TO synapse_user;
\q
```

## 3 Automatisation : Script de réinitialisation de la base (Reset DB)

### A) Pourquoi faire cela ?

Lorsque vous changez de poste (passage de XX a YY), deux problemes surviennent :

- Cohérence : si vous aviez créé un utilisateur spécifique (ex : `user_118`), il ne correspond plus à votre nouveau poste (`user_120`).
- Conflits : il peut rester des donnees ou des sessions actives liees a l'ancienne IP qui bloquent le redemarrage propre du service.

Ce script permet de faire table rase (Drop) et de recreer une base vierge et saine (Create) en une seule commande, sans taper de SQL manuellement.

---

### B) Creation du script `reset_db.sh`

Sur la VM `db`, en tant que root (ou user avec sudo), creez ce fichier :

```bash
sudo nano /usr/local/bin/reset_db.sh
```

Collez le contenu suivant :

```bash
#!/bin/bash
# Script de reinitialisation de la base Matrix/Synapse
# A utiliser lors d'un changement de machine ou pour repartir a zero.

DB_NAME="synapse"
DB_USER="synapse_user"
# Mettez ici le mot de passe defini dans homeserver.yaml
DB_PASS="aubin"

echo "ATTENTION : Ce script va SUPPRIMER toutes les donnees de la base '$DB_NAME'."
read -p "Etes-vous sur ? (o/n) " -n 1 -r
echo
if [[ ! $REPLY =~ ^[Oo]$ ]]; then
    echo "Annulation."
    exit 1
fi

# 1. Tuer les connexions existantes (Sinon le DROP echoue)
echo "[1/4] Deconnexion des sessions actives..."
sudo -u postgres psql -c "SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE datname = '$DB_NAME';" > /dev/null 2>&1

# 2. Supprimer la base et l'utilisateur
echo "[2/4] Suppression de l'ancienne base..."
sudo -u postgres psql -c "DROP DATABASE IF EXISTS $DB_NAME;"
sudo -u postgres psql -c "DROP USER IF EXISTS $DB_USER;"

# 3. Recreation
echo "[3/4] Creation de l'utilisateur et de la base..."
sudo -u postgres psql -c "CREATE USER $DB_USER WITH ENCRYPTED PASSWORD '$DB_PASS';"
sudo -u postgres psql -c "CREATE DATABASE $DB_NAME OWNER $DB_USER;"
sudo -u postgres psql -c "GRANT ALL PRIVILEGES ON DATABASE $DB_NAME TO $DB_USER;"

# 4. Verification
echo "[4/4] Verification..."
if sudo -u postgres psql -lqt | cut -d \| -f 1 | grep -qw $DB_NAME; then
    echo "SUCCES : La base '$DB_NAME' est prete pour le nouveau XX."
else
    echo "ERREUR : La base n'a pas ete creee."
fi
```

---

### C) Rendre le script executable

```bash
sudo chmod +x /usr/local/bin/reset_db.sh
```

---

### D) Utilisation

Quand vous changez de machine, apres avoir mis a jour vos IPs et le fichier `pg_hba.conf`, lancez simplement :

```bash
sudo /usr/local/bin/reset_db.sh
```

---

### E)Probleme du script

| Probleme | Cause racine | Solution |
| --- | --- | --- |
| `database "synapse" is being accessed by other users` | Le serveur Matrix est toujours connecte et le script n'a pas reussi a tuer la session. | Arretez Synapse sur l'autre VM (`systemctl stop matrix-synapse`) avant de lancer le script. |
| `Permission denied` | Vous n'avez pas lance le script avec `sudo` ou en root. | Utilisez `sudo /usr/local/bin/reset_db.sh`. |

---

## 4) Tests de validation

| Test             | Commande                                        | Résultat attendu   |
| ---------------- | ----------------------------------------------- | ------------------ |
| IP               | `ip addr show enp0s3`                           | `10.42.XX.3/16`    |
| Résolution noms  | `getent hosts matrix db rproxy element`         | IP correctes       |
| PostgreSQL actif | `sudo systemctl status postgresql --no-pager`   | `active (running)` |
| Port 5432 écoute | `sudo ss -lntp \| grep 5432`                    | écoute sur 5432    |
| DB listée        | `sudo -u postgres psql -c "\l" \| grep synapse` | `synapse` apparaît |

Test depuis `matrix` (vrai test bout en bout) :

```bash
psql -h db -U synapse_user -d synapse -c "SELECT 1;"
```

Résultat attendu : `1`.

---

## 4) Problème

- Après `ifdown` : plus de réseau → normal. Utiliser la console `vmiut` pour terminer la config puis `ifup`.
- `ssh -J virt user@10.42.XX.3` ne marche pas : vérifier que l’IP statique est bien en place (`ip addr`) et que vous êtes sur le bon `XX`.
- `ssh postgresql` ne marche pas : vérifier `~/.ssh/config` (indentation + `ProxyJump virt`).
- `FATAL: no pg_hba.conf entry` : vérifier la ligne `/32` et l’IP de `matrix`, puis `sudo systemctl restart postgresql`.
- `connection refused` : vérifier `listen_addresses`, service PostgreSQL et port 5432.



---

- Page precedente: [5.2 : Configuration de la VM Element](config-element-vm.md)
- Page suivante: [5.4 : Installation de l'application Element (VM element)](installation-app-element.md)
