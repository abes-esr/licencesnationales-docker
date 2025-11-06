# licencesnationales-docker

Configuration docker 🐳 pour déployer l'application de gestion des accès de licencesnationales. Ce README fait également office de fiche d'exploitation.

## URLs de licencesnationales

Les URL de l'applications sur les différents environnements :

- prod :
  - https://acces.licencesnationales.fr/ : URL publique de licencesnationales-front
  - https://acces.licencesnationales.fr/v1/ : URL publique de licencesnationales-web
  - http://diplotaxis3-prod.v102.abes.fr:10401/ : URL interne de licencesnationales-front (accès via VPN)
  - http://diplotaxis3-prod.v102.abes.fr:10400/v1/ : URL interne de licencesnationales-web (accès via VPN)
- test :
  - https://acces-test.licencesnationales.fr/ : URL publique de licencesnationales-front (accès via IP Renater)
  - https://acces-dev.licencesnationales.fr/v1/ : URL publique de licencesnationales-web (accès via IP Renater)
  - http://diplotaxis3-test.v202.abes.fr:10401/ : URL interne de licencesnationales-front (accès via VPN)
  - http://diplotaxis3-test.v202.abes.fr:10400/v1/ : URL interne de licencesnationales-web (accès via VPN)
- dev :
  - https://acces-dev.licencesnationales.fr/ : URL publique de licencesnationales-front (accès via VPN)
  - https://acces-dev.licencesnationales.fr/v1/ : URL publique de licencesnationales-web (accès via VPN)
  - http://diplotaxis3-dev.v212.abes.fr:10401/  : URL interne de licencesnationales-front (accès via VPN)
  - http://diplotaxis3-dev.v212.abes.fr:10400/v1/  : URL interne de licencesnationales-web (accès via VPN)
- local :
  - http://127.0.0.1:10401/ : URL interne de licencesnationales-front
  - http://127.0.0.1:10400/v1/ : URL interne de licencesnationales-web


## Prérequis

Disposer de :
- ``docker``
- ``docker-compose``

## Installation

Déployer la configuration docker dans un répertoire :
```bash
# adaptez /opt/pod/ avec l'emplacement où vous souhaitez déployer l'application
cd /opt/pod/
git clone https://github.com/abes-esr/licencesnationales-docker.git
```

Configurer l'application depuis l'exemple du [fichier ``.env-dist``](./.env-dist) (ce fichier contient la liste des variables avec des explications et des exemples de valeurs) :
```bash
cd /opt/pod/licencesnationales-docker/
cp .env-dist .env
# personnaliser alors le contenu du .env
```

**Note : les mots de passe de la base de donnée xml de test ne sont pas présent dans le fichier au moment de la copie. Vous devez les renseigner manuellement en editant le fichier .env dans la console avec nano par exemple**

Démarrer l'application :
```bash
cd /opt/pod/licencesnationales-docker/
docker-compose up -d
```

## Démarrage et arret

Pour démarrer l'application :
```bash
cd /opt/pod/licencesnationales-docker/
docker-compose up -d
```

Pour arrêter l'application :
```bash
cd /opt/pod/licencesnationales-docker/
docker-compose stop
```

## Supervision

Pour vérifier que l'application est démarrée, on peut consulter l'état des conteneurs :
```bash
cd /opt/pod/licencesnationales-docker/
docker-compose ps
```

Pour vérifier que l'application est bien lancée, on peut consulter les logs :
```bash
cd /opt/pod/licencesnationales-docker/
docker-compose logs --tail=50 -f
```
A noter que ces logs sont envoyées automatiquement au puits de log de l'Abes à l'aide du client filebeat installé sur le noeud docker et à la configuration que nous lui indiquons dans les labels.

## Sauvegardes

Pour sauvegarder l'application, il faut :
- Sauvegarder la base de données (base Oracle sur les serveurs "orpins"). En plus des sauvegardes de type RMan , des exports sont réalisés avec expdp.
- Sauvegarder le fichier ``/opt/pod/licencesnationales-docker/.env`` qui est un fichier non versionné et qui permet de configurer tous les conteneurs docker de l'application (les autres fichiers sont versionnés sur le github de ``licencesnationales-docker``)

Ces deux opérations sont prises en charge par le SIAT.

## Restauration
### Restauration de la base de données Oracle

- se connecter avec le compte oracle sur ononis (machine Oracle) :
 
