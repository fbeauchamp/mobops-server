
#Mobops (serveur)

Le serveur est chargé de 
 
* Recolter et maintenir des données à jour provenant du système d'alerrte , du SIG et du SIRH
* Protéger ces SI en ne les exposant pas directement. par construction une surcharge de mobops ne doit pas avoir d'impacts sur les serveurs 
* Exposer via des api sécurisée pour mobops( le clizent)

##Installation
Prérequis : 

* [postgresql](http://postgis.net/) 9.+  avec [HSTORE](http://www.postgresql.org/docs/9.1/static/sql-createextension.html)  et [Postgis 2.x](http://postgis.net/) activés
* [sqlite](http://postgis.net/)  
* Nginx
* Stunnel
* imagemagick 
* télécharger mobops(server)====

Puis lancer les commande suivantes : 
    
    npm install 
    npm install -g pm2
    pm2 start pm2.json

##API
Tous les appels à l'api sont maintenant servis par le script **pulled.js** . Les réponses sont toujours au format  json
####Gestion des alertes

* `PUT /alert/` : l'utilisateur connecté s'inscrit à une alerte
* `GET /alert/` : retourne les alerts de l'utilisateur
* `DELETE /alert/:id/` : l'utilisateur supprime une alerte
* `POST /alert/enable/:id` : l'utilisateur connecté active une de ses alerte
* `POST /alert/disable/:id` : l'utilisateur connecté deactive une de ses alerte
* `GET /alert/message/:token` : affiche un message d'une alerte. Mets à jour le champ `ack_time` de ce message pour le marqué comme lu
* `GET /alert/login/:token` : connecte l'utilisateur à mobops et le redirige vers la page d'accueil

####SIG

* `GET /buildings/:lng_min/:lng_max/:lat_min/:lat_max` : retourne le GeoJSON des batiments de ce secteur
* `GET /tiles/:z/:x/:y.png` : affiche une tuile du fond de carte
* `GET /tiles/metadata.json` : retourne les metadata associés aux tuiles
* `GET /geosearch?X=X_LAMBERT&Y_Y_LAMBERT` : retourne els points d'intéret autour d'un point *Attention, coordonnées en lambert 93*

####Indicateurs  de contexte

* `GET /context` : retourne les indicateurs de contexte (vigipirate , crue, ...)
* `POST /context` : cré ou modifie un  indicateur de contexte. Seulement pour les utilisateurs avec un profil admin

####Authentification 

* `GET /login.json`   Vérifie si l'utilisateur courant a une session express active. Renvoie `{status:'authenticated',authenticated:true}`  dans ce cas, `{status:'not authenticated',authenticated:false}` sinon.  
* `POST /login.json`  Paramètres login et password. Authentifie l'utilisateur auprès de l'Active Directory, le crée dans la base de données à partir de l'AD si besoin. Cre la session dasn express.
* `GET /logout` : déconnecte l'utilisateur

#### Opérations
`GET /operation/current.json` **pulled**  Renvoi la liste des opérations actives accessibles par l'utilisateur en fonction de son authentification (filtrage des attributs, ajout du délai). Ceci permet de bénéficier de la compression json pour le premier chargement plutôt que de tout passer par les websocket
####Dispo

* `GET /dispo` : retourne le listing des agents dispos avec leur centre et leur qualif
* `GET /dispo/log/:center_id` : retourne les changement au cours des 6 dernières heures sur la dispo de ce centre
* 
#### Upload

* `POST /upload` **pulled** paramètes : operation_id, des pièces jointes. upload un fichier. Si il s'agit d'une image, il crée une miniature.
* `GET /upload/:file_id` **pulled**  télécharge un fichier uploadé précédemment.

#### Sitac

* `PUT /poly` **pulled**  paramètres : lats, lngs, color, pattern, operation_id. Crée une nouvelle plyline dans la base de données. utilisé pour gérer les dessins de SITAC.
* `DELETE /poly` **pulled** paramètre : id. Efface une polyline.

#### Photos des utilisateurs
`GET /mugshot/:matricule/avatar.png` : retourne la photo de l'utilisateur en fonction de son matricule

#### Interface avec l'extranet
`GET /portail/consigne/:center_id` : retourne un json des pages dans le dossier du centre sur l'extranet. Utilisé pour récupérer des consignes
 

##Architecture

###Base de données

La majeure parti des tables ont un champ `attributes` de type _hstore_ . L'idée est que les champs communs soient matérialisés par de vraies colonnes et que les champs qui varient soit stockés dans dans le champ HSTORE. **Attention**, toutes les données du champ HSTORE sont au format chaine de caractères. 

Pour simplifier l'accès à la base de données, un ORM simpliste est présent dans `/lib/persister.js` avec les fonctionnatlités suivantes : 

* découverte auto de la structure des tables : colonnes, champs matérialisés, index unique
* fonction upsert ( update if exist, insert if not exists) qui utilise les indexes uniques et primaire de l'autodécouverte pour mettre à jour le bon enregistrement
* fonction insdate ( insert puis update)
* get : retourne un enregistrement par son id
* where : recherche d'un ensemble d'enregistrement validant des filtres. Filtres supportées
    * field:value
    * field: null
    * field: 'not null'
* query : execute une requete, et décode les champs du hstore      
* find : comme where mais ne retourne qu'un enregistrement
* ecoute des table emettant évenement sur les changements


Toutes les tables ont des triggers qui émettent des évenements lors d'insertion et de mise à jour. Dans le cas des mise à jours seuls les champs modifiées et les champs permettant d'identifier l'enregistrement concernées sont emis.


###Micro services
**synchronizator** . Chargé de récuperer les données du système d'information d'alerte. 

* Renseigne les tables `dipos`, `agent`, `vehicle`, `operation`, `message`
 
**authorizator**  Chargé de construire de matérialiser les différent niveau d'accès aux données, et d'associer ces données filtrées aux profils. 

* Ecoute les modifications sur les tables `dispo`, `agent`, `vehicle`,`operation`,`line`,`file` 
* Renseigne les table `filtered_model` et `profil_filtered_model`. Rempli la table `profil_user` uniquement concernant le profil COS spécifique à  une opération 

**profilator** Chargé d'associer les utilisateurs aux profils.

* Ecoute les tables `reference.agent`, `reference.agent_qualification`, `reference.qualification`, `dispo`
* Rempli la table `profil_user`

**pulled** Chargé de répondre aux requêtes HTTP

* Exposé sur internet par nginx
* Interroge les tables `filtered_model`, `profil_user`, `dispo`, `user` , `alert`, `alert_message`
* Interroge l'active directory , le fichier mbtile,  le SIG et l'extranet
* Rempli les tables `file` ( lors de l'upload d'un fichier) , `line` (lors d'un dessin de Sitac)
* Enrichi la table `user` (lors des connexions)

**pusher** Chargé de pousser les modifications serveur vers le client à l'aide de socket.io

* Exposé par NGINX
* Ecoute les modifications sur les tables `audit`,`profil_filtered_model`, `profil_user`, `filtered_model`

 
**monitor** Chargé de donner des information sur les autres services

###Frontal
Stunnel est chargé de prendre le trafic https, de le décoder et de le transmettre a Nginx sous forme de trafix http.

Nginx est chargé de prendre le trafic http et de le transmettre au service concerne, qu'il soit sur le serveur courant ou sur une autre machine. Il s'occupe aussi de la compression gzip, et de certains header de cache. 

Nginx sert aussi les fichiers statiques du client: 

* `/` sert le client en version  **prod**
* `/dist` sert le client en version **preprod** (packagé, mais pas validé)
* `/app` sert le client version **dev**, non packagé.

##Todo
 
* porter le format mbtile vers postgresql pour diminuer les prerequis
* refactor /node/wms_search pour ne plus dépendre du'n serveur tiers, et rester en wgs84

