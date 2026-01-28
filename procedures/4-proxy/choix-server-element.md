## 4.1 Analyse et justification des choix technologiques

### Objectif
Cette section détaille l'analyse comparative et la justification de nos choix technologiques pour la semaine 4. L'objectif est de sélectionner les outils les plus adaptés pour servir le client Element Web et protéger le serveur Synapse.

---

### Schéma d'architecture logique (Cible Semaine 4)

```text
[ UTILISATEUR ] 
       |
       | (Port 80 / 443)
       v
[ VM RPROXY (10.42.XX.2) ] <--- Choix : NGINX (Reverse Proxy)
       |
       |------------------------------------------|
       | (Proxy Pass)                             | (Service Statique)
       v                                          v
[ VM MATRIX (10.42.XX.1) ]                 [ VM MATRIX (10.42.XX.1) ]
  Serveur Synapse (Port 8008)                Client Element Web (Port 8080)
                                             Choix : NGINX (Serveur Web)
```

---

## 4.1.1 Choix du serveur web pour Element Web

Element Web est un client Matrix qui fonctionne exclusivement en mode "web-front". Contrairement à Synapse, il s'agit d'une archive contenant des fichiers statiques (HTML, CSS, JavaScript).

### Évaluation des solutions

| Critères | Nginx | Apache |
| --- | --- | --- |
| **Gestion des ressources** | Architecture événementielle (très économe en RAM). | Architecture basée sur les processus (plus gourmande). |
| **Fichiers statiques** | Conçu spécifiquement pour servir du contenu statique à haute vitesse. | Très performant, mais avec un léger surplus de consommation (overhead). |
| **Configuration** | Syntaxe moderne, centralisée dans un seul fichier. | Utilisation de fichiers `.htaccess` qui complexifie la lecture globale. |
| **Courbe d'apprentissage** | Configuration intuitive et moins verbeuse. | Configuration très riche mais parfois obscure pour un débutant. |

### Justification du choix de l'équipe : Nginx
*   **Performance sur le contenu statique :** Nginx est techniquement l'outil le plus optimisé pour servir des fichiers sans exécution de code serveur.
*   **Légèreté :** Nos VM disposant de ressources limitées, la faible empreinte mémoire de Nginx est un avantage déterminant.
*   **Cohérence technique :** Utiliser Nginx pour le serveur web et le reverse proxy permet d'uniformiser nos compétences et la maintenance.

---

## 4.1.2 Choix du logiciel de reverse proxy

Pour la VM `rproxy`, nous avons besoin d'un "aiguilleur" capable de recevoir les requêtes sur le port 80 et de les transmettre à notre serveur Matrix.

### Tableau comparatif des reverse proxies

| Solution | Points forts | Points faibles |
| --- | --- | --- |
| **Nginx** | Polyvalent, configuration de proxy très directe (`proxy_pass`). | Moins d'options de statistiques avancées en version gratuite. |
| **Apache** | Très modulaire (nombreux modules `mod_proxy`). | Configuration verbeuse et plus complexe à sécuriser. |
| **HAProxy** | Spécialiste pur du proxy, performances extrêmes. | **Incapable de servir des fichiers statiques** (ne pourrait pas héberger Element). |

### Justification du choix : Nginx
*   **Polyvalence :** Nginx peut à la fois agir comme proxy pour Synapse et servir les fichiers d'Element Web.
*   **Simplicité :** La directive `proxy_pass` permet une redirection propre vers `10.42.XX.1:8008` en quelques lignes.
*   **Homogénéité :** Utiliser le même logiciel sur toutes les machines simplifie la gestion des fichiers de configuration.

---

## Section Tests de validation

Afin de valider ces choix lors de la mise en œuvre, nous effectuerons les tests suivants :

1.  **Validation de syntaxe :** Utilisation de `sudo nginx -t` après chaque modification pour éviter les erreurs de démarrage.
2.  **Vérification de l'empreinte mémoire :** Utilisation de la commande `top` ou `htop` pour comparer la consommation de Nginx par rapport au reste du système.
3.  **Test de réponse HTTP :** Utilisation de `curl -I localhost:8080` pour vérifier que Nginx renvoie bien les bons headers (Server: nginx).

---

## Section dédiée aux problèmes (Troubleshooting)

| Problème | Cause possible | Solution |
| :--- | :--- | :--- |
| **Conflit de port 80** | Le site `default` de Nginx occupe le port 80 à l'installation. | Supprimer le lien : `sudo rm /etc/nginx/sites-enabled/default`. |
| **Erreur 502 Bad Gateway** | Le Reverse Proxy ne parvient pas à joindre la VM Matrix. | Vérifier l'IP `10.42.XX.1` et s'assurer que Synapse est démarré. |
| **Fichiers Element Web non trouvés** | Mauvais chemin spécifié dans la directive `root`. | Vérifier que le chemin dans `nginx.conf` correspond au dossier d'extraction d'Element Web. |
| **Erreurs de permissions (403)** | L'utilisateur `www-data` n'a pas accès aux fichiers statiques. | Vérifier les droits avec `chmod -R 755` sur le dossier Element. |

---

- Page précédente: [Pré-requis et positionnement](README.md)
- Page suivante: [4.2 : Introduction et choix d'un reverse proxy](reverse-proxy-choix.md)