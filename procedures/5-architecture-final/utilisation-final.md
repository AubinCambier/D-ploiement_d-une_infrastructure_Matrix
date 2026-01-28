## 5.7 Utilisation finale et validation fonctionnelle

### Objectif

Valider que l’architecture distribuée fonctionne **de bout en bout** en réalisant un scénario utilisateur complet : accès web, création de compte, création de salon, envoi de message.
Cette étape prouve que **les 4 VMs** (`element`, `rproxy`, `matrix`, `db`) communiquent correctement.

---

## 0) Convention “XX” (

* Réseau IUT : **10.42.0.0/16** (routeur/DNS : **10.42.0.1**)
* Votre plage dédiée : **10.42.XX.0/24**
* Adresses :

  * `matrix` : `10.42.XX.1` (Synapse `8008`)
  * `rproxy` : `10.42.XX.2` (Nginx reverse proxy `80`)
  * `db` : `10.42.XX.3` (PostgreSQL `5432`)
  * `element` : `10.42.XX.4` (Nginx + Element `80`)
* Accès “université” : via **`:9090`** côté machine physique, redirigé vers `rproxy:80` (tunnel/forward)

---

## 1) Schéma des flux utilisateur

### Flux principal (depuis le réseau de l’université)

```
[Navigateur - machine physique]
        |
        | URL: http://element.phys.iutinfo.fr:9090
        | (Host = element.phys.iutinfo.fr)
        v
[Accès :9090 sur machine physique]
        |
        | Tunnel/forward -> 10.42.XX.2:80
        v
[VM rproxy 10.42.XX.2:80]
   |                         \
   | Host=element...          \ Host=matrix...
   v                           v
[VM element 10.42.XX.4:80]   [VM matrix 10.42.XX.1:8008]
                                   |
                                   | TCP 5432
                                   v
                             [VM db 10.42.XX.3:5432]
```

---

## 2) Pré-requis (avant de tester)

Vérifier que :

1. Les 4 VMs sont démarrées.
2. Les IP sont correctes (dans `10.42.XX.0/24`) et en `/16` sur l’interface.
3. La résolution locale / nommage est cohérente :

   * sur chaque VM, `/etc/hosts` permet de résoudre `matrix`, `db`, `element`, `rproxy`.
4. `rproxy` route correctement :

   * Host `element.phys...` → `10.42.XX.4:80`
   * Host `matrix.phys...` → `10.42.XX.1:8008`
5. `element` a un `config.json` qui pointe vers `matrix.phys.iutinfo.fr:9090`.
6. `matrix` a `enable_registration: true` (si vous voulez tester l’inscription via Element).

---

## 3) Tests techniques rapides (avant le test “navigateur”)

Ces tests évitent de perdre du temps sur une erreur “bête”.

### A. Sur `rproxy` : vérifier Nginx + routage

1. Nginx actif :

```bash
sudo systemctl status nginx --no-pager
```

Attendu : `active (running)`.

2. Écoute HTTP :

```bash
sudo ss -lntp | grep ':80'
```

Attendu : LISTEN sur `:80`.

3. Test routage vers Synapse (en simulant le Host) :

```bash
curl -s -H "Host: matrix.phys.iutinfo.fr" http://localhost/_matrix/client/versions
```

Attendu : réponse JSON.

4. Test routage vers Element :

```bash
curl -I -H "Host: element.phys.iutinfo.fr" http://localhost/
```

Attendu : réponse HTTP (souvent `200 OK`).

### B. Sur `matrix` : vérifier service + DB

1. Synapse actif :

```bash
sudo systemctl status matrix-synapse --no-pager
```

Attendu : `active (running)`.

2. Vérifier les logs (erreurs DB notamment) :

```bash
sudo tail -n 50 /var/log/matrix-synapse/homeserver.log
```

Attendu : pas d’erreur PostgreSQL.

3. (Optionnel mais conseillé) test DB depuis `matrix` :

```bash
psql -h db -U synapse_user -d synapse -c "SELECT 1;"
```

Attendu : `1`.

---

## 4) Accès à l’interface Web (test utilisateur)

### A. Depuis la machine physique (réseau université)

1. Ouvrir Firefox/Chrome.
2. Aller sur :

* `http://element.phys.iutinfo.fr:9090`


![Résultat attendu :](../../ressources/5/welcom-matrix.png "welcom")

* page Element visible,
* boutons “Se connecter” / “Créer un compte”.

Si la page ne charge pas :

* vérifier le tunnel/forward `:9090`,
* vérifier `rproxy` et le bon `phys`.

---

## 5) Création de compte (test crucial)

Ce test prouve :

* Element → `rproxy` → Synapse
* Synapse → PostgreSQL (écriture DB)

1. Cliquer sur “Créer un compte”.

2. Vérifier le serveur affiché dans l’écran d’inscription :

* vous devez voir quelque chose du type :

  * **hébergé par `matrix.phys.iutinfo.fr:9090`**

Si vous voyez `matrix.org` :

* le `config.json` de la VM `element` est mauvais.

Correction rapide :

* cliquer “Modifier” / “Edit”
* renseigner :

  * `http://matrix.phys.iutinfo.fr:9090`

3. S’inscrire (exemple de test) :

* Nom d’utilisateur : `testuser`
* Mot de passe : `secret123`
* Email : vide
* Valider l’inscription

Erreurs fréquentes :

* “Registration has been disabled” :

  * sur `matrix`, il manque `enable_registration: true` dans `homeserver.yaml` puis redémarrage.

---

## 6) Connexion et envoi de message (validation finale)

1. Si la création a réussi, vous êtes connecté (ou connectez-vous avec le compte créé).

2. Créer un salon :

* cliquer sur `+` (Rooms/Salons)
* “Créer un nouveau salon”
* Nom : `Test Architecture`
* créer

3. Envoyer un message :

* envoyer : `Bonjour ceci est un test distribué`

Résultat attendu :

* le message apparaît envoyé (pas bloqué “en attente”),
* pas d’erreur côté UI.

---

## 8) Probleme courant

| Symptôme                                | Cause probable                                 | Diagnostic                                                                               | Solution                                                                 |
| --------------------------------------- | ---------------------------------------------- | ---------------------------------------------------------------------------------------- | ------------------------------------------------------------------------ |
| Page blanche / site inaccessible        | tunnel absent / mauvais `phys` / rproxy éteint | depuis machine physique : `curl -I http://localhost:9090`                                | relancer tunnel, vérifier `rproxy`, vérifier URL                         |
| Roue infinie à la connexion             | Element ne joint pas Matrix                    | vérifier `config.json` sur `element`                                                     | `base_url` doit être `http://matrix.phys.iutinfo.fr:9090` + hard refresh |
| “Invalid Homeserver”                    | reverse proxy mal routé                        | sur `rproxy` : `curl -H "Host: matrix.phys..." http://localhost/_matrix/client/versions` | corriger le bloc `matrix` (proxy_pass vers `10.42.XX.1:8008`)            |
| “Internal Server Error” à l’inscription | problème DB côté Synapse                       | sur `matrix` : `sudo tail -f /var/log/matrix-synapse/homeserver.log`                     | corriger password DB / host DB / pg_hba, puis restart synapse            |
| “Registration disabled”                 | inscription désactivée dans Synapse            | vérifier `enable_registration`                                                           | ajouter `enable_registration: true`, restart                             |

---

- Page precedente: [5.6 : Installation et configuration du Reverse Proxy](config-rproxy.md)
- Page suivante: [Sommaire Partie 5](README.md)
