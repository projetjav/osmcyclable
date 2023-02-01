# osmcyclable
Projet étude OSM

Important : le fichier d'étude a été fractionné en 10 parties car GitHub refuse les fichiers de plus de 25 (!) Mo. Après avoir téléchargé toutes les parties, n'importe quel archiveur (WinRAR, 7Zip au moins) pourra réassembler le fichier. 

=========================================================================

Prérequis :

-Avoir une base de données créée et exploitable dans PostgreSQL

-Avoir installé les extensions hstore et PostGIS

=========================================================================

Commande osm2pgsql nécessaire pour traiter le fichier :

```
osm2pgsql.exe --host [hôte, habituellement 127.0.0.1] --port [port, habituellement 5432] --username [nom postgres] –-password --database [nom bdd] --slim --latlong --hstore --style "C:\Chemin\d'acces\vers\default.style" --input-reader pbf "C:\Chemin\d’acces\avec_antislashs\fichier.osm.pbf"
```

L’invite de commandes demandera ensuite d’entrer le mot de passe ; après le mot de passe entré, la lecture du fichier devrait se lancer.

Les tables démarrant par « planet » sont générées automatiquement par osm2pgsql lors de la lecture du fichier de données. Voici une explication rapide de chaque table :
-	planet_osm_line : chaque « ligne » géométrique, que ce soit des routes, câbles, ou étendues d’eaux, est représentée ici. L’identifiant OSM est donné en premier, puis une liste de tags associés à l’objet. La colonne « name » permet de connaitre le nom assigné à l’objet, la colonne « tags » les tags qui n’ont pas eu leur propre colonne, et la colonne « way » donne la géométrie de l’objet. Important : cette table donne aussi des lignes en dehors du Centre.
-	planet_osm_nodes : chaque point géométrique est représenté ici. Un objet sera généralement constitué de plusieurs points. Cette table donne l’identifiant ainsi que la position (latitude et longitude) de chaque point. Important : le nombre de points est très élevé (plus de 20 millions) ce qui rend toute requête dans cette table très demandante en ressources.
-	planet_osm_point : cette table indique les points qui représentent un objet à eux seuls. En d’autres termes, les objets définis par un seul point, comme les pylônes. Cette table fonctionne comme la table « line » : l’identifiant est donné, puis une liste de colonnes avec les tags associés, puis une colonne « tags » avec les tags restants et une colonne « way » pour la géométrie. Important : cette table donne aussi des points en dehors du Centre.
-	planet_osm_polygon : les polygones sont représentés dans cette table de la même manière que les points et les lignes (identifiants, colonnes de tags, « tags » supplémentaires, « way » géométrique).
-	planet_osm_rels : les relations, objets composés de plusieurs « ways », sont représentées ici. La colonne « parts » donne les différentes « ways » qui composent la relation avec leurs identifiants ; la colonne « members » donne plus d’informations. Cette table comporte aussi une colonne « tags » permettant d’avoir les différentes informations associées aux relations.
-	planet_osm_roads : cette table donne les différentes lignes utilisables pour un rendu géographique - c'est à dire principalement les routes, mais aussi les étendues d'eau (rivières, fleuves) dans la région Centre. La première colonne donne leur identifiant, puis les suivantes donnent les tags associés. On notera la colonne « bicycle » ; cependant le terme « cycleway » peut aussi être retrouvé dans la colonne « tags ».
-	planet_osm_ways : chaque « way » est représentée ici avec trois colonnes : l’identifiant, les nodes qui composent la way, et les tags associés à la way.

=========================================================================

Les tables démarrant par « osm » ont été créées par moi pour mieux me servir des données et répondre aux attentes du projet. L’ordre de création des tables est dans l’ordre du document.
Voici une liste des tables utilisées :

Table osm_bicycle_rels :
Cette table donne les identifiants des relations, le nombre de ways qui les composent, et les identifiants de ces ways. Seules les relations définies comme « cyclables » sont répertoriées.
Attention : un petit nombre de relations sont définies comme « bicycle,no » dans les tags. Il n’est pas possible de les enlever en demandant « EXCEPT ‘bicycle,no’ » car les deux mots sont considérés comme deux tags différents. De même, enlever « EXCEPT ‘bicycle’ = ANY… AND ‘no’ = ANY… » supprimerait toutes les relations qui contiennent un tag « bicycle » et un tag « no », où qu’il soit et pas forcément lié.

```
CREATE TABLE osm_bicycle_rels AS
select id, array_length(parts, 1) as nb_ways, parts, members, tags from planet_osm_rels
WHERE 'bicycle' = ANY(tags)
```

=========================================================================

Table osm_bicycle_roads_centre : 
Cette table donne toutes les routes du Centre qui font partie d’une relation dite « cyclable ». 

