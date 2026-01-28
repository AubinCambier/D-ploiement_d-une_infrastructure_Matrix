# SAÉ 3.03 : Administration Système et Réseaux
## Déploiement d'une infrastructure de communication Matrix

**IUT A - Université de Lille** | **Département Informatique** | **2024-2025**

---

###  Membres du binôme
* **Aubin CAMBIER** ([aubin.cambier.etu@univ-lille.fr](mailto:aubin.cambier.etu@univ-lille.fr))
* **Ilyes MOBAREK** ([ilyes.mobarek.etu@univ-lille.fr](mailto:ilyes.mobarek.etu@univ-lille.fr))

---

###  Présentation du projet
Ce dépôt documente la conception et la mise en œuvre technique d'un service de messagerie instantanée décentralisé basé sur le protocole **Matrix**. 

L'objectif est de construire une infrastructure robuste et sécurisée en environnement virtualisé, intégrant :
* Le serveur de référence **Synapse** (Python/Twisted).
* Une base de données relationnelle **PostgreSQL**.
* Un **Reverse Proxy Nginx** pour la segmentation et la sécurité des flux.
* Le client lourd web **Element Web**.

---

###  Stack Technique
* **Système :** Debian 12 (Bookworm)
* **Virtualisation :** VirtualBox / `vmiut` (Serveur `dattier`)
* **Base de données :** PostgreSQL 15+
* **Serveur Web & Proxy :** Nginx
* **Messagerie :** Matrix Synapse
* **Réseau :** SSH Tunneling, ProxyJump, IPv4 Statique (`10.42.XX.0/24`)

---

###  Schéma d'Architecture Global
L'infrastructure finale repose sur une séparation des services entre deux machines virtuelles pour isoler le serveur applicatif des accès directs.

```text
       [ RÉSEAU PHYSIQUE IUT / MAISON ]
                    |
                    v (Tunnel SSH / ProxyJump)
       +---------------------------------------+
       |       Serveur Hôte : DATTIER          |
       +---------------------------------------+
                    |
       +------------+------------+
       |                         |
       v                         v
[ VM RPROXY : 10.42.XX.2 ]   [ VM MATRIX : 10.42.XX.1 ]
|  (Reverse Proxy Nginx) |   |  (Serveur Synapse)     |
|  (Port 80 -> Matrix)   |   |  (Base PostgreSQL)     |
+------------------------+   |  (Element Web : 8080)  |
                             +------------------------+
```

---

##  Organisation de la documentation

### [Partie 1 : Mise en place de l'environnement virtuel](procedures/1-machine-virtuelle/README.md)
*Initialisation et accès sécurisé à l'infrastructure.*
* **1.1** : [Connexion à distance](procedures/1-machine-virtuelle/connexion-serveur-virtualisation.md)
* **1.2** : [Authentification par clé SSH](procedures/1-machine-virtuelle/gestion-des-clés-de-connexion-ssh.md)
* **1.3** : [Gestion des VMs via `vmiut`](procedures/1-machine-virtuelle/creer-machine-virtuelle.md)
* **1.4** : [Accès Console et Première Sécurisation](procedures/1-machine-virtuelle/AccesVm.md)
* **1.5** : [Réseau Statique (10.42.XX.1)](procedures/1-machine-virtuelle/config-reseau.md)
* **1.6** : [Optimisation : Alias SSH & ProxyJump](procedures/1-machine-virtuelle/ssh-alias-connexion.md)

### [Partie 2 : Configuration des services pré-requis](procedures/2-configuration-service/README.md)
*Préparation du système et du stockage des données.*
* **2.1** : [Identité de la machine (Hostname)](procedures/2-configuration-service/config-hostname.md)
* **2.2** : [Délégation de privilèges (Sudo)](procedures/2-configuration-service/config-sudo.md)
* **2.3** : [SGBD PostgreSQL pour Matrix](procedures/2-configuration-service/install-PostgreSQL.md)

### [Partie 3 : Installation et configuration de Synapse](procedures/3-synapse/README.md)
*Déploiement du cœur de messagerie.*
* **3.1** : [Validation HTTP via Nginx](procedures/3-synapse/installation-verif-service-http.md)
* **3.2** : [Configuration avancée de Synapse](procedures/3-synapse/install-and-config-synapse.md)
* **3.3** : [Exposition via Tunnel SSH local](procedures/3-synapse/acces-element.md)
* **3.4** : [Automatisation du changement de salle (Scripting)](procedures/3-synapse/changement-machine.md)

### [Partie 4 : Professionnalisation & Reverse Proxy](procedures/4-proxy/README.md)
*Segmentation réseau et déploiement du client web.*
* **4.1 - 4.2** : [Analyses et choix technologiques](procedures/4-proxy/choix-server-element.md)
* **4.3** : [Déploiement de la VM `rproxy`](procedures/4-proxy/install-config-Vm-proxy.md)
* **4.4** : [Configuration du Reverse Proxy Nginx](procedures/4-proxy/mise-en-place-flux-reverse-proxy-nginx.md)
* **4.5** : [Hébergement d'Element Web sur `matrix`](procedures/4-proxy/config-element-web-mattrix.md)
* **4.6** : [Validation finale de l'architecture](procedures/4-proxy/Acces-service-validation-de-architecture.md)

---

## Validation et Tests
Chaque procédure inclut une section de **Tests de validation** permettant de vérifier le bon fonctionnement du service (tests `curl`, `ping`, vérification de ports `ss`, etc.).

Une section **Dépannage (Problèmes rencontrés)** est présente dans chaque fiche pour documenter les erreurs fréquentes (erreurs de syntaxe YAML, permissions PostgreSQL, problèmes de tunnels SSH).

---

##  Commencer ici
> [!TIP]
> Pour débuter le projet, veuillez consulter la **[Partie 1 : Mise en place de l'environnement virtuel](procedures/1-machine-virtuelle/README.md)**.
