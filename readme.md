#  Rapport TP Interopérabilité 
### P1207328 Titouan CHARY 

## Données numériques

Nous inspectons les colonnes de la table, il semble que les colonnes `H, TOT, ANNEE, DEGRE, VOL, PRIX`contiennent des données numériques. 

On fait une séléction afin d'identifier les potentiels problèmes sur les données 

```sql 
SELECT H, TOT, ANNEE, DEGRE, VOL, PRIX  FROM VINS;
```
| H | TOT | ANNEE | DEGRE | VOL | PRIX  |
|---|---|------|--------|-------|-------|
| 5 |  null |  null    | 0.75   | 16.75 |   null    |
| 1 | 1 |   null   | 12.00% | 0,75  |   null    |
| 1 | 4 |  null    | 12.00% | 0.75  | 6.05€ |
| 1 | 1 | 2005 | 12°    | 0.75  |   null    |
| null  | 2 | 2006 | 12.50% | 0.75  | 8.2€  |


On peut remarquer que les colonnes `degre`, `vol` et `prix` ont des caractères spéciaux qui empêche d'appliquer
le type NUMBER aux données. 

Afin de filter et nettoyer les données nous renommons les colonnes concernées :
 
```sql
ALTER TABLE VINS RENAME COLUMN VOL TO VOL_STRING;
ALTER TABLE VINS RENAME COLUMN DEGRE TO DEGRE_STRING;
ALTER TABLE VINS RENAME COLUMN PRIX TO PRIX_STRING;
```

Ensuite, nous modifions les valeurs de sorte à avoir le caractère `,` en tant que séparateur. 

```sql 
UPDATE VINS SET VINS.DEGRE_STRING = REPLACE(VINS.DEGRE_STRING, '.', ',');
UPDATE VINS SET VINS.VOL_STRING = REPLACE(VINS.VOL_STRING, '.', ',');
UPDATE VINS SET VINS.PRIX_STRING = REPLACE(VINS.PRIX_STRING, '.', ',');
```

Nous modifions maintenant les valeurs de sorte à supprimer tout caracètre n'étant pas un digit. 

```sql 
UPDATE VINS SET VINS.DEGRE_STRING = REGEXP_SUBSTR(VINS.DEGRE_STRING,'[+\-]?[0-9]*[,]?[0-9]*');
UPDATE VINS SET VINS.VOL_STRING = REGEXP_SUBSTR(VINS.VOL_STRING,'[+\-]?[0-9]*[,]?[0-9]*');
UPDATE VINS SET VINS.PRIX_STRING = REGEXP_SUBSTR(VINS.PRIX_STRING,'[+\-]?[0-9]*[,]?[0-9]*');
```

Nous créons maintenant 3 nouvelles colonnes et importons les données converties dedans, 
enfin nous supprimons les anciennes colonnes.

```sql 
ALTER TABLE VINS ADD PRIX NUMBER DEFAULT NULL  NULL;
ALTER TABLE VINS ADD VOL NUMBER DEFAULT NULL  NULL;
ALTER TABLE VINS ADD DEGRE NUMBER DEFAULT NULL  NULL;

UPDATE VINS SET PRIX = TO_NUMBER(PRIX_STRING);
UPDATE VINS SET VOL = TO_NUMBER(VOL_STRING);
UPDATE VINS SET DEGRE = TO_NUMBER(DEGRE_STRING);

ALTER TABLE VINS DROP COLUMN VOL_STRING;
ALTER TABLE VINS DROP COLUMN DEGRE_STRING;
ALTER TABLE VINS DROP COLUMN PRIX_STRING;
```
## Emplacement de stockage

Nous modifions la table VINS en y ajoutant deux colonnes NB_HAUT et NB_BAS, non nulles
avec pour valeur par défaut 0.

```sql
ALTER TABLE VINS ADD NB_HAUT NUMBER DEFAULT 0 NOT NULL;;
ALTER TABLE VINS ADD NB_BAS NUMBER DEFAULT 0 NOT NULL;;
```

Nous insérons les valeurs dans les colonnes associées : 
 - NB_HAUT est égale à H ou 0 si H est à null
 - NB_BAS est égale à TOT - NB_HAUT ou 0 si TOT est à null 