```
CREATE TABLE osm_bicycle_roads_centre AS
select distinct planet_osm_roads.* from planet_osm_roads, osm_bicycle_rels
where planet_osm_roads.osm_id = ANY(osm_bicycle_rels.parts)
```

=========================================================================

Table osm_bicycle_rels_centre :
Cette table donne toutes les relations cyclables qui contiennent au moins une route située dans le Centre. En d’autres termes, tous les itinéraires cyclables situés au moins partiellement dans le Centre.

```
CREATE TABLE osm_bicycle_rels_centre AS
SELECT DISTINCT osm_bicycle_rels.* FROM osm_bicycle_roads_centre, osm_bicycle_rels
where osm_bicycle_roads_centre.osm_id = ANY(osm_bicycle_rels.parts)
```

=========================================================================

Table osm_way_rel_cyclable1 :
Cette table existe en prérequis pour la table qui suit. Elle donne les différentes ways qui composent une relation ainsi qu’une colonne qui indique si la relation est cyclable ou non.

```
CREATE TABLE osm_way_rel_cyclable1 AS
select distinct planet_osm_rels.parts as way, planet_osm_rels.id as relation, planet_osm_rels.tags, 
case
	WHEN planet_osm_rels.id = osm_bicycle_rels.id THEN 1
	END rel_cyclable
from planet_osm_rels, osm_bicycle_rels
```

=========================================================================

Table osm_way_rel_cyclable2 :
Liée à la table précédente, cette table décompose l’array des différentes ways qui composent une relation pour avoir une ligne par way. Cela permet de voir à quelle relation appartient chaque way, et si la relation est cyclable ou non.

```
CREATE TABLE osm_way_rel_cyclable2 AS
select unnest(way) as way2, relation, rel_cyclable from osm_way_rel_cyclable1 
```

=========================================================================

Les tables définies ci-dessous correspondent au modèle de données utilisé pour étudier OSM.

Table fait_partie :
Représente les différents tronçons (« way ») et l’itinéraire (« relation ») dont ils font partie. De plus, la troisième colonne indique si cette relation est considérée comme un itinéraire cyclable ou non.

```
CREATE TABLE fait_partie AS
select way2 as idTroncon, relation as idItineraire, rel_cyclable as rel_cyclable from osm_way_rel_cyclable2
```

=========================================================================

Table troncon : 
Représente les identifiants de tous les tronçons de route du centre depuis la table de routes générée par osm2pgsql. Cette table devait aussi contenir la valeur de pente (en pourcents) soit l’inclinaison de la route, mais ayant trop peu d’informations à ce sujet dans les tags des tronçons (moins de 1% des tronçons avaient une valeur de pente définie), l’idée a été abandonnée.

```
CREATE TABLE troncon AS
select distinct osm_id as idTroncon from planet_osm_roads

ALTER TABLE troncon ADD PRIMARY KEY (idTroncon);
```

=========================================================================

Table itineraire : 
Représente les identifiants des relations/itinéraires, avec leur nom (ici les tags sont affichés dans la colonne nom, car différents tags représentent les noms), et le « niveau » de l’itinéraire. Le niveau représente l’importance du réseau : local, régional, national ou international.

```
CREATE TABLE itineraire AS
select id as idItineraire, tags as nomItineraire, 
case
	WHEN 'lcn'=ANY(tags) THEN 'local'
	WHEN 'rcn'=ANY(tags) THEN 'regional'
	WHEN 'ncn'=ANY(tags) THEN 'national'
	WHEN 'icn'=ANY(tags) THEN 'international'
	END niveau
from osm_bicycle_rels_centre

ALTER TABLE itineraire ADD PRIMARY KEY (idItineraire);

ALTER TABLE itineraire
ADD CONSTRAINT check_niveau
CHECK (
	niveau in ('local', 'regional', 'national', 'international')
)
```

=========================================================================

Table itineraire_sansniveau :
Cette table était une table temporaire servant à avoir une table « itineraire » avant de découvrir comment représenter le niveau. Elle fonctionne de la même manière que la table « itineraire », mais sans la colonne de niveau.

```
CREATE TABLE itineraire_sansniveau AS
select id as idItineraire, tags as nomItineraire from osm_bicycle_rels_centre
```

=========================================================================

Table osm_ways_slope : 
Cette table était censée permettre l’étude des pentes définies pour les routes d’OSM. Cependant, vu le faible taux de complétude de la colonne, l’idée a été abandonnée.

```
CREATE TABLE osm_ways_slope AS
SELECT osm_id as way,
tags -> 'incline' AS slope
FROM planet_osm_roads
```

Pour compter les tags de pente :

```
SELECT slope, count(slope) from osm_ways_slope
group by slope
```