- Récupérer la sauvegarde depuis sotora : 
```bash
rsync --progress -av devel@socorro.v104.abes.fr:/backup_pool/ononis-prod-dumps/daily.0/racine/backup-sql/ABES/LICENCESNATIONALES/dumpLICENCESNATIONALES.dmp /backup-sql/ABES/LICENCESNATIONALES/dumpLICENCESNATIONALES.dmp
```
*Pour sélectionner une sauvegarde autre que la plus récente, il suffit de remplacer daily.0 dans la commande par le jour souhaité (daily.1 pour la veille, daily.2 pour l'avant-veille, etc.)*

- Puis lancer les commandes suivantes pour importer le dump dans la base de données :

```bash
export NLS_LANG=AMERICAN_AMERICA.UTF8
export ORACLE_SID='ABES'
setsid impdp \'/ as sysdba\' SCHEMAS=LICENCESNATIONALES TABLE_EXISTS_ACTION=REPLACE dumpfile='dumpLICENCESNATIONALES.dmp' logfile=importLicencesNationales.log directory=DPDUMP_LICENCESNATIONALES
```
Si le directory object n'existe pas dans la base de données, il faut le créer : 
```bash
CREATE DIRECTORY DPDUMP_LICENCESNATIONALES AS '/backup-sql/ABES/LICENCESNATIONALES';
```

### Réinstallation de l'application

- Se connecter avec son compte développeur sur la machine de déploiement diplotaxis4-prod (via Putty etc.)

- Se positionner dans le répertoire des applications :
```bash
cd /opt/pod
```
- Récupérer le projet licencesnationales-docker et se positionner dans le répertoire :
```bash
git clone https://github.com/abes-esr/licencesnationales-docker.git
cd licencesnationales-docker
```
- Récupérer le .env depuis sotora (authentification nécessaire) :
```bash
rsync -av devel@sotora.v104.abes.fr:/backup_pool/diplotaxis4-prod/daily.0/racine/opt/pod/licencesnationales-docker/.env /opt/pod/licencesnationales-docker/.env
```
- Lancer les containers : 
```
sudoc docker compose up -d
```
## Déploiement continu

Les objectifs des déploiements continus de licencesnationales sont les suivants (cf [poldev](https://github.com/abes-esr/abes-politique-developpement/blob/main/01-Gestion%20du%20code%20source.md#utilisation-des-branches)) :
- git push sur la branche ``develop`` provoque un déploiement automatique sur le serveur ``diplotaxis1-dev``
- git push (le plus couramment merge) sur la branche ``main`` provoque un déploiement automatique sur le serveur ``diplotaxis1-test``
- git tag X.X.X (associé à une release) sur la branche ``main`` permet un déploiement (non automatique) sur le serveur ``diplotaxis1-prod``

Licencesnationales est déployé automatiquement en utilisant l'outil watchtower. Pour permettre ce déploiement automatique avec watchtower, il suffit de positionner à ``false`` la variable suivante dans le fichier ``/opt/pod/licencesnationales-docker/.env`` :
```env
LN_WATCHTOWER_RUN_ONCE=false
```

Le fonctionnement de watchtower est de surveiller régulièrement l'éventuelle présence d'une nouvelle image docker de ``licencesnationales-web``, ``licencesnationales-batch`` et ``licencesnationales-front``, si oui, de récupérer l'image en question, de stopper le ou les les vieux conteneurs et de créer le ou les conteneurs correspondants en réutilisant les mêmes paramètres ceux des vieux conteneurs. Pour le développeur, il lui suffit de faire un git commit+push par exemple sur la branche ``develop`` d'attendre que la github action build et publie l'image, puis que watchtower prenne la main pour que la modification soit disponible sur l'environnement cible, par exemple la machine ``diplotaxis1-dev``.

Le fait de passer ``LN_WATCHTOWER_RUN_ONCE`` à false va faire en sorte d'exécuter périodiquement watchtower. Par défaut cette variable est à ``true`` car ce n'est pas utile voir cela peut générer du bruit dans le cas d'un déploiement sur un PC en local.


## Mise à jour de la dernière version

Pour récupérer et démarrer la dernière version de l'application vous pouvez le faire manuellement comme ceci :
```bash
docker-compose pull
docker-compose up
```
Le ``pull`` aura pour effet de télécharger l'éventuelle dernière images docker disponible pour la version glissante en cours (ex: ``develop-api`` ou ``main-api``). Sans le pull c'est la dernière image téléchargée qui sera utilisée.

Ou bien [lancer le conteneur ``licencesnationales-watchtower``](https://github.com/abes-esr/licencesnationales-docker/blob/develop/README.md#d%C3%A9ploiement-continu) qui le fera automatiquement toutes les quelques secondes pour vous.


## Architecture

![](https://docs.google.com/drawings/d/e/2PACX-1vQtkQo11L20Tc_OTkz6o3t2pqPKK4jdp0gNAd1lpyA_r1ExQGKTnTAZ09nuJl9DnGpUkbjzlgzflH5s/pub?w=1268&amp;h=641)  
([lien](https://docs.google.com/drawings/d/1vlTB03DNbbT8l-0Ca4KpGdLxyKGHqdRo1UKT39aJjIA/edit) pour modifier le schéma)

A noter que les images docker de licencesnationales sont générées à partir des codes open sources disponibles ici :
- https://github.com/abes-esr/licencesnationales-back/ via la chaine d'intégration continue [build-test-pubtodockerhub.yml](https://github.com/abes-esr/licencesnationales-back/actions/workflows/build-test-pubtodockerhub.yml) qui pousse les images sur [dockerhub ici](https://hub.docker.com/r/abesesr/licencesnationales/)
- https://github.com/abes-esr/licencesnationales-front/ via la chaine d'intégration continue [build-test-pubtodockerhub.yml](https://github.com/abes-esr/licencesnationales-front/actions/workflows/build-test-pubtodockerhub.yml) qui pousse les images sur [dockerhub ici](https://hub.docker.com/r/abesesr/licencesnationales/)
