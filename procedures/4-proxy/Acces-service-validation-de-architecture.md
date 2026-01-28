# 4.6 Acces aux services et validation de l'architecture

### Objectif

A ce stade du projet, l'architecture complete est en place :

* La VM `matrix` (`10.42.XX.1`) héberge :
  * Le serveur Synapse (port `8008`) ;
  * L'interface Element Web (port `8080`).
* La VM `rproxy` (`10.42.XX.2`) joue le rôle de reverse proxy et redirige les requêtes HTTP vers Synapse.
* Les deux machines sont situées dans un réseau privé non accessible directement depuis la machine physique.

L'objectif de cette procédure est donc de :
* Configurer les accès depuis la machine physique via des tunnels SSH ;
* Vérifier que toute la chaîne de communication fonctionne correctement de bout en bout.

---

### Schéma d'architecture (Validation des Flux)

```text
                                     RÉSEAU PRIVÉ (10.42.XX.0/24)
                                    |
[ PC PHYSIQUE ] --(Port 8080)-------|------> [ VM MATRIX (10.42.XX.1) ]
      |                             |               |-- Element Web (Port 8080)
      |                             |               |-- Synapse (Port 8008)
      |                             |                         ^
      |                             |                         | (Redirection Proxy)
      |                             |                         |
[ PC PHYSIQUE ] --(Port 9090)-------|------> [ VM RPROXY (10.42.XX.2) ]
                                    |               |-- Nginx Reverse Proxy (Port 80)
```

---

## 4.6.1 Configuration des tunnels SSH (redirections de ports)

### Principe général

Les machines virtuelles utilisent des adresses privées (`10.42.0.0/16`) qui ne sont pas routables depuis le réseau de la salle de TP. Pour accéder aux services, nous utilisons des tunnels SSH.

Afin de permettre l'évaluation et l'accès depuis d'autres postes du réseau (ex: le poste de l'enseignant), nous configurons ces tunnels pour qu'ils écoutent sur toutes les interfaces réseau de la machine physique (`0.0.0.0`), et non uniquement sur `localhost`.

### ⚠️ Note sur la Sécurité
Précisez qu'une fois le tunnel configuré en **0.0.0.0**, n'importe qui sur le réseau de l'IUT peut accéder à votre service via l'IP de votre machine physique. Si la confidentialité est requise, il est préférable de restreindre à **127.0.0.1** dans la configuration.

### Configuration via le fichier SSH (Recommandé)

Dans le fichier `~/.ssh/config` de la machine physique, nous ajoutons les directives `LocalForward` suivantes :

```text
Host vm
    HostName 10.42.XX.1
    User user
    ProxyJump virt
    # Tunnel pour Element Web (Port 8080) - Accessible par tout le réseau
    LocalForward 0.0.0.0:8080 localhost:8080
    # Tunnel pour le Proxy Synapse (Port 9090) - Accessible par tout le réseau
    LocalForward 0.0.0.0:9090 10.42.XX.2:80
```

### Commandes manuelles équivalentes

Si les tunnels doivent être ouverts manuellement dans un terminal, voici les commandes à utiliser :

#### 1. Accès à l'interface Element Web

```bash
ssh -L 0.0.0.0:8080:localhost:8080 user@10.42.XX.1
```
#### 2. Accès à Synapse via le Reverse Proxy

```bash
ssh -L 0.0.0.0:9090:10.42.XX.2:80 user@10.42.XX.1
```


### Acces a l'interface Element Web

Commande a executer depuis la machine physique :

```bash
ssh -L 8080:localhost:8080 user@10.42.XX.1
```

Explication :

* `8080:localhost:8080` :
  * le port 8080 du Mac/PC est relie au port 8080 de la VM `matrix`,
* `user@10.42.XX.1` :
  * connexion SSH vers la VM hebergeant Element Web.

Une fois connecte, l'interface Element Web est accessible a l'adresse :

