
## 3.2 Installation et configuration de Synapse

### Objectif

Installer le serveur de messagerie Matrix (implementation Synapse) et le configurer pour qu'il fonctionne dans notre reseau prive, en utilisant la base de donnees PostgreSQL preparee en semaine 2.

---

### Schéma d'architecture (Flux de Synapse)

```text
[ Client (Navigateur/Element) ]
       |
       | (Port 8008 - HTTP)
       v
[ Serveur Synapse (VM Matrix) ]
       |
       | (Port 5432 - TCP/IP)
       v
[ Base de données PostgreSQL ]
```

---

## 1. Installation du paquet

Nous utilisons le depot officiel de Matrix.org pour avoir une version recente.

Connectez-vous a la VM (`ssh vm`) et executez :

```bash
sudo apt install -y lsb-release wget apt-transport-https
sudo wget -O /usr/share/keyrings/matrix-org-archive-keyring.gpg https://packages.matrix.org/debian/matrix-org-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/matrix-org-archive-keyring.gpg] https://packages.matrix.org/debian/ $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/matrix-org.list
sudo apt update
sudo apt install matrix-synapse-py3 yamllint -y
```

Configuration importante durant l'installation : le systeme vous demandera un "Server Name". Vous devez entrer le nom de votre machine physique suivi du port `8008`.

Format : `machine-physique.iutinfo.fr:8008`

Exemple : `freneXX.iutinfo.fr:8008`

![serveur name](../../ressources/3/serveur%20name.png)

---

## 2. Configuration de Synapse (`homeserver.yaml`)

Le fichier de configuration se trouve dans `/etc/matrix-synapse/homeserver.yaml`. Nous devons modifier plusieurs sections. Ouvrez le fichier :

```bash
sudo nano /etc/matrix-synapse/homeserver.yaml
```

### A. Autoriser l'ecoute reseau

Par defaut, Synapse n'ecoute que sur `127.0.0.1`. Pour y acceder via le tunnel SSH, il doit ecouter sur toutes les interfaces. Cherchez la section `listeners` et modifiez `bind_addresses` :

```yaml
bind_addresses:
  - '0.0.0.0'
```

### B. Desactiver la federation (reseau prive)

Comme la VM n'a pas acces a Internet, elle ne peut pas verifier les cles sur les serveurs publics. Si on ne desactive pas ca, le serveur sera tres lent ou plantera. Ajoutez cette ligne a la fin du fichier :

```yaml
trusted_key_servers: []
```

### C. Connecter la base PostgreSQL

Par defaut, Synapse utilise SQLite. Nous devons le forcer a utiliser PostgreSQL (installe en semaine 2). Commentez (ajoutez `#`) la section `database: name: sqlite3...` et ajoutez ceci :

```yaml
database:
  name: psycopg2
  args:
    user: "matrix"
    password: "matrix"
    database: "matrix"
    host: "localhost"
    cp_min: 5
    cp_max: 10
```

### D. Activer les inscriptions

Pour pouvoir creer des utilisateurs :

```yaml
enable_registration: true
enable_registration_without_verification: true
```

Note : la ligne `enable_registration_without_verification` est indispensable car nous n'avons pas configure de serveur mail (SMTP) pour valider les comptes.

### E. Configuration du secret partage (Registration Shared Secret)

Cette cle est necessaire pour creer des utilisateurs administrateurs via la ligne de commande. Cherchez ou ajoutez la ligne :

```yaml
registration_shared_secret: "un_mot_de_passe_complexe_ici"
```

Sauvegardez et quittez.

---

## 3. Verification et demarrage

Avant de demarrer, verifiez toujours la syntaxe YAML (une erreur d'espace empeche le demarrage) :

```bash
yamllint /etc/matrix-synapse/homeserver.yaml
```

Si tout est correct, redemarrez le service :

```bash
sudo systemctl restart matrix-synapse
```

Verifiez l'etat :

```bash
sudo systemctl status matrix-synapse
```

Vous devez voir `Active: active (running)`.

![serveur actif](../../ressources/3/service%20active.png)

---

## 4. Creation du premier utilisateur (admin)

Utilisez la commande fournie par Synapse pour creer votre compte :

```bash
register_new_matrix_user -c /etc/matrix-synapse/homeserver.yaml http://localhost:8008
```

* User : `admin` (ou votre prenom)
* Password : (votre choix)
* Make admin : `yes`

---

## Section Tests de validation

Effectuez ces tests pour confirmer le bon fonctionnement du service :

1.  **Vérification de l'écoute port :** Tapez `ss -ltn | grep 8008`.
    *Résultat attendu : Une ligne LISTEN sur 0.0.0.0:8008.*
2.  **Test de réponse HTTP locale :** Tapez `curl http://localhost:8008`.
    *Résultat attendu : Une réponse JSON ou HTML indiquant "It works! Matrix is running".*
3.  **Vérification des logs en direct :** Tapez `sudo tail -f /var/log/matrix-synapse/homeserver.log`.
    *Résultat attendu : Pas d'erreurs de type "DatabaseError" ou "ConnectionRefused".*

---

## Section dédiée aux problèmes (Troubleshooting)

| Problème | Cause possible | Solution |
| :--- | :--- | :--- |
| **"No 'registration_shared_secret' defined"** | Clé absente du fichier YAML. | Ajoutez `registration_shared_secret: "..."` dans `homeserver.yaml` et redémarrez. |
| **Service en échec (failed) au démarrage** | Synapse refuse les inscriptions sans vérification par défaut. | Ajoutez `enable_registration_without_verification: true` dans la config. |
| **"DatabaseError: collation mismatch"** | PostgreSQL n'est pas en locale "C". | Supprimez et recréez la DB : `sudo -u postgres createdb -O matrix matrix --encoding=UTF8 --locale=C -T template0`. |
| **"bind_addresses" error** | Syntaxe YAML incorrecte (espacement). | Vérifiez avec `yamllint`. L'indentation doit être de 2 espaces exactement. |
| **Connexion DB échouée** | L'utilisateur matrix n'a pas les droits ou mauvais MDP. | Vérifiez la section `database` dans le YAML et re-testez la connexion manuelle avec `psql`. |

<hr>

Page precedente : [3.1 : Installation et vérification d’un service HTTP](installation-verif-service-http.md)

Page suivante : [3.3 : Accès via Element Web et tunnel SSH](acces-element.md)