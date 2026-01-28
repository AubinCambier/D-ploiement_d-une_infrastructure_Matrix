## 5.1 Creation et configuration initiale (VM `matrix`)

### 1) Schema d'architecture de l'adressage

* Le reseau virtuel "principal" de l'IUT est **10.42.0.0/16** (routeur + DNS : **10.42.0.1**).
* L'IUT attribue a chaque etudiant une **plage dediee** : **10.42.XX.0/24** (XX depend du tableau Moodle).
* Toutes vos VMs doivent imperativement utiliser une IP **dans votre plage**.

Representation simple :

* Reseau IUT : 10.42.0.0/16
  * Passerelle/DNS : 10.42.0.1
  * Votre sous-plage : 10.42.XX.0/24
    * VM `matrix` : 10.42.XX.1
    * (Exemple pour une 2e VM : `rproxy` : 10.42.XX.2)

---

### 2) Procedure pas a pas (depuis le PC de TP)

#### A. Trouver votre "XX" (reference Moodle)

1. Ouvrez le tableau d'attribution sur Moodle :
   `https://moodle.univ-lille.fr/pluginfile.php/2793265/mod_resource/content/6/attribution-ips.html`

2. Reperez votre ligne.

   * Exemple : si le tableau montre `aubin.cambier.etu` -> `10.42.118.1-254`, alors **XX = 118**.
   * Dans la procedure, on ecrit toujours **XX** (variable), pas "118" en dur.

3. Choisissez l'IP de la VM `matrix` :

   * Convention : **premiere adresse utilisable** -> `10.42.XX.1`

---

#### B. Initialisation et acces console

1. Connexion a l'hote de virtualisation :

```bash
ssh virt
```

2. Creer et demarrer la VM :

```bash
vmiut creer matrix
vmiut demarrer matrix
```

3. Ouvrir la console :

```bash
vmiut console matrix
```

Identifiants par defaut :

* `user` / `user`
* `root` / `root`

---

#### C. Configuration systeme (en root)

Passez root :

```bash
su -
```

1. Configuration du système :

```bash
apt update
apt install sudo -y
```
2.Installer les outils de base (éditeur, pager, arborescence, synchro, HTTP)
```bash
apt install -y vim less tree rsync curl

```

2. Donner les droits sudo a `user` :

```bash
usermod -aG sudo user
```

3. Nom d'hote (persistant) : choisissez un nom unique (evite la confusion avec d'anciennes VMs)
   Exemple : `matrix-XX`

```bash
hostnamectl set-hostname matrix-XX
```

---

#### D. Reseau statique (IP attribuee Moodle) — remplacer le DHCP

Important : si la VM etait en DHCP avant, il faut **supprimer ou commenter le bloc DHCP**, puis le remplacer par le bloc statique. Sinon, vous risquez d'avoir une configuration incoherente.

1. Verifiez le nom de l'interface (si ce n'est pas `enp0s3`, adaptez) :

```bash
ip link
```

2. Editez `/etc/network/interfaces` :

```bash
sudo nano /etc/network/interfaces
```

3. Reperez l'ancien bloc DHCP (exemple typique) :

```text
allow-hotplug enp0s3
iface enp0s3 inet dhcp
```

4. **Supprimez-le (ou commentez-le)**, puis mettez **a la place** ce bloc statique (en gardant `XX`) :

```text
auto enp0s3
iface enp0s3 inet static
    address 10.42.XX.1/16
    gateway 10.42.0.1
```

Option DNS (si besoin) :

```text
    dns-nameservers 10.42.0.1
```

5. Verifiez qu'il n'existe pas une autre conf reseau qui remet du DHCP (bonne pratique) :

* Regardez si vous avez des fichiers dans `/etc/network/interfaces.d/` qui definissent aussi `enp0s3`.

6. Appliquez la config (a faire de preference depuis la **console** VM pour eviter de vous couper si vous etiez en SSH) :

```bash
sudo ifdown enp0s3 && sudo ifup enp0s3
```

---

#### E. Deploiement SSH (depuis le PC de TP)

Une fois l'IP en place :

```bash
ssh-copy-id user@10.42.XX.1
```

---

### 3) Tests de validation

1. Test "appartenance a la plage" :

```bash
ip addr show enp0s3
```

Vous devez voir `10.42.XX.1`.

2. Test route / passerelle :

```bash
ip route
ping -c 2 10.42.0.1
```

3. Test DNS + Internet :

```bash
host www.univ-lille.fr
apt update
```

4. Test sudo :

```bash
sudo apt update
```

5. Test SSH (depuis PC TP) :

```bash
ssh user@10.42.XX.1
```

---

### 4) Troubleshooting

| Probleme | Cause racine | Solution |
| --- | --- | --- |
| IP deja utilisee | IP hors de votre plage, ou deja prise | Re-verifier la ligne Moodle, rester dans `10.42.XX.1-254`, changer d'IP |
| L'IP change "toute seule" / pas celle attendue | Vous avez laisse un bloc `inet dhcp` (ou une conf dans `interfaces.d`) | Supprimer/commenter le DHCP et garder uniquement le bloc `inet static` |
| `apt update` ne marche pas | passerelle/masque/DNS faux | Verifier `gateway 10.42.0.1`, `/16` + `255.255.0.0`, ajouter `dns-nameservers 10.42.0.1` si besoin |
| `ifdown: interface not configured` | mauvais nom d'interface | `ip link`, remplacer `enp0s3` par le bon nom |
---

- Page precedente: [5.0 : Pre-requis (VM et reseau)](Pré-requis.md)
- Page suivante: [5.2 : Configuration de la VM Element](config-element-vm.md)
