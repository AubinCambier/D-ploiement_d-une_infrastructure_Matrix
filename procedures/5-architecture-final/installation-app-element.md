# 5.4 Installation de l’application Element (VM `element`)


- **`XX` représente le numéro de votre machine physique**, c’est-à-dire les chiffres dans son nom.
- Exemple : si votre machine physique est `frene118`, alors **`XX = 118`** et `matrix.XX.iutinfo.fr` devient `matrix.frene118.iutinfo.fr`.
- Même règle pour l’adressage interne : réseau **10.42.0.0/16** et IP du type **`10.42.XX.x/16`**.

---

## Objectif

Installer **Element Web** sur la VM `element` via **Nginx**, puis configurer Element pour qu’il se connecte à votre serveur Matrix **via le Reverse Proxy exposé sur la machine physique en `:9090`**.

---

## 1) Schéma d’architecture, flux et adressage

### Adressage (exemple à adapter, réseau 10.42.0.0/16)

- VM `matrix` (Synapse) : `10.42.XX.1/16` (Synapse sur `:8008`)
- VM `rproxy` (Reverse Proxy) : `10.42.XX.2/16` (reverse proxy sur `:80`)
- VM `postgresql` (base Synapse) : `10.42.XX.3/16` (PostgreSQL sur `:5432`)
- VM `element` (Nginx + Element statique) : `10.42.XX.4/16` (Nginx sur `:80`)

### Flux (ASCII clair)

```
(1) Navigateur
    |
    | HTTP (téléchargement des fichiers Element)
    v
VM element (10.42.XX.4:80)  ---- sert HTML/JS/CSS ----
    |
    | (2) Requêtes Matrix envoyées par Element (JS) vers base_url
    v
Machine physique :9090  ---- tunnel SSH ---->  VM rproxy (10.42.XX.2:80)
                                         |
                                         | reverse proxy HTTP
                                         v
                                   VM matrix (10.42.XX.1:8008) : Synapse
                                         |
                                         | SQL TCP
                                         v
                               VM postgresql (10.42.XX.3:5432)
```

---

## 2) Procédure pas à pas (avec explications)

> Dans cette procédure, les commandes qui modifient le système utilisent `sudo` (vous restez en utilisateur normal).

### A. Installer Nginx (serveur web)

1. Mettre à jour l’index des paquets :

```bash
sudo apt update
```

**Pourquoi :** récupère la liste des paquets disponibles et évite d’installer avec un index obsolète.

2. Installer Nginx :

```bash
sudo apt install -y nginx
```

**Pourquoi :** Nginx sert les fichiers statiques d’Element au navigateur. `-y` évite de confirmer à la main.

3. Démarrer Nginx et l’activer au démarrage :

```bash
sudo systemctl enable --now nginx
```

**Pourquoi :** le service est actif immédiatement et redémarrera automatiquement après reboot.

---

### B. Télécharger et déployer Element Web dans `/var/www/html/element`

1. Installer les outils nécessaires :

```bash
sudo apt install -y wget tar
```

**Pourquoi :** `wget` télécharge l’archive, `tar` l’extrait.

2. Télécharger une version officielle (exemple) :

```bash
cd /tmp
ELEMENT_VER="1.11.59"
wget "https://github.com/vector-im/element-web/releases/download/v${ELEMENT_VER}/element-v${ELEMENT_VER}.tar.gz"
```

**Pourquoi :** `/tmp` convient pour du temporaire, et la variable évite les erreurs de frappe.

3. Extraire l’archive :

```bash
tar -xzvf "element-v${ELEMENT_VER}.tar.gz"
```

**Pourquoi :** décompression + extraction des fichiers.

4. Préparer le répertoire final et y déplacer les fichiers :

```bash
sudo mkdir -p /var/www/html/element
sudo rm -rf /var/www/html/element/*
sudo mv "element-v${ELEMENT_VER}/"* /var/www/html/element/
```

**Pourquoi :**

- `mkdir -p` crée le dossier si besoin,
- `rm -rf .../*` évite les conflits avec une ancienne installation,
- `mv` place les fichiers exactement là où Nginx ira les chercher.

5. Appliquer des droits cohérents après déploiement :

```bash
sudo chown -R www-data:www-data /var/www/html/element
```

**Pourquoi :** Nginx (utilisateur `www-data`) doit pouvoir lire les fichiers.

---

### C. Configurer Nginx pour servir Element

1. Éditer le site :

```bash
sudo nano /etc/nginx/sites-available/element
```

**Pourquoi :** ce fichier définit comment Nginx sert votre application.

2. Mettre la configuration (celle que vous utilisez) :

```nginx
server {
    listen 80;
    listen [::]:80;

    root /var/www/html/element;
    index index.html;

    server_name _;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

**Pourquoi :**

- `root` pointe vers votre dossier Element,
- `index` sert la page d’accueil,
- `try_files` renvoie 404 si le fichier n’existe pas.

3. Activer le site, tester la config, recharger Nginx :

```bash
sudo rm -f /etc/nginx/sites-enabled/default
sudo ln -s /etc/nginx/sites-available/element /etc/nginx/sites-enabled/element

