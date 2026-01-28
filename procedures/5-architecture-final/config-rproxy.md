# 5.6 Installation et Configuration du Reverse Proxy (VM `rproxy`) — Détails et explications

## Raccourci si vous êtes pressé

Si vous n’avez pas le temps de suivre la procédure manuelle et expliquée, vous pouvez aller **directement à l’étape 8** (“Script one-shot”) et exécuter le script qui configure tout automatiquement.

---

## 0) Convention “XX” (à respecter partout)

* Réseau virtuel IUT : **10.42.0.0/16** (routeur + DNS : **10.42.0.1**)
* Votre plage dédiée (vue sur Moodle) : **10.42.XX.0/24**
* IPs (convention) :

  * `matrix` : `10.42.XX.1` (Synapse sur `8008`)
  * `rproxy` : `10.42.XX.2` (reverse proxy)
  * `db` : `10.42.XX.3`
  * `element` : `10.42.XX.4` (serveur web sur `80`)

---

## 1) Contexte initial : ce qu’on fait au début

Au début de cette procédure, on part d’une VM `rproxy` existante (VM nue ou config Semaine 4 non adaptée).
L’objectif est de transformer cette VM en **aiguilleur** : elle ne stocke rien, ne calcule rien, elle **redirige**.

Pourquoi cette machine est critique :

* c’est la **seule** VM exposée (via le port 9090 côté accès “université”),
* elle protège `matrix`, `db`, `element` qui restent dans le réseau privé,
* si elle tombe, **plus d’accès** à Matrix et Element.

---

## 2) Schéma d’architecture logique (Host-based routing)

### Entrée

Le client accède à :

* `http://matrix.phys.iutinfo.fr:9090`
* `http://element.phys.iutinfo.fr:9090`

### Traitement

Nginx lit l’en-tête HTTP `Host`.

### Sortie

* si `Host = matrix...` → forward vers `10.42.XX.1:8008`
* si `Host = element...` → forward vers `10.42.XX.4:80`

Schéma :

```
Client -> (9090 via accès univ/tunnel) -> rproxy
                     |
                     +--> Host=matrix...  -> 10.42.XX.1:8008
                     |
                     +--> Host=element... -> 10.42.XX.4:80
```

---

## 3) Commandes manuelles (procédure “référence”, sans script)

> Toutes les commandes admin se font avec `sudo`.

### A. Configuration système

1. Définir le hostname :

```bash
sudo hostnamectl set-hostname rproxy
```

Ce que ça fait : change le nom interne de la machine.
Pourquoi : permet de reconnaître immédiatement la VM dans le prompt/logs.

---

### B. Configuration réseau (`/etc/network/interfaces`)

Objectif : fixer `rproxy` en **10.42.XX.2/16**.

1. (Optionnel) vérifier le nom de l’interface :

```bash
ip link
```

Pourquoi : confirmer que l’interface s’appelle bien `enp0s3`.

2. Éditer la configuration :

```bash
sudo nano /etc/network/interfaces
```

3. Mettre (ou remplacer) le bloc de `enp0s3` par :

```text
auto enp0s3
iface enp0s3 inet static
    address 10.42.XX.2/16
    gateway 10.42.0.1
```

Explications :

* `inet static` : IP fixe (indispensable pour un proxy),
* `address` : IP de la VM `rproxy`,
* `gateway` : routeur du réseau virtuel,
* `dns-nameservers` : DNS du routeur.

4. Appliquer la config :

```bash
sudo ifdown enp0s3 && sudo ifup enp0s3
```

Ce que ça fait : coupe puis relance l’interface.
Pourquoi : le système relit `/etc/network/interfaces` et applique la nouvelle IP.

---

### C. Configuration Nginx (`/etc/nginx/sites-available/sae-proxy`)

Objectif : router selon `Host` :

* `matrix.phys.iutinfo.fr` → `10.42.XX.1:8008`
* `element.phys.iutinfo.fr` → `10.42.XX.4:80`

#### 1) Installer et activer Nginx (si pas déjà fait)

```bash
sudo apt update
sudo apt install -y nginx
sudo systemctl enable --now nginx
```

Pourquoi : Nginx est le reverse proxy.

#### 2) Éditer la config du site

```bash
sudo nano /etc/nginx/sites-available/sae-proxy
```

Mettre la config suivante (remplacer `phys` par le nom de votre machine physique dans le DNS) :

```nginx
server {
    listen 9090;
    server_name element.phys.iutinfo.fr;

    location / {
        proxy_pass http://10.42.XX.4:80;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}

server {
    listen 9090;
    server_name matrix.phys.iutinfo.fr;

    location / {
        proxy_pass http://10.42.XX.1:8008;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        client_max_body_size 50M;
    }
}
```

Analyse des directives importantes :

* `listen 9090;` : port demandé par le sujet
* `server_name ...;` : sélection du bloc selon `Host`
* `proxy_pass ...;` : redirection vers la VM cible
* `X-Forwarded-For` : conserve l’IP du client côté logs

