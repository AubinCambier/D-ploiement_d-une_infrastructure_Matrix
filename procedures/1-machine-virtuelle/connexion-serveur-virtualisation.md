
# 1.1 Connexion à distance au serveur de virtualisation (`dattier.iutinfo.fr`)

### Objectif

Accéder au serveur de virtualisation `dattier.iutinfo.fr` en SSH, en vérifiant l’empreinte de sa clé lors de la première connexion afin de s’assurer de l’identité du serveur.

---

### Schéma d'architecture

```text
CAS 1 : Depuis l'extérieur (Maison)
[ PC Personnel ] --( Internet + VPN )--> [ tp.iutinfo.fr ] --( Réseau Interne )--> [ dattier.iutinfo.fr ]

CAS 2 : Depuis l'IUT
[ Machine de TP ] --( Réseau Local XX )------------------------------------------> [ dattier.iutinfo.fr ]
```

---

### Contexte

Deux situations sont possibles :

* **Depuis une machine personnelle à domicile** : vous devez d’abord passer par le serveur d’accès `tp.iutinfo.fr`, puis vous connecter à `dattier`.
* **Depuis une machine de TP à l’IUT** : vous pouvez vous connecter directement à `dattier`.

Dans tous les cas, la vérification de l’empreinte SSH ne s’effectue qu’à la **première connexion** (ou si la clé change).

---

## A. Connexion depuis chez vous (PC -> `tp.iutinfo.fr` -> `dattier`)

### Étape A1 — Se connecter au serveur d’accès `tp.iutinfo.fr`

Depuis un terminal sur votre ordinateur 
Si vous avez deja installe le VPN de l'IUT, continuez. Sinon, consultez [la fiche VPN](https://infotuto.univ-lille.fr/fiche/vpn) pour l'installer et voir comment le configurer.
Exécutez :

```bash
ssh <login>.etu@tp.iutinfo.fr
```

Exemple (à adapter à votre identifiant) :

```bash
ssh prenom.nom.etu@tp.iutinfo.fr
```

Ensuite :

* saisissez votre mot de passe IUT (il ne s’affiche pas à l’écran, c’est normal) ;
* vous arrivez sur une machine du parc (par exemple `frene07`).

![hostname](../../ressources/1/connectionmaison.png)

Vous pouvez vérifier votre position avec :

```bash
hostname
```

---

### Étape A2 — Se connecter au serveur de virtualisation `dattier`

Depuis la machine du parc (ex : `frene07`), exécutez :

```bash
ssh dattier.iutinfo.fr
```

---

## B. Connexion depuis l’IUT (machine de TP -> `dattier`)

Depuis la machine de TP, exécutez directement :

```bash
ssh dattier.iutinfo.fr
```

---

# Première connexion : vérification de l’empreinte SSH

### Pourquoi cette vérification est-elle nécessaire ?

Lors d’une première connexion à un serveur SSH, votre client SSH ne connaît pas encore la clé publique du serveur. Il vous affiche donc une empreinte (fingerprint) et vous demande si vous souhaitez faire confiance à ce serveur. Cette étape permet d’éviter de se connecter à un serveur usurpé.

Vous verrez un message similaire à :

```text
The authenticity of host 'dattier.iutinfo.fr (172.18.48.20)' can't be established.
ED25519 key fingerprint is SHA256:QynRpdPucTVcwhMrD3824pqUviVFCgPwxwhkDyGyVSg.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

### Empreintes attendues pour `dattier.iutinfo.fr`

Comparez l’empreinte affichée à celles fournies par le sujet :

| Algorithme | Empreinte attendue (Référence XX)                    |
| ---------- | ---------------------------------------------------- |
| RSA        | `SHA256:mPCb5nJD8F6YOg/aDCRjqF/ZW3Ei9iLpzXw5UDCIH8g` |
| ECDSA      | `SHA256:+XNYpzmoYKDnwaB1xqCA2Yu7mBZEK5zvtfXYw1zDO1Y` |
| ED25519    | `SHA256:QynRpdPucTVcwhMrD3824pqUviVFCgPwxwhkDyGyVSg` |

En pratique, le message SSH affiche généralement l’empreinte **ED25519** : c’est celle qui doit correspondre exactement.

### Si l’empreinte correspond

Vous pouvez accepter la connexion en répondant :

```text
yes
```

Puis, si la connexion se poursuit, saisissez votre mot de passe IUT lorsque cela vous est demandé.

### Si l’empreinte ne correspond pas

Vous ne devez pas accepter. Répondez `no`, interrompez la connexion, et signalez le problème (risque de sécurité ou mauvaise machine contactée).

---

## Enregistrement dans `known_hosts` (preuve)

Après validation, le serveur est enregistré dans le fichier `~/.ssh/known_hosts` de la machine depuis laquelle vous avez lancé la commande `ssh`.

Vous pouvez le vérifier avec :

```bash
cat ~/.ssh/known_hosts
```

---

## Résultat attendu

Une fois connecté, votre invite de commande indique que vous êtes sur `dattier`, par exemple :

```text
<login>@dattier:~$
```

Vous pouvez confirmer en exécutant :

```bash
hostname
```

---

## (Optionnel) Obtenir les empreintes du serveur sans se connecter

Pour afficher les empreintes que le serveur publie (et les comparer au tableau), vous pouvez utiliser :

```bash
ssh-keyscan dattier.iutinfo.fr | ssh-keygen -lf -
```

Cette commande aide à justifier la comparaison des empreintes dans un rapport.

---

## Section dédiée aux problèmes

| Problème | Cause possible | Solution |
| :--- | :--- | :--- |
| **« Permission denied »** | Mot de passe incorrect, identifiant mal tapé ou mauvais compte. | Vérifiez votre login `.etu` et votre mot de passe IUT. Assurez-vous d'être sur la bonne machine d'origine. |
| **« WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED »** | La clé du serveur a changé (réinstallation ou attaque MITM). | Ne l'ignorez pas. Si le changement est légitime, nettoyez votre fichier avec `ssh-keygen -R dattier.iutinfo.fr`. |
| **« Connection timed out »** | Problème de réseau ou VPN non activé. | Vérifiez votre connexion internet et assurez-vous que le VPN est "Connected". |
| **« Host unreachable »** | Tentative de connexion directe à dattier depuis l'extérieur sans passer par la passerelle. | Relisez l'étape A1 : connectez-vous d'abord à `tp.iutinfo.fr`. |

---

## Tests de validation

Afin de vérifier que la procédure est réussie, effectuez les tests suivants :

1.  **Test de positionnement :** Tapez `hostname`. Le résultat doit être strictement `dattier`.
2.  **Test d'identité :** Tapez `whoami`. Le résultat doit être votre login IUT.
3.  **Vérification de l'IP :** Tapez `hostname -I`. L'adresse IP doit correspondre à celle du réseau de virtualisation (ex: `172.18.48.20`).
4.  **Vérification de la persistance :** Déconnectez-vous (`exit`) et reconnectez-vous. Le serveur ne doit plus vous demander de valider l'empreinte (fingerprint).

<hr>

- Page précédente: [Sommaire](README.md)
- Page suivante: [Gestion des clés de connexion SSH](gestion-des-clés-de-connexion-ssh.md)