```sql
UPDATE VINS SET NB_HAUT = COALESCE(H,0);
UPDATE VINS SET NB_BAS = COALESCE(TOT-VINS.NB_HAUT,0);
```

## Regions 

L'appellation devrait déterminer la région, cela signifie que pour une appellation donnée, nous devrions avoir 
une et une seule région associée. Nous recherchons les anomalies.

```sql
SELECT DISTINCT REGION FROM VINS ORDER BY REGION;
```

| Region            | 
|-------------------| 
| Alsace            | 
| Anjou             | 
| Beaujolais        | 
| Bergerac          | 
| Bordeaux          | 
| Bourgogne         | 
| CDR               | 
| Champagne         | 
| Cote-du-rhne      | 
| Cote-du-Rhône     | 
| Côte-du-Rhône     | 
| Cotes de Provence | 
| Côtes de Provence | 
| Cotes-du-Rhone    | 
| Cotes-du-Rhône    | 
| Côtes-du-Rhône    | 
| Jurançon          | 
| Languedoc         | 
| Monbazillac       | 
| Val-de-Loire      | 


Nous corrigeons les anomalies en choisissant arbitrairement un nom parmis tout ceux utilisés. 
(**CDR, Côtes de Provence**)

```sql
UPDATE VINS SET VINS.REGION = REPLACE(VINS.REGION, 'Cote-du-rhne', 'CDR');
UPDATE VINS SET VINS.REGION = REPLACE(VINS.REGION, 'Cote-du-Rhône', 'CDR');
UPDATE VINS SET VINS.REGION = REPLACE(VINS.REGION, 'Côte-du-Rhône', 'CDR');
UPDATE VINS SET VINS.REGION = REPLACE(VINS.REGION, 'Cotes-du-Rhone', 'CDR');
UPDATE VINS SET VINS.REGION = REPLACE(VINS.REGION, 'Cotes-du-Rhône', 'CDR');
UPDATE VINS SET VINS.REGION = REPLACE(VINS.REGION, 'Côtes-du-Rhône', 'CDR');
UPDATE VINS SET VINS.REGION = REPLACE(VINS.REGION, 'Cotes de Provence','Côtes de Provence');
```

Nous cherchons les potentielles autres anomalies

```sql
SELECT DISTINCT a.APPELLATION, a.REGION, b.REGION
FROM VINS a JOIN VINS b on a.APPELLATION = b.APPELLATION AND (a.REGION <> b.REGION OR a.REGION is NULL );
```

|      Appellation                      |    Region         |      Region       |    
|----------------|------|-----------| 
| Pouilly Fuissé | null | Bourgogne | 
| Pouilly Fuissé | null | null      | 

Puis nous les corrigeons 

```sql 
UPDATE VINS SET VINS.REGION = 'Bourgogne' WHERE APPELLATION LIKE 'Pouilly Fuissé';
```

## Appellation manquante