```text
http://localhost:8080
```

### Acces a Synapse via le reverse proxy

Commande a executer depuis la machine physique (dans un autre terminal) :

```bash
ssh -L 9090:localhost:80 user@10.42.XX.2
```

Explication :

* `9090:localhost:80` :
  * le port 9090 de la machine physique est relie au port 80 de la VM `rproxy`,
* `user@10.42.XX.2` :
  * connexion SSH vers la machine reverse proxy.

Ce tunnel permet d'acceder a Synapse uniquement via le proxy, comme dans une architecture de production.

---

## 4.6.2 Validation de la chaine complete (test fonctionnel)

Pour verifier que le reverse proxy redirige correctement les requetes vers Synapse, ouvrez un navigateur sur la machine physique et accedez a l'URL suivante :

```text
http://localhost:9090/_matrix/client/versions
```

### Resultat attendu

Un document JSON s'affiche, listant les versions du protocole Matrix supportees par le serveur.

Exemple (simplifie) :

```json
{
  "versions": ["r0.6.1", "v1.1", "v1.2", "v1.3", "..."],
  "unstable_features": { ... }
}
```

### Interpretation

Ce resultat confirme que la chaine suivante fonctionne correctement :

```
Machine physique (localhost:9090)
        ↓
VM rproxy (port 80)
        ↓
VM matrix (Synapse – port 8008)
```

Le reverse proxy joue donc correctement son role d'aiguilleur et protege l'acces direct au serveur d'application.

---

## Section dédiée aux Tests de validation

### Test 2 : Connexion via Element Web
```text
Accéder à http://localhost:8080.

Cliquer sur "Modifier" le serveur d'accueil.

Entrer l'adresse du proxy : http://localhost:9090.

Se connecter avec le compte user crée juste avant.
```
Si la connexion réussit, l'architecture est totalement opérationnelle.

---

## 4.6.3 Depannage – erreurs courantes et solutions

Le tableau ci-dessous synthetise les problemes rencontres durant la mise en place de l'architecture et les solutions appliquees.

| Probleme rencontre | Cause probable | Solution appliquee |
| --- | --- | --- |
| Erreur X11 / Display | Le deport graphique ne fonctionne pas via SSH ou `vmiut console`. | Utiliser `vmiut info <vm>` pour recuperer l'IP et se connecter en SSH texte. |
| `user is not in the sudoers file` | L'utilisateur standard n'a pas les droits administrateur. | Passer en `root` avec `su -`, installer `sudo` et ajouter l'utilisateur avec `usermod -aG sudo user`. |
| Page "Welcome to Nginx" persistante | Le site par defaut est toujours actif sur le port 80. | Supprimer le lien symbolique : `sudo rm /etc/nginx/sites-enabled/default`. |
| Site inaccessible (404 / connexion refusee) | La configuration existe mais n'est pas activee. | Verifier `sites-enabled` et recharger Nginx avec `sudo systemctl reload nginx`. |
| Erreur de syntaxe Nginx | Faute de frappe dans un fichier de configuration. | Toujours executer `sudo nginx -t` avant rechargement. |
| Acces impossible depuis la machine physique | Tunnel SSH incorrect (mauvaise IP ou mauvais port). | Verifier les IP statiques : `10.42.XX.1` (matrix) et `10.42.XX.2` (rproxy). |

---

## Conclusion de validation

A l'issue de cette procedure :

* Element Web est accessible depuis la machine physique ;
* Synapse n'est jamais expose directement ;
* toutes les communications passent par le reverse proxy ;
* l'architecture respecte les bonnes pratiques de separation des roles.

Cette validation confirme que l'infrastructure deployee est fonctionnelle, coherente et conforme aux objectifs de la SAE.

<hr>

page precedente : 
[4.5 : Installation et configuration d'Element Web sur la VM matrix](install-element-web.md)

Revenir au menu : [Sommaire: Architecture final](../5-architecture-final/README.md)
