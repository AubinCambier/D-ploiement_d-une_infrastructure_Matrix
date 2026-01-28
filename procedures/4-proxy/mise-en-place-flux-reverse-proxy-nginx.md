## 4.4 Mise en place du flux reverse proxy dans Nginx

Une fois le serveur Nginx installe sur la machine virtuelle `rproxy`, il est necessaire de le configurer pour qu'il joue pleinement son role de reverse proxy.

Le principe est le suivant :

Toute requete HTTP recue par `rproxy` sur le port 80 doit etre redirigee vers le serveur Synapse installe sur la machine `matrix` (`10.42.XX.1`) sur le port `8008`.

Ainsi, le client ne communique jamais directement avec Synapse :
le reverse proxy agit comme point d'entree unique, ameliorant la securite et la maitrise des flux reseau.

---

### Schéma d'architecture (Flux du Reverse Proxy)

```text
[ Client / PC Physique ] 
       |
       | (Requete HTTP - Port 80)
       v
[ VM RPROXY (10.42.XX.2) ] 
       |
       |-- [ Nginx ] (Aiguilleur)
       |      |
       |      | (Redirection interne : proxy_pass)
       |      v
[ VM MATRIX (10.42.XX.1) ]
       |
       |-- [ Synapse (Port 8008) ]
```

---

## 4.4.1 Creation du fichier de configuration Nginx

Sous Debian, Nginx utilise une organisation standard :

* `/etc/nginx/sites-available/` : fichiers de configuration disponibles
* `/etc/nginx/sites-enabled/` : configurations reellement actives

Nous allons creer un fichier dedie au service Synapse afin d'avoir une configuration claire et isolee.

### Creation du fichier

Sur la VM `rproxy`, en tant qu'administrateur :

```bash
sudo nano /etc/nginx/sites-available/synapse
```

### Contenu du fichier de configuration

```nginx
server {
    listen 80;                      # Nginx ecoute les requetes HTTP sur le port 80
    server_name localhost;          # Nom logique du serveur (suffisant dans notre contexte)

    location / {                    # Toutes les requetes HTTP (racine /)
        proxy_pass http://10.42.XX.1:8008;  # Redirection vers la VM matrix (Synapse)

        # Transmission des informations reelles du client
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

### Explication pedagogique des directives principales

**`listen 80;`**
Definit le port d'ecoute du reverse proxy.
Le port 80 est le port HTTP standard utilise par les navigateurs.

**`proxy_pass`**
Directive centrale du reverse proxy.
Elle indique a Nginx ou transmettre les requetes recues.
Sans cette ligne, aucune redirection vers Synapse n'est possible.

**`proxy_set_header`**
Ces directives sont essentielles pour un service applicatif comme Synapse :

* elles permettent de transmettre l'IP reelle du client,
* elles evitent que Synapse pense que toutes les connexions proviennent du proxy,
* elles facilitent le debogage et la securite.

---

## 4.4.2 Activation de la configuration

Creer un fichier dans `sites-available` ne suffit pas :
Nginx ne charge que les fichiers presents dans `sites-enabled`.

### Activation du site Synapse

```bash
sudo ln -s /etc/nginx/sites-available/synapse /etc/nginx/sites-enabled/
```

Cette commande cree un lien symbolique, ce qui permet :

* d'activer le site,
* de conserver une configuration propre et modulaire.

### Desactivation du site par defaut

Par defaut, Nginx active un site nomme `default` qui ecoute deja sur le port 80.
Cela provoquerait un conflit de port avec notre reverse proxy.

```bash
sudo rm /etc/nginx/sites-enabled/default
```

Cette suppression garantit que seule notre configuration Synapse est utilisee.

---

## 4.4.3 Verification et rechargement du service

Avant toute mise en production, il est indispensable de verifier la syntaxe de la configuration.

### Verification de la configuration

```bash
sudo nginx -t
```

Resultat attendu :

```text
syntax is ok
test is successful
```

Cette etape permet d'eviter un arret du service du a une erreur de configuration.

### Application des changements

```bash
sudo systemctl reload nginx
```

Le rechargement :

* applique la nouvelle configuration,
* sans interrompre le service web.

---

## Section dédiée aux problèmes (Troubleshooting)

| Problème | Cause possible | Solution |
| :--- | :--- | :--- |
| **Erreur "502 Bad Gateway"** | Synapse est arrêté sur la VM matrix ou l'IP est mauvaise. | Vérifiez l'IP `10.42.XX.1` et assurez-vous que Synapse est actif (`systemctl status matrix-synapse`). |
| **`nginx -t` échoue** | Erreur de syntaxe (point-virgule manquant, accolade oubliée). | Lisez le message d'erreur de Nginx, il indique la ligne exacte à corriger dans le fichier `synapse`. |
| **"bind() to 0.0.0.0:80 failed"** | Le site `default` est encore actif ou un autre service utilise le port 80. | Supprimez bien le lien dans `sites-enabled/default` et redémarrez Nginx. |
| **La redirection ne se fait pas** | Le lien symbolique n'a pas été créé ou `reload` a été oublié. | Vérifiez la présence du fichier dans `sites-enabled` et relancez `sudo systemctl reload nginx`. |

---

## Section Tests de validation

Effectuez ces tests pour confirmer le succès de la configuration :

1.  **Test de syntaxe :** Tapez `sudo nginx -t`. (Résultat attendu : *successful*).
2.  **Test de connectivité interne :** Depuis la VM `rproxy`, tapez `curl -I http://10.42.XX.1:8008`.
    *Résultat attendu : Une réponse HTTP (200, 301 ou 404), prouvant que le proxy peut joindre la VM Matrix.*
3.  **Test du flux Reverse Proxy :** Depuis la VM `rproxy`, tapez `curl -I http://localhost`.
    *Résultat attendu : Une réponse provenant de Synapse (via Nginx).*
4.  **Vérification des processus :** Tapez `ps aux | grep nginx`.
    *Résultat attendu : Plusieurs processus Nginx doivent être en cours d'exécution.*

---

## Resultat attendu

A l'issue de cette procedure :

* la machine `rproxy` ecoute sur le port 80,
* toute requete HTTP recue est transmise a `matrix:8008`,
* Synapse n'est plus expose directement,
* l'architecture respecte le principe du reverse proxy en production.

<hr>

- Page précédente: [4.3 : Installation et configuration réseau de la VM rproxy](install-config-Vm-proxy.md)
- Page suivante: [4.5 : Installation et configuration d'Element Web sur la VM matrix](config-element-web-mattrix.md)