Deux vins ayant un même domaine et une même cuvée (lorsqu'elle est connue) ont la même appellation, nous cherchons donc les anomalies. 

```sql
SELECT DISTINCT a.DOMAINE, b.DOMAINE, a.CUVEE, b.CUVEE, a.APPELLATION, b.APPELLATION
  FROM VINS a JOIN VINS b on a.DOMAINE = b.DOMAINE AND a.CUVEE = b.CUVEE AND (a.APPELLATION <> b.APPELLATION OR a.APPELLATION is NULL)
ORDER BY a.DOMAINE;
```

| a.Domaine       | b.Domaine       |  a.Cuvée                        |  b.Cuvée                        |  a.Appellation |  b.Appellation                         | 
|-----------------|-----------------|---------------------------------|---------------------------------|----------------|---------------------------------------| 
| Bracoud         | Bracoud         | Merlot                          | Merlot                          |                | Vin de pays des collines Rhodaniennes | 
| Bracoud         | Bracoud         | Merlot                          | Merlot                          |                |                                       | 
| Chirat          | Chirat          | Nouvel'R (chardonnay)           | Nouvel'R (chardonnay)           |                |                                       | 
| Chirat          | Chirat          | Nouvel'R (chardonnay)           | Nouvel'R (chardonnay)           |                | Vin de pays des collines Rhodaniennes | 
| Domaine Daridan | Domaine Daridan | Vieilles vignes - fût de chênes | Vieilles vignes - fût de chênes |                |                                       | 
| Domaine Daridan | Domaine Daridan | Vieilles vignes - fût de chênes | Vieilles vignes - fût de chênes |                | Cour Cheverny                         | 


Enfin nous corrigeons les anomalies.  

```sql
UPDATE VINS SET VINS.APPELLATION = 'Vin de pays des collines Rhodaniennes' WHERE DOMAINE = 'Bracoud' AND CUVEE = 'Merlot';
UPDATE VINS SET VINS.APPELLATION = 'Vin de pays des collines Rhodaniennes' WHERE DOMAINE = 'Chirat' AND CUVEE = 'Nouvel''R (chardonnay)';
UPDATE VINS SET VINS.APPELLATION = 'Cour Cheverny' WHERE DOMAINE = 'Domaine Daridan' AND CUVEE = 'Vieilles vignes - fût de chênes';
```

## Données dupliquées

Afin d'identifier les lignes dupliquées, le plus simple est de faire un SELECT sur l'ensemble des colonnes ainsi qu'un GROUP BY 
sur ces mêmes colonnes et de ne filter que celles dont le count(*) est supérieur à 1 : 

```sql 
SELECT  LIEU, H, TOT, COULEUR, REGION, APPELLATION, ANNEE, DOMAINE, CUVEE, COMMENTAIRES, PRIX, DEGRE, VOL, NB_HAUT, NB_BAS
FROM VINS
GROUP BY LIEU, H, TOT, COULEUR, REGION, APPELLATION, ANNEE, DOMAINE, CUVEE, COMMENTAIRES, PRIX, DEGRE, VOL, NB_HAUT, NB_BAS
HAVING COUNT(*) > 1;
```

Cependant cette approche ne nous donne pas d'indications sur la clef primaire mais seulement sur les lignes dupliquées. 

- Les attributs `H, TOT, COMMENTAIRES, PRIX, DEGRE, VOL, NB_HAUT, NB_BAS`, ne sont évidemment pas membre de la clef primaire, car ils ne donnent pas d'indications
sur les bouteilles de vin ciblées.
- Nous savons de plus que APPELLATION → REGION
- Ainsi que CUVEE, DOMAINE → APPELLATION, or CUVEE peut être null par moment, il faut donc préserver APPELLATION
- Les attributs restants sont `LIEU, DOMAINE, CUVEE, ANNEE, COULEUR, APPELLATION`
    - un vin pouvant être stocké dans deux lieux différents, `LIEU`doit faire partie de la clef. 
    - Le domaine, l'année et la cuvée, sont évidemment nécessaire car on ne peut pas les inférer grace aux autres colonnes
    - Enfin, pour un même domaine, cuvée, année et lieu, un vin peut avoir été produit en blanc ou en rouge, il faut donc que `couleur` fasse partie de la clef primaire

Afin de supprimer les lignes dupliquées concernées, on execute la requête suivante. 

```sql
DELETE VINS WHERE ROWID IN (
  SELECT ROWID FROM
    VINS
  WHERE
    (LIEU, COULEUR, APPELLATION, ANNEE, DOMAINE, CUVEE) IN (
      SELECT  LIEU, COULEUR, APPELLATION, ANNEE, DOMAINE, CUVEE
      FROM VINS
      GROUP BY LIEU, COULEUR, APPELLATION, ANNEE, DOMAINE, CUVEE
      HAVING COUNT(*) > 1
  )
);
```

Nous devrions mettre à jour la clef primaire, cependant la table contient des **cuvées null** ainsi que des **années null**

```sql
ALTER TABLE VINS ADD CONSTRAINT id_key PRIMARY KEY (LIEU, DOMAINE, COULEUR, CUVEE, ANNEE);
```

## Annexes

Script de nettoyage complet

```sql 
DROP TABLE VINS;

CREATE TABLE VINS AS SELECT * FROM TIW1DATA.VINS;

ALTER TABLE VINS RENAME COLUMN VOL TO VOL_STRING;
ALTER TABLE VINS RENAME COLUMN DEGRE TO DEGRE_STRING;
ALTER TABLE VINS RENAME COLUMN PRIX TO PRIX_STRING;

UPDATE VINS SET VINS.DEGRE_STRING = REPLACE(VINS.DEGRE_STRING, '.', ',');
UPDATE VINS SET VINS.VOL_STRING = REPLACE(VINS.VOL_STRING, '.', ',');
UPDATE VINS SET VINS.PRIX_STRING = REPLACE(VINS.PRIX_STRING, '.', ',');

UPDATE VINS SET VINS.DEGRE_STRING = REGEXP_SUBSTR(VINS.DEGRE_STRING,'[+\-]?[0-9]*[,]?[0-9]*');
UPDATE VINS SET VINS.VOL_STRING = REGEXP_SUBSTR(VINS.VOL_STRING,'[+\-]?[0-9]*[,]?[0-9]*');
UPDATE VINS SET VINS.PRIX_STRING = REGEXP_SUBSTR(VINS.PRIX_STRING,'[+\-]?[0-9]*[,]?[0-9]*');

ALTER TABLE VINS ADD PRIX NUMBER DEFAULT NULL  NULL;
ALTER TABLE VINS ADD VOL NUMBER DEFAULT NULL  NULL;
ALTER TABLE VINS ADD DEGRE NUMBER DEFAULT NULL  NULL;

UPDATE VINS SET PRIX = TO_NUMBER(PRIX_STRING);
UPDATE VINS SET VOL = TO_NUMBER(VOL_STRING);
UPDATE VINS SET DEGRE = TO_NUMBER(DEGRE_STRING);

ALTER TABLE VINS DROP COLUMN VOL_STRING;
ALTER TABLE VINS DROP COLUMN DEGRE_STRING;
ALTER TABLE VINS DROP COLUMN PRIX_STRING;

ALTER TABLE VINS ADD NB_HAUT NUMBER DEFAULT 0 NOT NULL;;
ALTER TABLE VINS ADD NB_BAS NUMBER DEFAULT 0 NOT NULL;;

UPDATE VINS SET NB_HAUT = COALESCE(H,0);
UPDATE VINS SET NB_BAS = COALESCE(TOT-VINS.NB_HAUT,0);

UPDATE VINS SET VINS.REGION = REPLACE(VINS.REGION, 'Cote-du-rhne', 'CDR');
UPDATE VINS SET VINS.REGION = REPLACE(VINS.REGION, 'Cote-du-Rhône', 'CDR');
UPDATE VINS SET VINS.REGION = REPLACE(VINS.REGION, 'Côte-du-Rhône', 'CDR');
UPDATE VINS SET VINS.REGION = REPLACE(VINS.REGION, 'Cotes-du-Rhone', 'CDR');
UPDATE VINS SET VINS.REGION = REPLACE(VINS.REGION, 'Cotes-du-Rhône', 'CDR');
UPDATE VINS SET VINS.REGION = REPLACE(VINS.REGION, 'Côtes-du-Rhône', 'CDR');
UPDATE VINS SET VINS.REGION = REPLACE(VINS.REGION, 'Cotes de Provence','Côtes de Provence');
UPDATE VINS SET VINS.REGION = 'Bourgogne' WHERE APPELLATION LIKE 'Pouilly Fuissé';

UPDATE VINS SET VINS.APPELLATION = 'Vin de pays des collines Rhodaniennes' WHERE DOMAINE = 'Bracoud' AND CUVEE = 'Merlot';
UPDATE VINS SET VINS.APPELLATION = 'Vin de pays des collines Rhodaniennes' WHERE DOMAINE = 'Chirat' AND CUVEE = 'Nouvel''R (chardonnay)';
UPDATE VINS SET VINS.APPELLATION = 'Cour Cheverny' WHERE DOMAINE = 'Domaine Daridan' AND CUVEE = 'Vieilles vignes - fût de chênes';

DELETE VINS WHERE ROWID IN (
  SELECT ROWID FROM
    VINS
  WHERE
    (LIEU, COULEUR, APPELLATION, ANNEE, DOMAINE, CUVEE) IN (
      SELECT  LIEU, COULEUR, APPELLATION, ANNEE, DOMAINE, CUVEE
      FROM VINS
      GROUP BY LIEU, COULEUR, APPELLATION, ANNEE, DOMAINE, CUVEE
      HAVING COUNT(*) > 1
  )
);

```
