## 3.1 Installation et verification d'un service HTTP

### Objectif

Avant d'installer le serveur complexe (Synapse), nous devons nous assurer que la machine virtuelle (VM) est capable de fournir un service reseau de base. Nous allons installer le serveur web leger Nginx et verifier qu'il repond aux requetes locales.

---

### Schéma d'architecture (Tunnel SSH et flux HTTP)

```text
[ Poste Physique (Navigateur) ] 
       |
       | (Port Local 9090) <--- Tunnel SSH ---> (Port distant 80)
       v
[ Serveur Dattier (Passerelle) ]
       |
       v
[ VM Matrix (10.42.XX.1) ]
       |
       |-- [ Service Nginx ] (Écoute sur le port 80)
```

---

## Installation du serveur web

Connectez-vous a votre machine virtuelle via SSH.

```bash
ssh vm
```

Mettez a jour la liste des paquets et installez Nginx :

```bash
sudo apt update
sudo apt install nginx -y
```

---

## Verification du service

Une fois l'installation terminee, le service doit demarrer automatiquement. Verifiez son etat :

```bash
sudo systemctl status nginx
```

Resultat attendu : vous devez reperer la ligne indiquant `Active: active (running)`. Appuyez sur `q` pour quitter l'affichage du statut.

---

## Test de connexion locale (curl)

Pour tester le serveur depuis la VM elle-meme (qui n'a pas d'interface graphique), nous utilisons l'outil en ligne de commande `curl`.

Installez `curl` :

```bash
sudo apt install curl -y
```

Effectuez une requete HTTP sur l'interface locale :

```bash
curl http://localhost
```

Resultat attendu : le terminal doit afficher le code HTML brut de la page d'accueil de Nginx (`<!DOCTYPE html>... Welcome to nginx!`). Cela valide que le serveur ecoute bien sur le port 80.

![Welcome to nginx](../../ressources/3/Welcome%20to%20nginx.png)

---

## Analyse : pourquoi l'acces depuis le PC physique echoue ?

Si vous essayez d'acceder a `http://10.42.XX.1` depuis le navigateur de votre machine physique, cela ne fonctionnera pas.

Cause technique : la machine virtuelle est situee dans un reseau prive virtuel (`10.42.0.0/16`) gere par le serveur `dattier`. Ce reseau est masque (NAT) et n'est pas routable directement depuis le reseau physique de la salle de TP.

Solution : le tunnel SSH. Nous devons configurer une redirection de port (Port Forwarding) via SSH pour acceder au port 80 de la VM a travers une connexion securisee.

---

## Mise en place du tunnel SSH

Sur votre machine physique, modifiez votre fichier de configuration SSH pour ajouter une redirection de port.

```bash
nano ~/.ssh/config
```

Modifiez le bloc `Host vm` pour y ajouter la ligne `LocalForward`. Nous allons rediriger le port `9090` de votre machine physique vers le port 80 (web) de la VM.

```text
Host vm
    Hostname 10.42.XX.1
    User user
    ProxyJump virt
    LocalForward 9090 localhost:80
```

Sauvegardez et quittez.

---

## Validation

Fermez votre connexion SSH actuelle (`exit`) et reconnectez-vous pour activer le tunnel :

```bash
ssh vm
```

Sur votre machine physique, ouvrez un navigateur web (Firefox/Chrome) et allez a l'adresse :

```text
http://localhost:9090
```

Vous devriez voir la page "Welcome to nginx!".

---

## Section dédiée aux problèmes (Troubleshooting)

| Problème | Cause possible | Solution |
| :--- | :--- | :--- |
| **`curl` renvoie "Connection refused"** | Nginx n'est pas démarré ou n'écoute pas sur le port 80. | Vérifiez avec `sudo systemctl restart nginx`. |
| **`localhost:9090` ne répond pas dans le navigateur** | Le tunnel SSH n'est pas actif ou le fichier config est mal lu. | Vérifiez que vous avez bien relancé la session `ssh vm` après modification du fichier config. |
| **"bind: Address already in use" au lancement de SSH** | Le port 9090 est déjà utilisé par une autre application sur votre PC. | Changez le port local dans le fichier config (ex: `LocalForward 9191 localhost:80`). |
| **Le navigateur affiche "Welcome to nginx" mais sur Dattier** | Erreur dans le `LocalForward` (cible erronée). | Assurez-vous que la ligne est bien `9090 localhost:80` (localhost ici désigne la VM Matrix). |

---

## Section Tests de validation

Effectuez ces tests pour confirmer le bon fonctionnement de l'infrastructure réseau :

1.  **Vérification du service distant :** Sur la VM, tapez `ss -ltn | grep :80`.
    *Résultat attendu : Une ligne indiquant LISTEN sur le port 80.*
2.  **Vérification du tunnel local :** Sur votre machine physique (Linux/Mac/PowerShell), tapez `netstat -an | grep 9090` (ou `ss -ltn | grep 9090`).
    *Résultat attendu : Votre PC doit écouter sur le port 9090.*
3.  **Test d'accès Web :** Chargez `http://localhost:9090` dans votre navigateur.
    *Résultat attendu : Affichage de la page par défaut de Nginx.*

<hr>

- Page précédente: [Pré-requis et positionnement](../3-synapse/pré-requis.md)
- Page suivante: - [3.2 : Installation et configuration de Synapse](install-and-config-synapse.md)
