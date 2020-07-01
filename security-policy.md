# Politique de sécurité

Le principal intérêt de modifier une console de jeux est de pouvoir exécuter des copies de jeux non conformes, dites piratées, pour en supprimer le coût monétaire. Due à la grande popularite des consoles et de certains jeux, les moyens mis en place pour découvrir puis exploiter des failles de sécurité sont d'ampleurs.

**Le but est simple: garantir la conformité des jeux via l'intégrite du système.**

Les consoles ont beaucoup evolué depuis les premières générations, notamment au niveau du support de distribution des jeux. La distribution dématerialisée s'est étendue depuis les plateformes PC telle Steam pour devenir une sorte de standard (coûts de distribution amoindris, contrôle des données et des flux de transmission). Même s'il existe encore les supports physique (boites de jeux), ils ne contiennent souvent qu'un code, ou un CD avec une clé permettant l'activation à distance du contenu.  
Par conséquent, la source de vérité de la légitimité d'une contenu à été déplacée de la console (chez le consommateur) vers des services et serveurs distants disposant de moyens plus robustes.
Ces nouveaux moyens de distribution ont amené à devoir développer des systèmes qui ne gèrent plus seulement la consommation des jeux mais également leur distribution. Les consoles sont devenues d'abord interconnectées avec le *live*, des sessions multijoueurs en ligne sur des serveurs de l'éditeur du jeu ou du constructeur de la console. Puis les comptes utilisateurs, les bibliothèques dématerialisées et synchronisées dans le *cloud*, etc ... Ces systèmes partagent d'avantage de similitudes avec nos ordinateurs dit de bureau, se rapprochant désormais plus d'une plateforme multimédia connectée qu'une console dédiée uniquement aux jeux.

#### Fiabilité et Maîtrise 
Pour moi, la plus grande différence de la console comparé à un ordinateur est son appartenance au constructeur. Les consoles sont commercialisées pour un usage précis et l'utilisateur est à la mercie du fabricant car il ne contrôle ni matériel ni logiciel.  
Cette maîtrise totale du système par le fabricant n'est pas problématique car le consommateur en est conscient lors de l'achat, lui garantissant une fiabilité du produit pour l'usage destiné. Sans modifications de configuration ni variété de plateformes à supporter, le système est dimensionné et testé dans le cadre de l'usage pour lequel il est commercialisé.  

Une seconde différence majeure est qu'une console n'est pas orientée utilisateur. Si on regarde l'architecture du système, un *dashboard* (espace dédié à l'utilisateur) remplace GNU. Le système est concu pour ne gérer qu'un utilisateur: le propriétaire de la console. Celui-ci est confiné dans le *dashboard*, il n'administre pas son système.  
Il n'y a pas non plus besoin d'un environnement CLI et de tout les utilitaires pour de la maintenance ou de l'administration à distance que fournis GNU de base car les flottes de consoles sont mises à jour à distance.

#### Sécurité  
Les consoles sont commercialisées en masse pour consommer des divertissements et doivent offrir le meilleur rendement qualité prix aux consommateurs pour leur expérience de jeux. Cependant, si la sécurité ne semble pas un prix à payer pour les utilisateurs (encore que, avec la démocratisation des consoles connectées et comptes utilisateur), c'est un critère primoridal pour les éditeurs de jeux sans quoi leur travail pourrait être detourné. 

## Contraintes

**Système d'exploitation GNU/Linux**  
Les console sont designée sur mesure pour répondre à un besoin précis. Le matériel est contrôlé et le logiciel développé pour celui-ci.  
Le noyau Linux supporte un maximum de configurations matériel possible et GNU est un environnement multi-utilisateur. Dans les systèmes de console, il n'y a qu'un kernel pour l'exécution des jeux et un hyperviseur pour gérer le système.  
GNU/Linux n'est pas adapté à ce besoin, il faut le modifier lourdement ou ne prendre que le noyau Linux configuré exactement pour son besoin.  

**Carte Up Squared**  
Carte de développement architecture x86, processeur Intel, standard UEFI, multiples ports I/O (dont GPIO).  
Distribution Yocto fournie, alternative possible avec support de distribution Ubuntu bureau ou serveur.  
N'ayant pas le temps de faire ce projet sous Yocto, je me suis tourné vers une distribution *Ubuntu server* car plus legère que celle de bureau (pas d'interface graphique).

## Modèle de menace

Dans ce projet, j'essaierai de tenir compte des aspects affectant les anciennes générations de consoles (système autonome avec support physique) comme les nouvelles (système connecté avec jeux dématerialisés) à travers différentes attaques qui menaces:
- la conformité des jeux exécutés d'une part
- l'intégrité du système qui maintient cette conformité d'autre part

### Détournement des vérifications
Il n'est pas nécessaire de compromettre tout le système pour bénéficier de jeux piratés, il suffit d'empêcher ou tromper la vérification de conformité de ceux-ci au lancement via un gain de privilège logiciel ou encore des modifications / glitchs matériel.  
*Cette problématique s'adresse principalement aux anciennes générations disposant de supports physique, c'est plus compliqué pour les versions dématerialisées*.

**Vulnérabilités**
::: danger
- primitives cryptographiques biaisées
- racine de confiance compromise
:::

**Contre mesures**
::: secure
- fiabilité des implémentations choisies
- configuration selon recommandations
- évaluation de la racine de confiance (ROM / ASIC, TPM, flash...)
:::

### Exécution arbitraire de code
Si un attaquant parvient à obtenir des privilèges nécessaires pour exécuter n'importe quel programme sur la console, il peut alors lancer ces propres copies de jeux.

**Vulnérabilités**
::: danger
- éxécutables SUID
- faille logiciel
- configuration système de fichiers
:::

