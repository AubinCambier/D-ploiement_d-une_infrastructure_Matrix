## 4.2 Introduction et choix d'un reverse proxy

### Presentation du concept

Dans une architecture reseau professionnelle, il est deconseille d'exposer directement un serveur d'application (comme Synapse) aux requetes des clients. On utilise pour cela un reverse proxy.

Contrairement a un proxy classique qui permet aux utilisateurs d'un reseau local d'acceder a Internet, le reverse proxy agit a l'inverse : il intercepte les requetes provenant de l'exterieur et les redirige vers le bon service interne.

---

### Schéma d'architecture (Flux Reverse Proxy)

```text
                                     RÉSEAU PRIVÉ (10.42.XX.0/24)
                                    |
[ UTILISATEUR ] --(Requête HTTP)----|------> [ VM RPROXY (10.42.XX.2) ]
      ^                             |               |
      |                             |               | (Aiguillage)
      |                             |               v
      |                             |        [ VM MATRIX (10.42.XX.1) ]
      |                             |               |
[ RÉPONSE HTTP ] <------------------|---------------|-- [ Synapse : 8008 ]
                                    |
```

---

### Justification de l'utilisation d'un reverse proxy

L'introduction de ce service dans notre infrastructure repond a trois besoins majeurs :

* Performance : un serveur web est bien plus efficace pour livrer du contenu statique qu'un serveur d'application Python comme Synapse.
* Securite : le reverse proxy sert de bouclier. Il est le seul point d'entree visible, masquant ainsi l'adresse reelle et la structure de la VM Matrix.
* Evolutivite (hebergement mutualise) : il permet de diriger le trafic vers differents services (Synapse ou Element) en fonction du nom de domaine utilise, le tout sur une seule adresse IP.

---

### Evaluation des solutions de reverse proxy

Nous avons etudie trois logiciels capables de remplir ce role afin de selectionner le plus adapte a notre deploiement.

| Criteres | Nginx | Apache (mod_proxy) | HAProxy |
| --- | --- | --- | --- |
| Role principal | Serveur web et proxy | Serveur web modulaire | Proxy et load balancer pur |
| Consommation RAM | Tres faible | Moyenne a elevee | Tres faible |
| Contenu statique | Oui (excellent) | Oui | Non (incapable d'heberger Element) |
| Configuration | Simple et concise | Verbeuse (beaucoup de lignes) | Specifique (tres technique) |

---

### Justification du choix : Nginx

Le choix de notre equipe s'est porte sur Nginx pour les raisons suivantes :

* Polyvalence : contrairement a HAProxy, Nginx peut a la fois rediriger le trafic vers Synapse et servir les fichiers d'Element Web. Cette polyvalence est un atout majeur pour l'evolution de notre architecture.
* Simplicite de mise en oeuvre : la configuration d'un transfert de flux se fait via une directive unique (`proxy_pass`), ce qui limite les erreurs de parametrage pour un premier deploiement.
* Legerete : Nginx est concu pour consommer un minimum de ressources, ce qui est crucial pour notre nouvelle machine virtuelle dediee `rproxy`.
* Coherence globale : utiliser Nginx pour Element Web et pour le reverse proxy nous permet d'uniformiser notre environnement technique et de faciliter la maintenance du projet.

---

## Section dédiée aux problèmes (Analyse de risques)

| Problème potentiel | Cause probable | Mesure préventive |
| :--- | :--- | :--- |
| **Point de défaillance unique (SPOF)** | Si le reverse proxy tombe, tout le service Matrix est inaccessible. | Prévoir une configuration simple et des sauvegardes du fichier `.conf`. |
| **Mauvaise redirection** | Erreur de configuration de l'IP cible (10.42.XX.1). | Vérifier systématiquement l'IP statique de la VM Matrix avant configuration. |
| **Lenteur des requêtes** | Trop de redirections successives ou mauvaise gestion du cache. | Utiliser Nginx pour sa rapidité de traitement des requêtes entrantes. |
| **Incompatibilité Element Web** | Choix d'un proxy pur (HAProxy) ne pouvant pas servir de fichiers. | **Choix validé de Nginx** qui permet de servir du contenu statique. |

---

## Section Tests de validation (Théoriques)

Pour valider que le choix technologique est opérationnel, les tests suivants seront réalisés en étape 4.3 et 4.4 :

1.  **Test de présence réseau :** Vérifier que la VM `rproxy` ping bien la VM `matrix`.
2.  **Test de syntaxe :** Utilisation de `nginx -t` pour valider les fichiers de configuration.
3.  **Test d'en-tête (Header) :** Vérifier que les requêtes reçues par Synapse contiennent bien l'adresse IP d'origine du client (via `X-Forwarded-For`).

<hr>

Etape suivante : [4.3 Installation et configuration de la VM `rproxy`](install-config-Vm-proxy.md)

- Page précédente: [4.1 : Choix du serveur web pour Element Web](choix-server-element.md)
- Page suivante:  [4.3 : Installation et configuration réseau de la VM rproxy](install-config-Vm-proxy.md)