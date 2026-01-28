## 5.2 Creation et configuration initiale (VM `element`)

### 1) Schema d'architecture de l'adressage

* Reseau virtuel principal IUT : `10.42.0.0/16`
* Passerelle (et souvent DNS) : `10.42.0.1`
* Plage dediee par etudiant (via Moodle) : `10.42.XX.0/24`
* IP choisie pour cette VM :
  * VM `element` : `10.42.XX.4` (4e adresse utilisable de votre plage)

---

### 2) Procedure pas a pas (depuis le PC de TP)

#### A. Trouver votre "XX" (reference Moodle)

Ouvrez le tableau d'attribution IP sur Moodle :
`https://moodle.univ-lille.fr/pluginfile.php/2793265/mod_resource/content/6/attribution-ips.html`

Reperez votre ligne.

Exemple : `... -> 10.42.118.1-254` => **XX = 118**.

Dans la procedure, on garde **XX** (variable), jamais "118" en dur.

---

#### B. Initialisation et acces console

Connexion a l'hote de virtualisation :

```bash
ssh virt
```

Creer et demarrer la VM :

```bash
vmiut creer element
vmiut demarrer element
```

Ouvrir la console :

```bash
vmiut console element
```

Identifiants par defaut :

* `user` / `user`
* `root` / `root`

---

#### C. Configuration systeme (en root)

Passer root :

```bash
su -
```

Mise a jour + installation des outils de base :

```bash
apt update
apt install -y sudo vim less tree rsync curl
```

Donner les droits sudo a l'utilisateur `user` :

```bash
usermod -aG sudo user
```

Definir un nom d'hote unique :

```bash
hostnamectl set-hostname element-XX
```

---

#### D. Reseau statique (IP `10.42.XX.4`) — remplacement du DHCP

Objectif : desactiver le DHCP et figer l'IP. Cela signifie : on supprime ou commente l'ancien bloc `inet dhcp`, puis on le remplace par un bloc `inet static`.

Verifiez le nom de l'interface (adaptez si ce n'est pas `enp0s3`) :

```bash
ip link
```

Editez le fichier :

```bash
nano /etc/network/interfaces
```

Reperez l'ancien bloc DHCP (exemple typique) :

```text
iface enp0s3 inet dhcp
```

Supprimez-le (ou commentez-le), puis mettez a la place ce bloc statique (note : ici on standardise avec `auto enp0s3`) :

```text
auto enp0s3
iface enp0s3 inet static
    address 10.42.XX.4/16
    gateway 10.42.0.1
```

A noter : `auto` et `allow-auto` sont equivalents, et `allow-hotplug` existe aussi ; ici on garde `auto` pour rester coherent dans la procedure.

Appliquez la configuration depuis la console (recommande) :

```bash
ifdown enp0s3 && ifup enp0s3
```

---

#### E. Deploiement SSH (depuis le PC de TP)

Une fois l'IP active :

```bash
ssh-copy-id user@10.42.XX.4
```

---

### 3) Tests de validation

| Test | Commande | Resultat attendu |
| --- | --- | --- |
| Verification IP | `ip addr show enp0s3` | Affiche `inet 10.42.XX.4/16` |
| Test passerelle | `ping -c 2 10.42.0.1` | 0% de perte |
| Test DNS | `host www.univ-lille.fr` | Retourne une IP |
| Test sudo | `sudo whoami` | Repond `root` |
| Test SSH | `ssh user@10.42.XX.4` | Connexion OK |

---

### 4) Troubleshooting

| Probleme | Cause racine | Solution |
| --- | --- | --- |
| Destination Host Unreachable | IP en conflit / interface mal montee / mauvaise interface | Verifier que .4 est libre dans votre plage, verifier le nom d'interface (`ip link`), relancer `ifdown/ifup` |
| L'IP ne "tient" pas | DHCP encore present | Verifier que le bloc `inet dhcp` est bien supprime/comment et qu'il n'y a pas une autre conf ailleurs |
| Acces Internet impossible | passerelle/masque/DNS faux | Verifier `gateway 10.42.0.1`, `/16` + `255.255.0.0`, `dns-nameservers 10.42.0.1` |
| `sudo` ne marche pas | session non relancee apres ajout au groupe | `exit` puis reconnexion (ou `su - user`) |

---

- Page precedente: [5.1 : Configuration de la VM Matrix](config-matrix-vm.md)
- Page suivante: [5.3 : Mise en place de PostgreSQL (VM db)](config-postgresql-vm.md)