sudo nginx -t
sudo systemctl reload nginx
```

**Pourquoi :**

- on retire le site par défaut pour éviter des collisions,
- le lien symbolique active votre site,
- `nginx -t` vérifie la syntaxe avant application,
- `reload` applique sans couper brutalement le service.

---

### D. Configurer Element pour pointer vers votre Reverse Proxy (étape critique)

1. Se placer dans le dossier d’Element :

```bash
cd /var/www/html/element
```

**Pourquoi :** simplifie les commandes et évite les erreurs de chemin.

2. Créer `config.json` à partir du modèle officiel :

```bash
sudo cp config.sample.json config.json
```

**Pourquoi :** Element attend une structure JSON complète. Partir du fichier sample évite un `config.json` incomplet qui empêcherait Element de fonctionner.

3. Éditer le bon fichier :

```bash
sudo nano /var/www/html/element/config.json
```

**Pourquoi :** c’est ce fichier qui contient la configuration réellement lue par Element.

4. Modifier la section **exacte** `default_server_config` (remplacer `XX` par votre numéro de machine) :

Exemple :

- si votre machine physique est `frene118`, alors vous remplacez `XX` par `118` dans les IP `10.42.XX.x` et par `frene118` dans les DNS `matrix.XX.iutinfo.fr`.

À mettre dans `config.json` :

```json
"default_server_config": {
  "m.homeserver": {
    "base_url": "http://matrix.XX.iutinfo.fr:9090",
    "server_name": "matrix.XX.iutinfo.fr:9090"
  },
  "m.identity_server": {
    "base_url": "https://vector.im"
  }
},
```

**Pourquoi :**

- `base_url` = l’URL que le navigateur contactera pour parler à Matrix. Ici, c’est la **machine physique :9090** (tunnel SSH vers `rproxy`),
- vous ne pointez pas directement vers la VM `matrix` car elle n’est pas accessible depuis l’extérieur.

5. Appliquer des permissions finales :

```bash
sudo chown -R www-data:www-data /var/www/html/element
sudo chmod 644 /var/www/html/element/config.json
```

**Pourquoi :**

- `chown` garantit que Nginx peut lire tous les fichiers,
- `chmod 644` rend `config.json` lisible par le serveur web.

---

## 3) Tests de validation

### Sur la VM `element`

| Test                      | Commande                                  | Résultat attendu         |
| ------------------------- | ----------------------------------------- | ------------------------ |
| Nginx actif               | `sudo systemctl status nginx --no-pager`  | `active (running)`       |
| Nginx écoute bien         | `sudo ss -lntp \| grep ':80'`            | une ligne LISTEN sur :80 |
| Fichier principal présent | `ls -l /var/www/html/element/index.html`  | fichier existant         |
| Config présente           | `ls -l /var/www/html/element/config.json` | fichier existant         |
| Test HTTP local           | `curl -I http://localhost/`               | `HTTP/1.1 200 OK`        |

### Depuis la machine physique / navigateur

- Ouvrir : `http://10.42.XX.4/` (adresse de la VM `element`)
  **Résultat attendu :** interface Element s’affiche.

### Test du homeserver via le tunnel reverse proxy

Depuis la machine physique :

- Ouvrir : `http://matrix.XX.iutinfo.fr:9090/_matrix/client/versions`
  **Résultat attendu :** réponse JSON (pas de “connection refused”).

---

## 4) Probleme courant

### A — Erreur Nginx (site ne charge pas / 404)

- Diagnostic :

```bash
sudo nginx -t
sudo tail -n 50 /var/log/nginx/error.log
```

- Solutions :

  - vérifier que `root` pointe bien vers `/var/www/html/element`
  - vérifier que `index.html` existe
  - recharger : `sudo systemctl reload nginx`

### B — Element ne se connecte pas au serveur (homeserver unreachable)

- Diagnostic :

  - vérifier dans `config.json` que `base_url` = `http://matrix.XX.iutinfo.fr:9090`
  - tester depuis la machine physique : `http://matrix.XX.iutinfo.fr:9090/_matrix/client/versions`

- Causes fréquentes :

  - tunnel SSH inactif,
  - mauvais `XX`,
  - mauvais port.

- Solution :

  - relancer tunnel,
  - corriger `config.json`,
  - recharger fort le navigateur (Ctrl+F5) ou navigation privée.

### C — Les modifications de `config.json` ne sont pas prises en compte

- Cause : cache navigateur.
- Solution : Ctrl+F5 / navigation privée.

---

- Page precedente: [5.3 : Mise en place de PostgreSQL (VM db)](config-postgresql-vm.md)
- Page suivante: [5.5 : Installation et configuration de Synapse](installation-synapse.md)