#### 3) Activer le site

```bash
sudo ln -sf /etc/nginx/sites-available/sae-proxy /etc/nginx/sites-enabled/sae-architecture
sudo rm -f /etc/nginx/sites-enabled/default
```

Pourquoi :

* Nginx lit `sites-enabled`,
* supprimer `default` évite la page “Welcome to nginx” et des collisions.

#### 4) Tester et recharger

```bash
sudo nginx -t
sudo systemctl reload nginx
```

Pourquoi :

* `nginx -t` évite d’appliquer une config cassée,
* `reload` applique sans couper brutalement.

---

## 5) Tests de validation

| Test                 | Commande                                                                                  | Résultat attendu                   |
| -------------------- | ----------------------------------------------------------------------------------------- | ---------------------------------- |
| Vérification IP      | `ip a`                                                                                    | `10.42.XX.2` apparaît sur `enp0s3` |
| Vérification écoute  | `sudo ss -lntp \| grep 9090`                                                              | Nginx écoute sur 9090              |
| Syntaxe Nginx        | `sudo nginx -t`                                                                           | `syntax is ok`                     |
| Routage vers Synapse | `curl -s -H "Host: matrix.phys.iutinfo.fr" http://localhost:9090/_matrix/client/versions` | JSON (versions)                    |
| Routage vers Element | `curl -I -H "Host: element.phys.iutinfo.fr" http://localhost:9090/`                       | `HTTP/1.1 200 OK`                  |

Notes :

* Le test Synapse vise un endpoint fiable : `/_matrix/client/versions`.
* Pour Element, une réponse HTTP cohérente est attendue (200 si OK ; éviter 502).

---

## 6) Problèmes courants (Troubleshooting)

| Symptôme                                       | Cause probable                                    | Diagnostic                                                                           | Solution                                                     |
| ---------------------------------------------- | ------------------------------------------------- | ------------------------------------------------------------------------------------ | ------------------------------------------------------------ |
| `502 Bad Gateway`                              | VM cible injoignable / service éteint             | `ping -c 1 10.42.XX.1` puis `curl -s http://10.42.XX.1:8008/_matrix/client/versions` | Démarrer `matrix`, vérifier Synapse (8008), corriger IP/port |
| Page “Welcome to nginx”                        | site default actif / mauvais match `server_name`  | `ls -l /etc/nginx/sites-enabled/`                                                    | supprimer `default`, activer `sae-architecture`, reload      |
| Mauvais routage                                | `server_name` incorrect                           | `sudo nginx -T \| grep server_name`                                                  | corriger les `server_name`, reload                           |
| `nginx -t` échoue                              | erreur de syntaxe                                 | lire la ligne d’erreur                                                               | corriger le fichier, retester                                |
| `curl` OK en local mais pas depuis l’extérieur | accès/tunnel 9090 pas actif (selon votre montage) | test depuis la machine physique sur l’URL attendue                                   | relancer le tunnel/forward, vérifier la config côté accès    |

---

## 7) Script final pour la procedure


Note importante : lorsqu'on doit changer de machine se scypte permet de reconfiger la vm rproxy pour qu'elle marche. Il reconfigure l'IP, le nom d'hote et les redirections Nginx automatiquement.


Fichier : `/usr/local/bin/migration_rproxy.sh`


```bash
#!/bin/bash
# Script de migration automatique RPROXY
# Ce script configure IP, Hostname et Nginx pour la nouvelle machine physique.
# Usage: sudo migration_rproxy.sh [NOUVEAU_NOM_MACHINE] [NOUVEAU_XX]


MACHINE=$1
XX=$2


# Verification des parametres
if [ -z "$MACHINE" ] || [ -z "$XX" ]; then
   echo "Erreur : Arguments manquants."
   echo "Usage : sudo migration_rproxy.sh <NOM_MACHINE> <XX>"
   echo "Exemple : sudo migration_rproxy.sh frene11 118"
   exit 1
fi


echo "--- Migration vers $MACHINE avec le reseau $XX ---"


# 1. Mise a jour de l'IP Systeme (/etc/network/interfaces)
echo "[1/3] Configuration de l'IP 10.42.$XX.2..."
# Utilisation de cat <<EOF pour ecrire tout le fichier d'un coup
cat <<EOF > /etc/network/interfaces
auto lo
iface lo inet loopback


auto enp0s3
iface enp0s3 inet static
   address 10.42.$XX.2/16
   gateway 10.42.0.1
   dns-nameservers 10.42.0.1
EOF


# Redemarrage reseau pour appliquer l'IP
ifdown enp0s3 --force > /dev/null 2>&1
ifup enp0s3 > /dev/null 2>&1


# 2. Mise a jour de la configuration Nginx
echo "[2/3] Reecriture de la configuration Nginx..."
CONFIG_FILE="/etc/nginx/sites-available/sae-proxy"


# Generation dynamique des blocs server avec les nouvelles variables
cat <<EOF > $CONFIG_FILE
server {
   listen 9090;
   server_name element.$MACHINE.iutinfo.fr;


   location / {
       proxy_pass http://10.42.$XX.4:80;
       proxy_set_header Host \$host;
       proxy_set_header X-Real-IP \$remote_addr;
       proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
   }
}


server {
   listen 9090;
   server_name matrix.$MACHINE.iutinfo.fr;


   location / {
       proxy_pass http://10.42.$XX.1:8008;
       proxy_set_header Host \$host;
       proxy_set_header X-Real-IP \$remote_addr;
       proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
       client_max_body_size 50M;
   }
}
EOF


# Creation du lien symbolique si inexistant
ln -sf $CONFIG_FILE /etc/nginx/sites-enabled/
# Suppression du defaut pour eviter les conflits
rm -f /etc/nginx/sites-enabled/default


# 3. Application
echo "[3/3] Redemarrage de Nginx..."
hostnamectl set-hostname "rproxy-$XX"
systemctl restart nginx


echo "Migration terminee avec succes !"
echo "Nouveaux liens :"
echo " - http://matrix.$MACHINE.iutinfo.fr:9090"
echo " - http://element.$MACHINE.iutinfo.fr:9090"
```