**Contre mesures**
::: secure
- signature des éxécutables (système et jeux)
- Isolation des processus (`namespaces`)
- Restriction des capacités en fonction des besoins (`MAC` / `capabilities`)
- Configuration des points de montage (`nosuid`)
:::

### Modification de la mémoire
Permet de tricher sur les jeux dont la console est la source de vérité. Ce sont généralement les jeux locaux unipersonnel, car les jeux en ligne ne font jamais confiance au client mais uniquement à leurs serveurs.

**Vulnérabilités**
::: danger
- attaques via DMA (GPU notamment)
- faille logicielle (programmes système)
:::

**Contre mesures**
::: secure
- IOMMU / MMU
- application du `Write xor eXecute` sur les zones mémoire (`NX` bit)
- chiffrement de la mémoire des jeux (support matériel requis)
- intégrité de la mémoire système
:::

### Extraction de secrets
Même si la console ne s'occupe plus de garantir la légitimité d'un jeu lors d'une version dématerialisée, tout le code source de celui-ci est téléchargé localement pour pouvoir y jouer.  
Un attaquant peut donc extraire ce code pour copier le jeu.

**Vulnérabilités**
::: danger
- dump du disque
:::

**Contre mesures**
::: secure
- chiffrement de la partition contenant les jeux (ou directement de chaque jeu avec clef dédiée)
:::

### Corruption du système
Modifier tout ou partie du système depuis son support de stockage (mémoire flash ou support amovible) pour notamment empêcher certaines vérifications.

**Vulnérabilités**
::: danger
- remplacement du système pour détourner la console (perte de contrôle du constructeur)
- modification des fichiers de configuration (ou options CLI) lors du démarrage
- rétrogradage du système de la console
:::

**Contre mesures**
::: secure
- chaîne de démarrage sécurisée UEFI
- racine de confiance dans le firmware
- ajout des signatures d'anciennes versions dans `dbx` (seule solution sans support matériel dédié)
:::

## Contre mesures

Concernant la partie physique, la console étant exposée chez le client, les niveaux de sécurité mis en place permettent de décourager les attaquants amateurs et ralentir ceux disposant de moyens modestes.  
Le système place sa confiance dans le matériel et les moyens de sécurité qu'il offre (car considérés comme beaucoup plus sûr). La racine de confiance se trouve dans le *firmware* qui enclenche le démarrage sécurisé et assure l'intégrité du système.  

Normalement, les consoles ne disposent pas d'environement interne (shell, utilitaires CLI) et cette menace d'intrusion n'est pas présente. Les seules menaces sont des attaques directement sur les protections mémoire ou stockage, les niveaux de droits matériel et les failles logicielles.  
Vu que GNU fournis un environnement entier et que Linux est beaucoup plus vaste (et complexe) que les systèmes habituels de console, il faut mettre en place des contre mesure logicielles pour ressources et processus.  
Dans ce cadre ci, la règle des moindres privilèges est appliquée en plus d'un cloisonnement des services permettant de minimiser l'impact d'une instrusion ou de l'exploitation d'une vulnérabilité dans l'attente d'un correctif de sécurité.

### Mise en place d'une chaîne de confiance
Grâce au standard UEFI supporté par la plateforme, un démarrage sécurisé garantie l'intégrité du système de la console.

### Chiffrement des données sensibles
La console contient des sources privées appartenant aux éditeurs de jeux comme aux concepteurs du système d'exploitation, des journaux pouvant révéler des informations sensibles et surtout des clés de chiffrement pour authentifier et communiquer à distance.  
Toutes ces données sont des cibles potentielles pour l'extraction et doivent ainsi être stockées sur un disque ou une partition chiffrée et dechiffrée lorsque le système est considéré stable et intègre (c'est à dire après démarrage).

### Partitionnement
Au minima 2 partitions permettent de gérer le système (en dehors de l'ESP):
- programmes et configuration système (rootfs) en lecture seule avec exécution
- configuration utilisateur et journaux système en lecture écriture sans exécution

Cette configuration pose tout le même le problème du téléchargement et stockage de nouveaux jeux. On pourrait envisager de créer une partition lecture seule avec exécution qui serait mise à jour à chaque installation d'un jeu (redémarrage système non nécessaire, seulement mise à jour de l'intégrité).

### Système de mise à jour
Il est nécéssaire de pouvoir garder un contrôle du constructeur sur la console et répondre aux vulnérabilités comme aux ajouts de fonctionnalités.  
Il faut établir un canal de communication sécurisé avec le serveur et vérifier l'authenticité des mises à jour dans un premier temps puis avoir un mécanisme d'installation de mise à jour pouvant se remettre d'erreurs (partitions A/B) pour garder un état stable.

### Monitoring
Surveillance de l'état du système, rapatriement des journaux pour étude chez le constructeur.  
Permet ainsi de suivre l'activité des consoles et prendre des mesures si certaines sont en compromission (bannissement des consoles ou récupération si possible).

### Séparation des rôles
Une fois le sytème en route et considéré comme sûr (via les mesures précédentes), il faut le maintenir dans cet état.  
Plutôt que d'assumer que le système est sain et essayer de vérifier ou bloquer chaque modification voir instrusion, je préfère considérer que tout est potentiellement malicieux et isoler chaque programme via des solutions de cloisonnement et de controle d'accès.  
Cette solution ne couvre pas seulement les menaces externes mais aussi internes comme les éditeurs de jeux contaminés ou malicieux.  

Création d'un environnement dédié pour chaque jeu, permet de profiter de la totalité du système (comme atuellement dans les consoles) tout en gardant une isolation, cela offre en plus une flexibilité dans les environnements voulus par les éditeurs.  
Le dashboard est également isolé car soumis aux actions utilisateur, chaque utilisateur peut également modifier son système à sa guise (dans la théorie).  