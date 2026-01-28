
## 2.2 Installation et configuration de sudo

### Objectif

Par defaut, seul le compte `root` peut effectuer des actions d'administration (installation de logiciels, modification du reseau, etc.). Utiliser ce compte en permanence est dangereux (risque d'erreur fatale).

Nous allons donc installer l'outil `sudo` (SuperUser DO) et autoriser notre utilisateur standard (`user`) a l'utiliser. Cela permet d'executer des commandes "en tant qu'administrateur" ponctuellement, en utilisant son propre mot de passe.

---

### Schéma logique de privilèges

```text
[ Utilisateur standard (user) ]
      |
      |-- Commande : sudo <action>
      |
      v
[ Vérification : Membre du groupe "sudo" ? ]
      |
      |-- (OUI) -> Demande mot de passe de 'user' -> [ Exécution en tant que ROOT ]
      |-- (NON) -> Message d'erreur : "user is not in the sudoers file"
```

---

### Etape 1 : Passage en root et installation

Comme `sudo` n'est pas encore installe, nous devons obligatoirement passer par le compte `root` une derniere fois pour l'installer.

Connectez-vous a la machine virtuelle.

Passez en `root` :

```bash
su -
```

Saisissez le mot de passe `root`.

Mettez a jour la liste des paquets et installez `sudo` :

```bash
apt update && apt install sudo -y
```

![install et update sudo](../../ressources/2/install%20et%20update%20sudo.png)

---

### Etape 2 : Ajout de l'utilisateur au groupe sudo

La configuration de `sudo` sur Debian repose sur les groupes. Le fichier de configuration `/etc/sudoers` indique que tous les membres du groupe `sudo` ont les droits d'administration.

Au lieu de modifier le fichier de configuration (ce qui est risque), il suffit d'ajouter l'utilisateur `user` a ce groupe.

Executez la commande suivante :

```bash
adduser user sudo
```

Le systeme doit confirmer l'ajout : ajout de l'utilisateur "user" au groupe "sudo". Ici ce n'est pas le cas car c'etait deja le cas.

![user sudo](../../ressources/2/user%20deja%20sudo.png)

---

### Etape 3 : Prise en compte des droits (point critique)

Attention : une modification de groupe n'est jamais prise en compte dans la session en cours. Si vous essayez d'utiliser `sudo` tout de suite, cela ne fonctionnera pas.

Vous devez obligatoirement fermer votre session.

Quittez le compte `root` :

```bash
exit
```

Quittez la session utilisateur (deconnexion SSH) :

```bash
exit
```

---

### Etape 4 : Verification

Reconnectez-vous a la machine virtuelle en SSH :

```bash
ssh vm
```

Testez vos nouveaux droits en lancant une commande administrative precedee de `sudo`. Par exemple, mettre a jour la liste des paquets :

```bash
sudo apt update
```

Le systeme va vous demander : `[sudo] Mot de passe de user :`. Saisissez le mot de passe de `user` (et non celui de `root`).

![user sudo apt update](../../ressources/2/sudo%20apt%20update.png)

---

## Section dédiée aux problèmes (Troubleshooting)

| Problème | Cause possible | Solution |
| :--- | :--- | :--- |
| **"user is not in the sudoers file"** | L'ajout au groupe a échoué ou la session n'a pas été redémarrée. | Vérifiez l'appartenance avec `groups`. Reconnectez-vous impérativement (Etape 3). |
| **"sudo: command not found"** | Le paquet `sudo` n'a pas été installé correctement. | Repassez en root (`su -`) et relancez `apt install sudo`. |
| **Mot de passe refusé** | Tentative d'utiliser le mot de passe root au lieu de celui de `user`. | Pour `sudo`, c'est toujours votre propre mot de passe (user) qui est demandé. |
| **Erreur de verrou (lock) apt** | Une autre instance d'apt (mise à jour auto) tourne en fond. | Attendez 1 à 2 minutes ou redémarrez la VM. |

---

## Section Tests de validation

Réalisez ces tests pour confirmer que votre utilisateur est bien un administrateur délégué :

1.  **Vérification des groupes :** Tapez `groups`.
    *Résultat attendu : La liste doit contenir le mot `sudo`.*
2.  **Test d'identité à privilèges :** Tapez `sudo whoami`.
    *Résultat attendu : Après saisie de votre mot de passe, le terminal doit répondre `root`.*
3.  **Test de lecture système :** Tapez `sudo ls /root`.
    *Résultat attendu : Le contenu du répertoire root doit s'afficher (accès normalement interdit à l'utilisateur standard).*

<hr>

- Page précédente: [Changement du nom de machine (hostname)](./change-hostname.md)
- Page suivante: [Installer PostgreSQL pour la gestion des utilisateurs/identifiants](install-PostgreSQL.md)