---

## 8) Script one-shot (refait toute la procédure automatiquement)

### Quand l’utiliser

* Si vous êtes pressé (raccourci annoncé au début)
* Si vous changez de machine et voulez reconfigurer vite

### Attention importante

Ce script **modifie le réseau** (`/etc/network/interfaces` + `ifdown/ifup`).
Si vous le lancez en SSH, vous risquez d’être déconnecté. Le plus propre est de le lancer depuis une console VM.

### Création du script

1. Créer le fichier :

```bash
sudo nano /usr/local/bin/setup_rproxy.sh
```

2. Coller le contenu :

```bash
#!/bin/bash
# Configuration complète de la VM rproxy (IP, hostname, Nginx reverse proxy)
# Usage : sudo setup_rproxy.sh <phys> <XX>
#   phys = nom de la machine physique (ex: frene01)
#   XX   = identifiant réseau (3e octet) (ex: 118)

set -eu

PHYS="${1:-}"
XX="${2:-}"

if [ -z "$PHYS" ] || [ -z "$XX" ]; then
  echo "Usage : sudo setup_rproxy.sh <phys> <XX>"
  echo "Exemple : sudo setup_rproxy.sh <NOM_MACHINE_PHYSIQUE> <XX>"
  exit 1
fi

echo "[1/6] Hostname"
hostnamectl set-hostname rproxy

echo "[2/6] Réseau statique (10.42.${XX}.2/16)"
# Réécriture du fichier réseau (équivalent à la saisie manuelle via nano, mais automatisé)
cat > /etc/network/interfaces <<EOF
auto lo
iface lo inet loopback

auto enp0s3
iface enp0s3 inet static
    address 10.42.${XX}.2/16
    gateway 10.42.0.1
    dns-nameservers 10.42.0.1
EOF

# Application réseau (peut couper l'accès)
ifdown enp0s3 --force >/dev/null 2>&1 || true
ifup enp0s3 >/dev/null 2>&1 || true

echo "[3/6] Installation Nginx"
apt update >/dev/null
apt install -y nginx >/dev/null
systemctl enable --now nginx >/dev/null

echo "[4/6] Configuration Nginx (Host-based routing)"
cat > /etc/nginx/sites-available/sae-proxy <<EOF
server {
    listen 9090;
    server_name element.${PHYS}.iutinfo.fr;

    location / {
        proxy_pass http://10.42.${XX}.4:80;

        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
    }
}

server {
    listen 9090;
    server_name matrix.${PHYS}.iutinfo.fr;

    location / {
        proxy_pass http://10.42.${XX}.1:8008;

        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;

        client_max_body_size 50M;
    }
}
EOF

echo "[5/6] Activation du site + désactivation default"
ln -sf /etc/nginx/sites-available/sae-proxy /etc/nginx/sites-enabled/sae-architecture
rm -f /etc/nginx/sites-enabled/default

echo "[6/6] Test et reload Nginx"
nginx -t >/dev/null
systemctl reload nginx >/dev/null

echo "OK. Liens attendus :"
echo " - http://matrix.${PHYS}.iutinfo.fr:9090"
echo " - http://element.${PHYS}.iutinfo.fr:9090"
```

3. Rendre exécutable :

```bash
sudo chmod +x /usr/local/bin/setup_rproxy.sh
```

### Exécution

```bash
sudo /usr/local/bin/setup_rproxy.sh <NOM_MACHINE_PHYSIQUE> <XX>
```

Après exécution, refaites les tests de la section 5.

---

- Page precedente: [5.5 : Installation et configuration de Synapse](installation-synapse.md)
- Page suivante: [5.7 : Utilisation finale et validation fonctionnelle](utilisation-final.md)
