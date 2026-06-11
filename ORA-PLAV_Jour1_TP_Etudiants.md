# Oracle PL/SQL Avance: ORA-PLAV
## Guide de travaux pratiques: Jour 1

> **Formation** : ORA-PLAV: Oracle PL/SQL Avance
> **Niveau** : Developpeurs PL/SQL en activite
> **Environnement** : Oracle 21c XE · SQL Developer 24.x
> **Schema de travail** : SCOTT (tables `EMP` et `DEPT`)

---

## Avant de commencer: Configuration SQL Developer

Avant d'executer le moindre bloc PL/SQL, effectuez ces deux verifications.

### Activer l'affichage des resultats

**Option A: Methode recommandee (la plus simple)**

Ajoutez toujours cette ligne en tete de votre feuille de travail :

```sql
SET SERVEROUTPUT ON SIZE UNLIMITED
```

Executez ensuite votre code avec **F5** (pas F9). Les resultats apparaissent dans le panneau **Script Output** en bas.

**Option B: Panneau DBMS Output graphique**

1. Menu **View** → **Dbms Output**
2. Dans le panneau qui s'ouvre, cliquer sur le **+** vert
3. Selectionner votre connexion → **OK**
4. Executer avec **F5**

> Si rien ne s'affiche apres F5 : verifiez que `SET SERVEROUTPUT ON SIZE UNLIMITED` est bien presente en premiere ligne de votre script.

### Tester que tout fonctionne

Copiez et executez ce bloc avec **F5** :

```sql
SET SERVEROUTPUT ON SIZE UNLIMITED

BEGIN
  DBMS_OUTPUT.PUT_LINE('Configuration OK - pret pour ORA-PLAV !');
END;
/
```

**Resultat attendu dans Script Output** :
```
Configuration OK - pret pour ORA-PLAV !
```

Si vous voyez ce message, vous etes pret.

---

## Table des matieres

1. [Structure d'un bloc PL/SQL](#1--structure-dun-bloc-plsql)
2. [Types de donnees avances](#2--types-de-donnees-avances)
3. [%TYPE et %ROWTYPE](#3--type-et-rowtype)
4. [RECORD](#4--record)
5. [Collections](#5--collections)
6. [Large Objects: CLOB et BLOB](#6--large-objects--clob-et-blob)
7. [Curseurs et REF CURSOR](#7--curseurs-et-ref-cursor)
8. [BULK COLLECT et FORALL](#8--bulk-collect-et-forall)
9. [Gestion des erreurs](#9--gestion-des-erreurs)

---

## 1 · Structure d'un bloc PL/SQL

Un bloc PL/SQL suit toujours la meme structure. Seul `BEGIN...END` est obligatoire.

```
DECLARE   -- optionnel : vos variables, types, curseurs
BEGIN     -- obligatoire : vos instructions
EXCEPTION -- optionnel : vos gestionnaires d'erreur
END;
/
```

---

### Exemple 1-A : Le bloc minimal

```sql
SET SERVEROUTPUT ON SIZE UNLIMITED

BEGIN
  DBMS_OUTPUT.PUT_LINE('Mon premier bloc PL/SQL');
END;
/
```

**Resultat :**
```
Mon premier bloc PL/SQL
```

**Ce qu'il faut retenir :**
- Le `/` final est indispensable: il dit a SQL Developer d'executer le bloc
- Sans `SET SERVEROUTPUT ON`, rien ne s'affiche
- Executez toujours avec **F5**, pas **F9**

---

### Exemple 1-B : Declarer et utiliser des variables

```sql
SET SERVEROUTPUT ON SIZE UNLIMITED

DECLARE
  v_nom      VARCHAR2(50)  := 'Oracle PL/SQL';
  v_version  NUMBER(4,1)   := 21.0;
  v_date     DATE          := SYSDATE;
BEGIN
  DBMS_OUTPUT.PUT_LINE('Nom     : ' || v_nom);
  DBMS_OUTPUT.PUT_LINE('Version : ' || v_version);
  DBMS_OUTPUT.PUT_LINE('Date    : ' || TO_CHAR(v_date, 'DD/MM/YYYY'));

  -- Modifier une variable apres declaration
  v_nom := 'Oracle 21c XE';
  DBMS_OUTPUT.PUT_LINE('Nouveau nom : ' || v_nom);
END;
/
```

**Resultat :**
```
Nom     : Oracle PL/SQL
Version : 21
Date    : 11/06/2025
Nouveau nom : Oracle 21c XE
```

**Ce qu'il faut retenir :**
- L'affectation utilise `:=` (pas `=`)
- `||` concatene des chaines de caracteres
- Les variables se declarent dans `DECLARE`, avant `BEGIN`

---

### Exemple 1-C : La section EXCEPTION: intercepter une erreur

Executez d'abord ce bloc **sans** EXCEPTION pour voir ce qui se passe :

```sql
SET SERVEROUTPUT ON SIZE UNLIMITED

BEGIN
  DBMS_OUTPUT.PUT_LINE('Avant l erreur');
  DBMS_OUTPUT.PUT_LINE(10 / 0);   -- division par zero
  DBMS_OUTPUT.PUT_LINE('Cette ligne ne s affiche jamais');
END;
/
```

**Resultat (erreur) :**
```
Avant l erreur
ORA-01476: diviseur egal a zero
```

Maintenant **avec** la section EXCEPTION :

```sql
SET SERVEROUTPUT ON SIZE UNLIMITED

BEGIN
  DBMS_OUTPUT.PUT_LINE('Avant l erreur');

  DBMS_OUTPUT.PUT_LINE(10 / 0);

EXCEPTION
  WHEN ZERO_DIVIDE THEN
    DBMS_OUTPUT.PUT_LINE('Erreur interceptee : division par zero');
    DBMS_OUTPUT.PUT_LINE('Code  : ' || SQLCODE);
    DBMS_OUTPUT.PUT_LINE('Msg   : ' || SQLERRM);
END;
/
```

**Resultat :**
```
Avant l erreur
Erreur interceptee : division par zero
Code  : -1476
Msg   : ORA-01476: diviseur egal a zero
```

**Ce qu'il faut retenir :**
- Quand une erreur se produit, Oracle saute directement a la section EXCEPTION
- `SQLCODE` retourne le code d'erreur Oracle (negatif)
- `SQLERRM` retourne le message complet
- Le code situe entre l'erreur et la fin de BEGIN ne s'execute jamais

---

### Exemple 1-D : Blocs imbriques: isoler les erreurs

```sql
SET SERVEROUTPUT ON SIZE UNLIMITED

DECLARE
  v_sal NUMBER := 0;
BEGIN
  DBMS_OUTPUT.PUT_LINE('=== Debut ===');

  -- Ce bloc imbrique gere son erreur localement
  BEGIN
    SELECT sal INTO v_sal FROM emp WHERE empno = 9999; -- n'existe pas
  EXCEPTION
    WHEN NO_DATA_FOUND THEN
      DBMS_OUTPUT.PUT_LINE('Employe 9999 absent, on utilise 0 par defaut');
      v_sal := 0;
  END;

  -- Le programme continue meme apres l'erreur du bloc imbrique
  DBMS_OUTPUT.PUT_LINE('Salaire utilise : ' || v_sal);
  DBMS_OUTPUT.PUT_LINE('=== Fin ===');
END;
/
```

**Resultat :**
```
=== Debut ===
Employe 9999 absent, on utilise 0 par defaut
Salaire utilise : 0
=== Fin ===
```

**Ce qu'il faut retenir :**
- L'imbrication de blocs confine l'erreur a son contexte local
- Le bloc parent continue son execution apres le bloc imbrique

---

## 2 · Types de donnees avances

### Exemple 2-A : DATE vs TIMESTAMP: voir la difference

```sql
SET SERVEROUTPUT ON SIZE UNLIMITED

DECLARE
  v_date DATE      := SYSDATE;
  v_ts   TIMESTAMP := SYSTIMESTAMP;
BEGIN
  -- DATE : precision a la seconde
  DBMS_OUTPUT.PUT_LINE('DATE      : ' || TO_CHAR(v_date, 'DD/MM/YYYY HH24:MI:SS'));

  -- TIMESTAMP : precision a la microseconde
  DBMS_OUTPUT.PUT_LINE('TIMESTAMP : ' || TO_CHAR(v_ts, 'DD/MM/YYYY HH24:MI:SS.FF6'));
END;
/
```

**Resultat (exemple) :**
```
DATE      : 11/06/2025 14:32:07
TIMESTAMP : 11/06/2025 14:32:07.412893
```

**Ce qu'il faut retenir :**
- `DATE` s'arrete a la seconde
- `TIMESTAMP` ajoute les fractions de seconde (ici 6 chiffres)
- Toujours utiliser `SYSTIMESTAMP` (pas `SYSDATE`) dans les tables d'audit

---

### Exemple 2-B : TIMESTAMP: mesurer la duree d'un traitement

```sql
SET SERVEROUTPUT ON SIZE UNLIMITED

DECLARE
  v_debut  TIMESTAMP(6);
  v_fin    TIMESTAMP(6);
  v_duree  INTERVAL DAY(0) TO SECOND(3);
BEGIN
  v_debut := SYSTIMESTAMP;

  -- Simulation d'un traitement
  FOR i IN 1..500000 LOOP
    NULL;
  END LOOP;

  v_fin   := SYSTIMESTAMP;
  v_duree := v_fin - v_debut;  -- soustraction de deux TIMESTAMP = INTERVAL

  DBMS_OUTPUT.PUT_LINE('Debut  : ' || TO_CHAR(v_debut, 'HH24:MI:SS.FF3'));
  DBMS_OUTPUT.PUT_LINE('Fin    : ' || TO_CHAR(v_fin,   'HH24:MI:SS.FF3'));
  DBMS_OUTPUT.PUT_LINE('Duree  : ' || v_duree);
  DBMS_OUTPUT.PUT_LINE('Sec    : ' || ROUND(EXTRACT(SECOND FROM v_duree), 3));
END;
/
```

**Resultat (exemple) :**
```
Debut  : 14:32:07.412
Fin    : 14:32:07.847
Duree  : +00 00:00:00.435
Sec    : 0.435
```

**Ce qu'il faut retenir :**
- La soustraction de deux `TIMESTAMP` donne directement un `INTERVAL`
- `EXTRACT(SECOND FROM v_duree)` extrait la partie secondes
- Pattern standard pour mesurer les performances d'un traitement

---

### Exemple 2-C : INTERVAL: calculer des delais

```sql
SET SERVEROUTPUT ON SIZE UNLIMITED

DECLARE
  v_commande DATE := SYSDATE;
  v_livraison DATE;
  v_garantie  DATE;
BEGIN
  -- Ajouter 3 jours et 12 heures a une date
  v_livraison := v_commande + INTERVAL '3 12:00:00' DAY TO SECOND;

  -- Ajouter 2 ans et 6 mois
  v_garantie  := v_commande + INTERVAL '2-6' YEAR TO MONTH;

  DBMS_OUTPUT.PUT_LINE('Commande   : ' || TO_CHAR(v_commande,  'DD/MM/YYYY'));
  DBMS_OUTPUT.PUT_LINE('Livraison  : ' || TO_CHAR(v_livraison, 'DD/MM/YYYY HH24:MI'));
  DBMS_OUTPUT.PUT_LINE('Garantie   : ' || TO_CHAR(v_garantie,  'DD/MM/YYYY'));

  -- Anciennete des employes
  DBMS_OUTPUT.PUT_LINE('');
  DBMS_OUTPUT.PUT_LINE('Anciennete des employes (dept 10) :');
  FOR r IN (SELECT ename, hiredate,
                   TRUNC(MONTHS_BETWEEN(SYSDATE, hiredate)/12) annees
              FROM emp WHERE deptno = 10 ORDER BY hiredate)
  LOOP
    DBMS_OUTPUT.PUT_LINE('  ' || RPAD(r.ename, 10) ||
      TO_CHAR(r.hiredate, 'DD/MM/YYYY') || ' -> ' || r.annees || ' ans');
  END LOOP;
END;
/
```

---

### Exemple 2-D : BOOLEAN et sa conversion pour SQL

```sql
SET SERVEROUTPUT ON SIZE UNLIMITED

DECLARE
  v_sal     NUMBER  := 4500;
  v_senior  BOOLEAN;
  v_flag    CHAR(1);
BEGIN
  -- BOOLEAN : resultat d'une expression
  v_senior := (v_sal > 3000);

  -- BOOLEAN ne peut pas aller dans une colonne SQL
  -- Il faut le convertir avec CASE
  v_flag := CASE WHEN v_senior THEN 'O' ELSE 'N' END;

  DBMS_OUTPUT.PUT_LINE('Salaire : ' || v_sal);
  DBMS_OUTPUT.PUT_LINE('Senior  : ' || v_flag);

  IF v_senior THEN
    DBMS_OUTPUT.PUT_LINE('Prime senior accordee');
  END IF;

  -- Les trois valeurs possibles d'un BOOLEAN
  DECLARE
    v_vrai  BOOLEAN := TRUE;
    v_faux  BOOLEAN := FALSE;
    v_nul   BOOLEAN;        -- NULL par defaut
  BEGIN
    DBMS_OUTPUT.PUT_LINE('');
    DBMS_OUTPUT.PUT_LINE('TRUE  : ' || CASE WHEN v_vrai  THEN 'vrai'
                                            WHEN NOT v_vrai THEN 'faux'
                                            ELSE 'null' END);
    DBMS_OUTPUT.PUT_LINE('FALSE : ' || CASE WHEN v_faux  THEN 'vrai'
                                            WHEN NOT v_faux THEN 'faux'
                                            ELSE 'null' END);
    DBMS_OUTPUT.PUT_LINE('NULL  : ' || CASE WHEN v_nul IS NULL THEN 'null' ELSE 'pas null' END);
  END;
END;
/
```

**Ce qu'il faut retenir :**
- `BOOLEAN` n'existe qu'en PL/SQL: il n'existe pas en SQL Oracle
- Pour stocker un booleen en table : utiliser `CHAR(1)` (`'O'`/`'N'`) ou `NUMBER(1)` (1/0)
- Un `BOOLEAN` non initialise vaut `NULL` (pas FALSE)

---

### Exemple 2-E : PLS_INTEGER: le type pour les compteurs

```sql
SET SERVEROUTPUT ON SIZE UNLIMITED

DECLARE
  v_debut_n  TIMESTAMP(6);
  v_debut_p  TIMESTAMP(6);
  v_fin      TIMESTAMP(6);

  v_count_n  NUMBER      := 0;    -- lent pour l'arithmetique
  v_count_p  PLS_INTEGER := 0;    -- rapide : arithmetique native CPU
BEGIN
  -- Mesurer avec NUMBER
  v_debut_n := SYSTIMESTAMP;
  FOR i IN 1..1000000 LOOP
    v_count_n := v_count_n + 1;
  END LOOP;
  v_fin := SYSTIMESTAMP;
  DBMS_OUTPUT.PUT_LINE('NUMBER      : ' || (v_fin - v_debut_n));

  -- Mesurer avec PLS_INTEGER
  v_debut_p := SYSTIMESTAMP;
  FOR i IN 1..1000000 LOOP
    v_count_p := v_count_p + 1;
  END LOOP;
  v_fin := SYSTIMESTAMP;
  DBMS_OUTPUT.PUT_LINE('PLS_INTEGER : ' || (v_fin - v_debut_p));

  DBMS_OUTPUT.PUT_LINE('');
  DBMS_OUTPUT.PUT_LINE('Regle : toujours utiliser PLS_INTEGER pour les');
  DBMS_OUTPUT.PUT_LINE('compteurs de boucle et les indices de collection.');
END;
/
```

---

## 3 · %TYPE et %ROWTYPE

### Exemple 3-A : Le probleme sans %TYPE

```sql
SET SERVEROUTPUT ON SIZE UNLIMITED

-- VERSION FRAGILE : type declare en dur
DECLARE
  v_ename_fragile VARCHAR2(10);  -- que se passe-t-il si la colonne passe a VARCHAR2(50) ?
BEGIN
  SELECT ename INTO v_ename_fragile FROM emp WHERE empno = 7839;
  DBMS_OUTPUT.PUT_LINE('(fragile) : ' || v_ename_fragile);
END;
/
```

```sql
SET SERVEROUTPUT ON SIZE UNLIMITED

-- VERSION ROBUSTE : type ancre sur la colonne
DECLARE
  v_ename  emp.ename%TYPE;    -- herite exactement du type de la colonne
  v_sal    emp.sal%TYPE;
  v_deptno emp.deptno%TYPE;
BEGIN
  SELECT ename, sal, deptno
    INTO v_ename, v_sal, v_deptno
    FROM emp WHERE empno = 7839;

  DBMS_OUTPUT.PUT_LINE('Nom    : ' || v_ename);
  DBMS_OUTPUT.PUT_LINE('Salaire: ' || v_sal);
  DBMS_OUTPUT.PUT_LINE('Dept   : ' || v_deptno);
END;
/
```

**Ce qu'il faut retenir :**
- `%TYPE` ancre la variable sur le type exact d'une colonne de table
- Si le DBA modifie le type de la colonne, votre code PL/SQL s'adapte automatiquement a la recompilation
- Standard absolu en production : bannir les declarations de type en dur

---

### Exemple 3-B : %ROWTYPE: charger toute une ligne

```sql
SET SERVEROUTPUT ON SIZE UNLIMITED

DECLARE
  -- Une seule variable qui contient TOUS les champs de EMP
  r_emp  emp%ROWTYPE;
BEGIN
  SELECT * INTO r_emp FROM emp WHERE empno = 7788;  -- SCOTT

  -- Acces aux champs par leur nom de colonne
  DBMS_OUTPUT.PUT_LINE('Nom      : ' || r_emp.ename);
  DBMS_OUTPUT.PUT_LINE('Poste    : ' || r_emp.job);
  DBMS_OUTPUT.PUT_LINE('Manager  : ' || NVL(TO_CHAR(r_emp.mgr), 'Aucun'));
  DBMS_OUTPUT.PUT_LINE('Salaire  : ' || r_emp.sal);
  DBMS_OUTPUT.PUT_LINE('Comm     : ' || NVL(TO_CHAR(r_emp.comm), '-'));
  DBMS_OUTPUT.PUT_LINE('Embauche : ' || TO_CHAR(r_emp.hiredate, 'DD/MM/YYYY'));
  DBMS_OUTPUT.PUT_LINE('Dept     : ' || r_emp.deptno);
END;
/
```

**Ce qu'il faut retenir :**
- `emp%ROWTYPE` cree une variable avec les memes champs que la table EMP
- Acces aux champs avec la notation pointee : `r_emp.ename`, `r_emp.sal`, etc.
- Si la table a 50 colonnes, le `%ROWTYPE` charge 50 champs: preferer un `RECORD` si vous n'en avez besoin que de quelques-uns

---

### Exemple 3-C : %ROWTYPE sur un curseur: plus leger

```sql
SET SERVEROUTPUT ON SIZE UNLIMITED

DECLARE
  -- Ce curseur ne selectionne que 3 colonnes (pas toutes les 8 de EMP)
  CURSOR c IS
    SELECT ename, sal, deptno FROM emp ORDER BY sal DESC;

  -- %ROWTYPE sur le curseur : seulement les 3 colonnes selectionnees
  r c%ROWTYPE;

BEGIN
  OPEN c;
  LOOP
    FETCH c INTO r;
    EXIT WHEN c%NOTFOUND;
    DBMS_OUTPUT.PUT_LINE(
      RPAD(r.ename, 10) || ' sal=' || LPAD(r.sal, 6) || ' dept=' || r.deptno
    );
  END LOOP;
  CLOSE c;
END;
/
```

**Ce qu'il faut retenir :**
- `c%ROWTYPE` ne contient que les colonnes selectionnees dans le curseur
- Beaucoup plus economique en memoire que `emp%ROWTYPE` si on n'a pas besoin de toutes les colonnes

---

## 4 · RECORD

Un `RECORD` est une structure personnalisee qui regroupe des champs de types differents.
C'est l'equivalent d'un `struct` en C, ou d'une classe sans methodes en Java.

### Exemple 4-A : Declarer et utiliser un RECORD

```sql
SET SERVEROUTPUT ON SIZE UNLIMITED

DECLARE
  -- Definition du type RECORD
  TYPE t_personne IS RECORD (
    nom         VARCHAR2(50),
    age         PLS_INTEGER,
    salaire     NUMBER(10,2),
    actif       BOOLEAN,
    date_entree DATE DEFAULT SYSDATE  -- valeur par defaut possible
  );

  -- Variable de ce type
  v_p  t_personne;

BEGIN
  -- Remplir les champs
  v_p.nom     := 'Dupont';
  v_p.age     := 35;
  v_p.salaire := 3500.00;
  v_p.actif   := TRUE;
  -- v_p.date_entree est deja SYSDATE (valeur par defaut)

  -- Lire les champs
  DBMS_OUTPUT.PUT_LINE('Nom     : ' || v_p.nom);
  DBMS_OUTPUT.PUT_LINE('Age     : ' || v_p.age);
  DBMS_OUTPUT.PUT_LINE('Salaire : ' || v_p.salaire);
  DBMS_OUTPUT.PUT_LINE('Actif   : ' || CASE WHEN v_p.actif THEN 'Oui' ELSE 'Non' END);
  DBMS_OUTPUT.PUT_LINE('Entre   : ' || TO_CHAR(v_p.date_entree, 'DD/MM/YYYY'));
END;
/
```

---

### Exemple 4-B : RECORD avec %TYPE: combinaison ideale

```sql
SET SERVEROUTPUT ON SIZE UNLIMITED

DECLARE
  -- RECORD dont les types sont ancres sur le schema
  -- Utile quand on veut combiner des colonnes de plusieurs tables
  TYPE t_fiche IS RECORD (
    nom         emp.ename%TYPE,
    poste       emp.job%TYPE,
    salaire     emp.sal%TYPE,
    departement dept.dname%TYPE,
    ville       dept.loc%TYPE
  );

  v_fiche t_fiche;

BEGIN
  -- Remplir via un SELECT avec jointure
  SELECT e.ename, e.job, e.sal, d.dname, d.loc
    INTO v_fiche.nom, v_fiche.poste, v_fiche.salaire,
         v_fiche.departement, v_fiche.ville
    FROM emp  e
    JOIN dept d ON d.deptno = e.deptno
   WHERE e.empno = 7698;  -- BLAKE

  -- Afficher
  DBMS_OUTPUT.PUT_LINE('Employe  : ' || v_fiche.nom);
  DBMS_OUTPUT.PUT_LINE('Poste    : ' || v_fiche.poste);
  DBMS_OUTPUT.PUT_LINE('Salaire  : ' || v_fiche.salaire);
  DBMS_OUTPUT.PUT_LINE('Dept     : ' || v_fiche.departement);
  DBMS_OUTPUT.PUT_LINE('Ville    : ' || v_fiche.ville);
END;
/
```

**Ce qu'il faut retenir :**
- Un `RECORD` peut combiner des colonnes de **plusieurs tables**: ce que `%ROWTYPE` ne peut pas faire
- Toujours ancrer les champs avec `%TYPE` pour la robustesse
- En production : declarer les types `RECORD` dans un package partage (`pkg_types`) pour les reutiliser

---

### Exemple 4-C : RECORD comme parametre de procedure

```sql
SET SERVEROUTPUT ON SIZE UNLIMITED

DECLARE
  TYPE t_employe IS RECORD (
    ename   emp.ename%TYPE,
    job     emp.job%TYPE,
    sal     emp.sal%TYPE,
    deptno  emp.deptno%TYPE
  );

  -- Procedure qui recoit un RECORD en parametre
  PROCEDURE afficher_employe(p_emp IN t_employe) IS
  BEGIN
    DBMS_OUTPUT.PUT_LINE('--- Employe ---');
    DBMS_OUTPUT.PUT_LINE('Nom    : ' || p_emp.ename);
    DBMS_OUTPUT.PUT_LINE('Poste  : ' || p_emp.job);
    DBMS_OUTPUT.PUT_LINE('Sal    : ' || p_emp.sal);
    DBMS_OUTPUT.PUT_LINE('Dept   : ' || p_emp.deptno);
    IF p_emp.sal < 1500 THEN
      DBMS_OUTPUT.PUT_LINE('/!\ Salaire sous le minimum');
    END IF;
  END;

  v_emp t_employe;

BEGIN
  v_emp.ename  := 'MARTIN';
  v_emp.job    := 'ANALYST';
  v_emp.sal    := 3200;
  v_emp.deptno := 20;

  afficher_employe(v_emp);
END;
/
```

**Ce qu'il faut retenir :**
- Passer un `RECORD` en parametre remplace une liste de 4 ou 5 parametres individuels
- Si on ajoute un champ au `RECORD`, on ne modifie que sa declaration: pas toutes les signatures de procedures

---

## 5 · Collections

PL/SQL propose trois types de collections. Chacun a son cas d'usage precis.

| Type | Taille | Stockable en DB | Cle |
|---|---|---|---|
| `TABLE OF` (Nested Table) | Dynamique, illimitee | Oui | Entier auto |
| `VARRAY(n) OF` | Fixe (max n) | Oui | Entier auto |
| `TABLE OF ... INDEX BY` (Associative Array) | Dynamique | Non | Entier ou VARCHAR2 |

---

### Exemple 5-A : Nested Table: operations de base

```sql
SET SERVEROUTPUT ON SIZE UNLIMITED

DECLARE
  TYPE t_noms IS TABLE OF VARCHAR2(30);

  -- Initialisation avec des valeurs (constructeur du type)
  v_noms t_noms := t_noms('Alice', 'Bob', 'Charlie', 'Diana', 'Eve');

BEGIN
  DBMS_OUTPUT.PUT_LINE('COUNT  : ' || v_noms.COUNT);
  DBMS_OUTPUT.PUT_LINE('FIRST  : ' || v_noms.FIRST);
  DBMS_OUTPUT.PUT_LINE('LAST   : ' || v_noms.LAST);
  DBMS_OUTPUT.PUT_LINE('');

  -- Parcours
  FOR i IN v_noms.FIRST..v_noms.LAST LOOP
    DBMS_OUTPUT.PUT_LINE('  [' || i || '] ' || v_noms(i));
  END LOOP;

  -- Ajouter un element
  v_noms.EXTEND;
  v_noms(v_noms.LAST) := 'Frank';
  DBMS_OUTPUT.PUT_LINE('');
  DBMS_OUTPUT.PUT_LINE('Apres EXTEND : COUNT = ' || v_noms.COUNT);

  -- Supprimer un element (cree un "trou")
  v_noms.DELETE(2);  -- 'Bob' supprime
  DBMS_OUTPUT.PUT_LINE('Apres DELETE(2) : COUNT = ' || v_noms.COUNT);
  DBMS_OUTPUT.PUT_LINE('EXISTS(2) = ' || CASE WHEN v_noms.EXISTS(2) THEN 'TRUE' ELSE 'FALSE' END);
END;
/
```

> **Attention aux trous** : apres `DELETE(2)`, l'indice 2 n'existe plus. Une boucle `FOR i IN 1..COUNT` planterait. Utiliser `FIRST`/`NEXT` pour parcourir en securite apres des suppressions.

---

### Exemple 5-B : Parcours securise avec FIRST / NEXT

```sql
SET SERVEROUTPUT ON SIZE UNLIMITED

DECLARE
  TYPE t_noms IS TABLE OF VARCHAR2(30);
  v_noms t_noms := t_noms('Alice', 'Bob', 'Charlie', 'Diana', 'Eve');
  v_i    PLS_INTEGER;

BEGIN
  -- Creer un trou
  v_noms.DELETE(2);
  v_noms.DELETE(4);

  DBMS_OUTPUT.PUT_LINE('Parcours avec FIRST/NEXT (ignore les trous) :');
  v_i := v_noms.FIRST;
  WHILE v_i IS NOT NULL LOOP
    DBMS_OUTPUT.PUT_LINE('  [' || v_i || '] ' || v_noms(v_i));
    v_i := v_noms.NEXT(v_i);  -- passe automatiquement au prochain indice existant
  END LOOP;
END;
/
```

---

### Exemple 5-C : VARRAY: liste ordonnee a taille fixe

```sql
SET SERVEROUTPUT ON SIZE UNLIMITED

DECLARE
  -- Maximum 7 elements
  TYPE t_jours IS VARRAY(7) OF VARCHAR2(15);

  v_semaine t_jours := t_jours('Lundi', 'Mardi', 'Mercredi', 'Jeudi', 'Vendredi', 'Samedi', 'Dimanche');

BEGIN
  DBMS_OUTPUT.PUT_LINE('Jours de la semaine :');
  FOR i IN 1..v_semaine.COUNT LOOP
    DBMS_OUTPUT.PUT_LINE('  ' || i || '. ' || v_semaine(i));
  END LOOP;

  DBMS_OUTPUT.PUT_LINE('');
  DBMS_OUTPUT.PUT_LINE('Limite max : ' || v_semaine.LIMIT);
  DBMS_OUTPUT.PUT_LINE('Count actuel : ' || v_semaine.COUNT);
END;
/
```

---

### Exemple 5-D : Associative Array: cache de lookup

C'est le cas d'usage le plus important en production : eviter des SELECT repetes sur une table de reference.

```sql
SET SERVEROUTPUT ON SIZE UNLIMITED

-- VERSION SANS cache : 1 SELECT sur DEPT par employe
DECLARE
  v_dname dept.dname%TYPE;
BEGIN
  DBMS_OUTPUT.PUT_LINE('=== Sans cache : 1 SELECT/employe ===');
  FOR r IN (SELECT ename, deptno FROM emp ORDER BY ename) LOOP
    SELECT dname INTO v_dname FROM dept WHERE deptno = r.deptno;
    DBMS_OUTPUT.PUT_LINE(RPAD(r.ename, 12) || ' -> ' || v_dname);
  END LOOP;
  DBMS_OUTPUT.PUT_LINE('=> ' || SQL%ROWCOUNT || ' SELECT sur DEPT');
END;
/
```

```sql
SET SERVEROUTPUT ON SIZE UNLIMITED

-- VERSION AVEC cache : 1 seul SELECT sur DEPT pour tous les employes
DECLARE
  TYPE t_cache IS TABLE OF dept.dname%TYPE INDEX BY PLS_INTEGER;
  v_cache t_cache;

BEGIN
  DBMS_OUTPUT.PUT_LINE('=== Avec cache Associative Array ===');

  -- Charger le referentiel une seule fois
  FOR r IN (SELECT deptno, dname FROM dept) LOOP
    v_cache(r.deptno) := r.dname;
  END LOOP;
  DBMS_OUTPUT.PUT_LINE('Cache charge (1 SELECT sur DEPT)');
  DBMS_OUTPUT.PUT_LINE('');

  -- Lookup en memoire pour chaque employe : 0 SELECT supplementaire
  FOR r IN (SELECT ename, deptno FROM emp ORDER BY ename) LOOP
    DBMS_OUTPUT.PUT_LINE(RPAD(r.ename, 12) || ' -> ' || v_cache(r.deptno));
  END LOOP;

  DBMS_OUTPUT.PUT_LINE('');
  DBMS_OUTPUT.PUT_LINE('=> 1 seul SELECT sur DEPT au total (au lieu de 14)');
END;
/
```

**Ce qu'il faut retenir :**
- L'Associative Array `INDEX BY PLS_INTEGER` fonctionne comme un dictionnaire en memoire
- Le lookup est tres rapide (O log n): aucun acces disque
- Sur des millions de lignes, cette difference est determinante

---

### Exemple 5-E : Associative Array avec cle VARCHAR2

```sql
SET SERVEROUTPUT ON SIZE UNLIMITED

DECLARE
  -- Cle = code pays (chaine), valeur = libelle
  TYPE t_pays IS TABLE OF VARCHAR2(50) INDEX BY VARCHAR2(3);
  v_pays t_pays;
  v_code VARCHAR2(3);

BEGIN
  v_pays('FR') := 'France';
  v_pays('DE') := 'Allemagne';
  v_pays('ES') := 'Espagne';
  v_pays('US') := 'Etats-Unis';

  -- Lookup direct par cle chaine
  DBMS_OUTPUT.PUT_LINE('FR : ' || v_pays('FR'));
  DBMS_OUTPUT.PUT_LINE('DE : ' || v_pays('DE'));

  -- Verifier l'existence avant d'acceder
  v_code := 'JP';
  IF v_pays.EXISTS(v_code) THEN
    DBMS_OUTPUT.PUT_LINE(v_code || ' : ' || v_pays(v_code));
  ELSE
    DBMS_OUTPUT.PUT_LINE(v_code || ' : code non reference');
  END IF;

  -- Parcourir toutes les cles
  DBMS_OUTPUT.PUT_LINE('');
  v_code := v_pays.FIRST;
  WHILE v_code IS NOT NULL LOOP
    DBMS_OUTPUT.PUT_LINE('  ' || v_code || ' -> ' || v_pays(v_code));
    v_code := v_pays.NEXT(v_code);
  END LOOP;
END;
/
```

---

## 6 · Large Objects: CLOB et BLOB

Les LOBs servent a stocker de grandes quantites de donnees : textes longs (CLOB), fichiers binaires (BLOB). Toute manipulation passe par le package `DBMS_LOB`.

### Exemple 6-A : Creer, ecrire et lire un CLOB temporaire

Les quatre etapes obligatoires : **CREATETEMPORARY → WRITEAPPEND → READ → FREETEMPORARY**

```sql
SET SERVEROUTPUT ON SIZE UNLIMITED

DECLARE
  v_clob  CLOB;
  v_ligne VARCHAR2(200);
BEGIN
  -- Etape 1 : creer le LOB en memoire
  DBMS_LOB.CREATETEMPORARY(v_clob, TRUE);
  DBMS_OUTPUT.PUT_LINE('LOB cree. Taille : ' || DBMS_LOB.GETLENGTH(v_clob));

  -- Etape 2 : ecrire du contenu avec WRITEAPPEND
  DBMS_LOB.WRITEAPPEND(v_clob, 25, 'Debut du rapport Oracle.' || CHR(10));

  FOR r IN (SELECT ename, job, sal FROM emp ORDER BY sal DESC) LOOP
    v_ligne := RPAD(r.ename, 10) || RPAD(r.job, 12) || r.sal || CHR(10);
    DBMS_LOB.WRITEAPPEND(v_clob, LENGTH(v_ligne), v_ligne);
  END LOOP;

  DBMS_OUTPUT.PUT_LINE('Taille apres ecriture : ' || DBMS_LOB.GETLENGTH(v_clob));

  -- Etape 3 : lire les 200 premiers caracteres
  DBMS_OUTPUT.PUT_LINE('');
  DBMS_OUTPUT.PUT_LINE('Contenu (debut) :');
  DBMS_OUTPUT.PUT_LINE(DBMS_LOB.SUBSTR(v_clob, 200, 1));

  -- Etape 4 : OBLIGATOIRE: liberer la memoire
  DBMS_LOB.FREETEMPORARY(v_clob);
  DBMS_OUTPUT.PUT_LINE('LOB libere.');
END;
/
```

---

### Exemple 6-B : Lire un grand CLOB par morceaux (chunks)

Pour un LOB de grande taille, on ne peut pas tout lire d'un coup dans une variable `VARCHAR2`. Il faut lire par morceaux successifs.

```sql
SET SERVEROUTPUT ON SIZE UNLIMITED

DECLARE
  v_clob   CLOB;
  v_ligne  VARCHAR2(200);

  -- Variables de lecture par chunks
  v_offset INTEGER := 1;
  v_amount INTEGER;
  v_buffer VARCHAR2(255);
  v_len    INTEGER;

BEGIN
  DBMS_LOB.CREATETEMPORARY(v_clob, TRUE);

  -- Construire un CLOB avec toutes les donnees EMP
  DBMS_LOB.WRITEAPPEND(v_clob, 30, '=== RAPPORT COMPLET ===' || CHR(10));
  FOR r IN (SELECT deptno, ename, sal FROM emp ORDER BY deptno, sal DESC) LOOP
    v_ligne := 'Dept ' || r.deptno || ' | ' || RPAD(r.ename,10) || ' | ' || r.sal || CHR(10);
    DBMS_LOB.WRITEAPPEND(v_clob, LENGTH(v_ligne), v_ligne);
  END LOOP;

  v_len := DBMS_LOB.GETLENGTH(v_clob);
  DBMS_OUTPUT.PUT_LINE('Taille : ' || v_len || ' caracteres');
  DBMS_OUTPUT.PUT_LINE('--- Lecture par chunks de 255 ---');

  -- Lire le CLOB entier par morceaux de 255 caracteres
  WHILE v_offset <= v_len LOOP
    v_amount := LEAST(255, v_len - v_offset + 1);
    DBMS_LOB.READ(v_clob, v_amount, v_offset, v_buffer);
    DBMS_OUTPUT.PUT(v_buffer);
    v_offset := v_offset + v_amount;
  END LOOP;

  DBMS_LOB.FREETEMPORARY(v_clob);
END;
/
```

---

### Exemple 6-C : FREETEMPORARY dans EXCEPTION: la regle absolue

```sql
SET SERVEROUTPUT ON SIZE UNLIMITED

-- VERSION INCORRECTE : si une erreur se produit, FREETEMPORARY n'est jamais appele
-- -> Fuite memoire dans le tablespace TEMP jusqu'a la fin de la session
DECLARE
  v_clob CLOB;
BEGIN
  DBMS_LOB.CREATETEMPORARY(v_clob, TRUE);
  DBMS_LOB.WRITEAPPEND(v_clob, 5, 'test');
  RAISE_APPLICATION_ERROR(-20001, 'Erreur simulee');
  DBMS_LOB.FREETEMPORARY(v_clob);  -- jamais atteinte !
EXCEPTION
  WHEN OTHERS THEN
    DBMS_OUTPUT.PUT_LINE('INCORRECTE : LOB non libere -> fuite memoire');
END;
/
```

```sql
SET SERVEROUTPUT ON SIZE UNLIMITED

-- VERSION CORRECTE : FREETEMPORARY aussi dans EXCEPTION
DECLARE
  v_clob CLOB;
BEGIN
  DBMS_LOB.CREATETEMPORARY(v_clob, TRUE);
  DBMS_LOB.WRITEAPPEND(v_clob, 5, 'test');

  RAISE_APPLICATION_ERROR(-20001, 'Erreur simulee');

  -- Liberation normale (cas sans erreur)
  DBMS_LOB.FREETEMPORARY(v_clob);

EXCEPTION
  WHEN OTHERS THEN
    -- Liberation dans le handler : couvre tous les cas d'erreur
    IF DBMS_LOB.ISTEMPORARY(v_clob) = 1 THEN
      DBMS_LOB.FREETEMPORARY(v_clob);
      DBMS_OUTPUT.PUT_LINE('CORRECTE : LOB libere dans EXCEPTION');
    END IF;
    DBMS_OUTPUT.PUT_LINE('Erreur : ' || SQLERRM);
END;
/
```

**Regle absolue :**
- `CREATETEMPORARY` alloue de la memoire dans le tablespace TEMP
- `FREETEMPORARY` **doit** etre appele dans le flux normal ET dans la section EXCEPTION
- Sans cela, la memoire reste allouee jusqu'a la fin de la session

---

## 7 · Curseurs et REF CURSOR

### Exemple 7-A : Cycle de vie: DECLARE / OPEN / FETCH / CLOSE

```sql
SET SERVEROUTPUT ON SIZE UNLIMITED

DECLARE
  -- Etape 1 : DECLARE: definit la requete (non encore executee)
  CURSOR c_emp10 IS
    SELECT empno, ename, sal FROM emp WHERE deptno = 10 ORDER BY sal DESC;

  r c_emp10%ROWTYPE;

BEGIN
  -- Etape 2 : OPEN: execute la requete
  OPEN c_emp10;
  DBMS_OUTPUT.PUT_LINE('Curseur ouvert : ' || CASE WHEN c_emp10%ISOPEN THEN 'OUI' ELSE 'NON' END);

  -- Etape 3 : FETCH: lire ligne par ligne
  LOOP
    FETCH c_emp10 INTO r;
    EXIT WHEN c_emp10%NOTFOUND;
    DBMS_OUTPUT.PUT_LINE(
      'Ligne ' || c_emp10%ROWCOUNT || ' : ' || RPAD(r.ename, 10) || r.sal
    );
  END LOOP;

  DBMS_OUTPUT.PUT_LINE('Total lu : ' || c_emp10%ROWCOUNT);

  -- Etape 4 : CLOSE: liberer les ressources
  CLOSE c_emp10;

EXCEPTION
  WHEN OTHERS THEN
    IF c_emp10%ISOPEN THEN CLOSE c_emp10; END IF;  -- toujours fermer dans EXCEPTION
    RAISE;
END;
/
```

---

### Exemple 7-B : Les attributs du curseur

```sql
SET SERVEROUTPUT ON SIZE UNLIMITED

DECLARE
  CURSOR c IS SELECT ename, sal FROM emp WHERE deptno = 30 ORDER BY sal;
  r c%ROWTYPE;
BEGIN
  DBMS_OUTPUT.PUT_LINE('Avant OPEN :');
  DBMS_OUTPUT.PUT_LINE('  %ISOPEN = ' || CASE WHEN c%ISOPEN THEN 'TRUE' ELSE 'FALSE' END);

  OPEN c;
  DBMS_OUTPUT.PUT_LINE('Apres OPEN :');
  DBMS_OUTPUT.PUT_LINE('  %ISOPEN   = ' || CASE WHEN c%ISOPEN THEN 'TRUE' ELSE 'FALSE' END);
  DBMS_OUTPUT.PUT_LINE('  %ROWCOUNT = ' || c%ROWCOUNT);

  FETCH c INTO r;
  DBMS_OUTPUT.PUT_LINE('Apres 1er FETCH (' || r.ename || ') :');
  DBMS_OUTPUT.PUT_LINE('  %FOUND    = ' || CASE WHEN c%FOUND    THEN 'TRUE' ELSE 'FALSE' END);
  DBMS_OUTPUT.PUT_LINE('  %NOTFOUND = ' || CASE WHEN c%NOTFOUND THEN 'TRUE' ELSE 'FALSE' END);
  DBMS_OUTPUT.PUT_LINE('  %ROWCOUNT = ' || c%ROWCOUNT);

  -- Vider le reste
  LOOP FETCH c INTO r; EXIT WHEN c%NOTFOUND; END LOOP;

  DBMS_OUTPUT.PUT_LINE('Apres dernier FETCH :');
  DBMS_OUTPUT.PUT_LINE('  %FOUND    = ' || CASE WHEN c%FOUND    THEN 'TRUE' ELSE 'FALSE' END);
  DBMS_OUTPUT.PUT_LINE('  %NOTFOUND = ' || CASE WHEN c%NOTFOUND THEN 'TRUE' ELSE 'FALSE' END);
  DBMS_OUTPUT.PUT_LINE('  %ROWCOUNT = ' || c%ROWCOUNT);

  CLOSE c;
END;
/
```

---

### Exemple 7-C : Curseur avec parametres

```sql
SET SERVEROUTPUT ON SIZE UNLIMITED

DECLARE
  -- Curseur acceptant deux parametres (le second a une valeur par defaut)
  CURSOR c_dept (p_deptno NUMBER, p_sal_min NUMBER DEFAULT 0) IS
    SELECT ename, sal, job FROM emp
     WHERE deptno = p_deptno AND sal >= p_sal_min
     ORDER BY sal DESC;

  r c_dept%ROWTYPE;

  PROCEDURE afficher(p_dept NUMBER, p_seuil NUMBER DEFAULT 0) IS
  BEGIN
    DBMS_OUTPUT.PUT_LINE('--- Dept ' || p_dept || ' (sal >= ' || p_seuil || ') ---');
    OPEN c_dept(p_dept, p_seuil);
    LOOP
      FETCH c_dept INTO r;
      EXIT WHEN c_dept%NOTFOUND;
      DBMS_OUTPUT.PUT_LINE('  ' || RPAD(r.ename,10) || RPAD(r.job,12) || r.sal);
    END LOOP;
    DBMS_OUTPUT.PUT_LINE('  Total : ' || c_dept%ROWCOUNT || ' employe(s)');
    CLOSE c_dept;
  END;

BEGIN
  afficher(10);          -- dept 10, tous salaires
  afficher(30, 1500);    -- dept 30, sal >= 1500
END;
/
```

---

### Exemple 7-D : REF CURSOR: retourner un jeu de resultats

```sql
SET SERVEROUTPUT ON SIZE UNLIMITED

-- Procedure qui retourne un SYS_REFCURSOR
CREATE OR REPLACE PROCEDURE get_employes (
  p_deptno IN  NUMBER,
  p_cur    OUT SYS_REFCURSOR
) IS
BEGIN
  OPEN p_cur FOR
    SELECT e.empno, e.ename, e.job, e.sal, d.dname
      FROM emp  e
      JOIN dept d ON d.deptno = e.deptno
     WHERE e.deptno = p_deptno
     ORDER BY e.sal DESC;
  -- La procedure ouvre le curseur mais ne le ferme PAS
  -- c'est l'APPELANT qui est responsable du CLOSE
END;
/

-- Consommer le REF CURSOR
DECLARE
  v_cur    SYS_REFCURSOR;
  v_empno  emp.empno%TYPE;
  v_ename  emp.ename%TYPE;
  v_job    emp.job%TYPE;
  v_sal    emp.sal%TYPE;
  v_dname  dept.dname%TYPE;
BEGIN
  get_employes(20, v_cur);  -- p_cur est maintenant ouvert

  LOOP
    FETCH v_cur INTO v_empno, v_ename, v_job, v_sal, v_dname;
    EXIT WHEN v_cur%NOTFOUND;
    DBMS_OUTPUT.PUT_LINE(RPAD(v_ename,10) || RPAD(v_job,12) || LPAD(v_sal,6));
  END LOOP;

  CLOSE v_cur;  -- OBLIGATOIRE : l'appelant doit fermer
END;
/
```

**Ce qu'il faut retenir :**
- Le `SYS_REFCURSOR` est le pont entre PL/SQL et les applications Java, Python, .NET
- La procedure **ouvre** le curseur, l'appelant **ferme** le curseur
- `SYS_REFCURSOR` est faiblement type : Oracle ne verifie pas la structure a la compilation

---

## 8 · BULK COLLECT et FORALL

### Pourquoi BULK COLLECT ?

Chaque `FETCH` individuel provoque un **context switch** entre le moteur PL/SQL et le moteur SQL. Sur 100 000 lignes, c'est 100 000 allers-retours. `BULK COLLECT` regroupe ces allers-retours en lots: gain typique : **10x a 50x**.

---

### Exemple 8-A : BULK COLLECT avec LIMIT: le pattern de base

```sql
SET SERVEROUTPUT ON SIZE UNLIMITED

DECLARE
  TYPE t_emp IS TABLE OF emp%ROWTYPE;
  CURSOR c IS SELECT * FROM emp;

  v_emps  t_emp;
  v_total PLS_INTEGER := 0;
  v_lot   PLS_INTEGER := 0;

BEGIN
  OPEN c;
  LOOP
    -- Lire 5 lignes a la fois (en production : 500 ou 1000)
    FETCH c BULK COLLECT INTO v_emps LIMIT 5;
    EXIT WHEN v_emps.COUNT = 0;

    v_lot   := v_lot + 1;
    v_total := v_total + v_emps.COUNT;

    DBMS_OUTPUT.PUT_LINE('Lot ' || v_lot || ' : ' || v_emps.COUNT || ' lignes');
    FOR i IN 1..v_emps.COUNT LOOP
      DBMS_OUTPUT.PUT_LINE('  ' || v_emps(i).ename || ' sal=' || v_emps(i).sal);
    END LOOP;

    v_emps.DELETE;  -- liberer la PGA entre les lots
  END LOOP;
  CLOSE c;

  DBMS_OUTPUT.PUT_LINE('Total : ' || v_total || ' employes en ' || v_lot || ' lots');
END;
/
```

**Ce qu'il faut retenir :**
- `LIMIT n` est **obligatoire**: sans lui, toute la table est chargee en memoire d'un coup
- `v_emps.DELETE` entre les lots libere la PGA
- En production : `LIMIT 500` est un bon point de depart

---

### Exemple 8-B : FORALL: DML en masse

```sql
SET SERVEROUTPUT ON SIZE UNLIMITED

CREATE TABLE ti_sal_test AS SELECT empno, ename, sal, deptno FROM emp;
/

DECLARE
  TYPE t_empno IS TABLE OF NUMBER;
  v_ids t_empno := t_empno(7369, 7499, 7521, 7566, 7654, 7698, 7782, 7788);

  v_debut TIMESTAMP(6) := SYSTIMESTAMP;
BEGIN
  -- FORALL : 1 seul appel SQL pour tout le tableau
  FORALL i IN 1..v_ids.COUNT
    UPDATE ti_sal_test SET sal = sal * 1.10 WHERE empno = v_ids(i);

  DBMS_OUTPUT.PUT_LINE('Lignes mises a jour : ' || SQL%ROWCOUNT);
  DBMS_OUTPUT.PUT_LINE('Duree : ' || (SYSTIMESTAMP - v_debut));

  -- Detail par iteration
  DBMS_OUTPUT.PUT_LINE('');
  FOR i IN 1..v_ids.COUNT LOOP
    DBMS_OUTPUT.PUT_LINE('empno=' || v_ids(i) || ' -> ' || SQL%BULK_ROWCOUNT(i) || ' ligne(s)');
  END LOOP;

  ROLLBACK;
END;
/

DROP TABLE ti_sal_test;
/
```

---

### Exemple 8-C : FORALL INDICES OF: gerer les trous

```sql
SET SERVEROUTPUT ON SIZE UNLIMITED

CREATE TABLE ti_test (id NUMBER, valeur VARCHAR2(30));
/

DECLARE
  TYPE t_noms IS TABLE OF VARCHAR2(30);
  v_noms t_noms := t_noms('Alice', 'Bob', 'Charlie', 'Diana', 'Eve');

BEGIN
  v_noms.DELETE(2);  -- trou a l'indice 2
  v_noms.DELETE(4);  -- trou a l'indice 4
  DBMS_OUTPUT.PUT_LINE('Indices existants : 1, 3, 5');

  -- IN 1..COUNT echouerait sur les trous
  -- INDICES OF traite UNIQUEMENT les indices existants
  FORALL i IN INDICES OF v_noms
    INSERT INTO ti_test VALUES (i, v_noms(i));

  DBMS_OUTPUT.PUT_LINE('Insere : ' || SQL%ROWCOUNT || ' lignes');

  FOR r IN (SELECT * FROM ti_test ORDER BY id) LOOP
    DBMS_OUTPUT.PUT_LINE('  id=' || r.id || ' val=' || r.valeur);
  END LOOP;

  ROLLBACK;
END;
/

DROP TABLE ti_test;
/
```

---

### Exemple 8-D : FORALL SAVE EXCEPTIONS: continuer malgre les erreurs

```sql
SET SERVEROUTPUT ON SIZE UNLIMITED

CREATE TABLE ti_clients (id NUMBER PRIMARY KEY, nom VARCHAR2(50) NOT NULL);
INSERT INTO ti_clients VALUES (1, 'Dupont');
INSERT INTO ti_clients VALUES (2, 'Martin');
COMMIT;

CREATE TABLE ti_cmd (id NUMBER PRIMARY KEY, client_id NUMBER REFERENCES ti_clients(id), montant NUMBER NOT NULL);
/

DECLARE
  TYPE t_id  IS TABLE OF NUMBER;
  TYPE t_cli IS TABLE OF NUMBER;
  TYPE t_mnt IS TABLE OF NUMBER;

  -- Donnees avec erreurs intentionnelles
  -- cmd 3 -> client 99 (FK violation)
  -- cmd 5 -> montant NULL (NOT NULL violation)
  v_ids     t_id  := t_id (1,    2,    3,    4,    5   );
  v_clients t_cli := t_cli(1,    2,    99,   1,    2   );
  v_montants t_mnt := t_mnt(100, 200,  300,  400,  NULL);

  e_bulk EXCEPTION;
  PRAGMA EXCEPTION_INIT(e_bulk, -24381);

BEGIN
  DBMS_OUTPUT.PUT_LINE('Insertion de ' || v_ids.COUNT || ' commandes...');

  FORALL i IN 1..v_ids.COUNT SAVE EXCEPTIONS
    INSERT INTO ti_cmd VALUES (v_ids(i), v_clients(i), v_montants(i));

  DBMS_OUTPUT.PUT_LINE('Toutes reussies : ' || SQL%ROWCOUNT);

EXCEPTION
  WHEN e_bulk THEN
    DBMS_OUTPUT.PUT_LINE('Reussies : ' || (v_ids.COUNT - SQL%BULK_EXCEPTIONS.COUNT));
    DBMS_OUTPUT.PUT_LINE('Echecs   : ' || SQL%BULK_EXCEPTIONS.COUNT);
    DBMS_OUTPUT.PUT_LINE('');
    FOR j IN 1..SQL%BULK_EXCEPTIONS.COUNT LOOP
      DBMS_OUTPUT.PUT_LINE(
        'Erreur iter. ' || SQL%BULK_EXCEPTIONS(j).ERROR_INDEX ||
        ' (id=' || v_ids(SQL%BULK_EXCEPTIONS(j).ERROR_INDEX) || ')' ||
        ' -> ORA-' || SQL%BULK_EXCEPTIONS(j).ERROR_CODE
      );
    END LOOP;
END;
/

SELECT * FROM ti_cmd ORDER BY id;
/

DROP TABLE ti_cmd;
DROP TABLE ti_clients;
/
```

**Ce qu'il faut retenir :**
- Sans `SAVE EXCEPTIONS` : la premiere erreur arrete tout le batch
- Avec `SAVE EXCEPTIONS` : Oracle continue, puis leve `ORA-24381` apres le `FORALL`
- `SQL%BULK_EXCEPTIONS(j).ERROR_INDEX` = indice de l'iteration en erreur
- `SQL%BULK_EXCEPTIONS(j).ERROR_CODE` = code Oracle (positif, sans le `-`)

---

## 9 · Gestion des erreurs

### Exemple 9-A : Exceptions predefinies: les plus courantes

```sql
SET SERVEROUTPUT ON SIZE UNLIMITED

DECLARE
  v_sal  emp.sal%TYPE;
  v_name emp.ename%TYPE;
BEGIN
  -- NO_DATA_FOUND : SELECT INTO ne trouve rien
  BEGIN
    SELECT sal INTO v_sal FROM emp WHERE empno = 9999;
  EXCEPTION
    WHEN NO_DATA_FOUND THEN
      DBMS_OUTPUT.PUT_LINE('NO_DATA_FOUND : employe 9999 inexistant');
      v_sal := 0;
  END;

  -- TOO_MANY_ROWS : SELECT INTO retourne plusieurs lignes
  BEGIN
    SELECT ename INTO v_name FROM emp WHERE deptno = 30;
  EXCEPTION
    WHEN TOO_MANY_ROWS THEN
      DBMS_OUTPUT.PUT_LINE('TOO_MANY_ROWS : plusieurs employes dept 30');
      DBMS_OUTPUT.PUT_LINE('=> utiliser un curseur ou BULK COLLECT');
  END;

  -- ZERO_DIVIDE
  BEGIN
    v_sal := 100 / 0;
  EXCEPTION
    WHEN ZERO_DIVIDE THEN
      DBMS_OUTPUT.PUT_LINE('ZERO_DIVIDE : division par zero');
  END;

  -- DUP_VAL_ON_INDEX : violation de contrainte unique / PK
  BEGIN
    INSERT INTO dept VALUES (10, 'DOUBLON', 'TEST');
  EXCEPTION
    WHEN DUP_VAL_ON_INDEX THEN
      DBMS_OUTPUT.PUT_LINE('DUP_VAL_ON_INDEX : dept 10 existe deja');
  END;

  DBMS_OUTPUT.PUT_LINE('Programme termine normalement');
END;
/
```

---

### Exemple 9-B : WHEN OTHERS: bon et mauvais usage

```sql
SET SERVEROUTPUT ON SIZE UNLIMITED

-- A NE JAMAIS FAIRE : avale les erreurs silencieusement
DECLARE
  v_sal NUMBER;
BEGIN
  SELECT sal INTO v_sal FROM emp WHERE empno = 9999;
EXCEPTION
  WHEN OTHERS THEN NULL;  -- DANGER : bug invisible, programme continue comme si rien
END;
/
```

```sql
SET SERVEROUTPUT ON SIZE UNLIMITED

-- LE BON PATTERN : logguer ET re-lever
CREATE TABLE ti_err_log (proc VARCHAR2(100), code NUMBER, msg VARCHAR2(4000), dt DATE DEFAULT SYSDATE);
/

DECLARE
  v_sal NUMBER;
BEGIN
  SELECT sal INTO v_sal FROM emp WHERE empno = 9999;
  DBMS_OUTPUT.PUT_LINE('Salaire : ' || v_sal);

EXCEPTION
  WHEN NO_DATA_FOUND THEN
    -- Erreur prevue : traitement specifique
    DBMS_OUTPUT.PUT_LINE('Employe absent, on continue avec sal=0');
    v_sal := 0;

  WHEN OTHERS THEN
    -- Erreur inattendue : logguer et re-lever
    INSERT INTO ti_err_log (proc, code, msg)
    VALUES ('demo_bloc', SQLCODE, SUBSTR(SQLERRM, 1, 4000));
    COMMIT;
    DBMS_OUTPUT.PUT_LINE('Erreur loggue : ' || SQLCODE || ' - ' || SQLERRM);
    RAISE;  -- toujours re-lever dans WHEN OTHERS
END;
/

DROP TABLE ti_err_log;
/
```

**Regle absolue :**
- `WHEN OTHERS THEN NULL` est **interdit en production**: il cache les bugs
- Dans `WHEN OTHERS` : toujours logguer `SQLCODE` + `SQLERRM`, puis `RAISE`

---

### Exemple 9-C : PRAGMA EXCEPTION_INIT: nommer les erreurs Oracle

```sql
SET SERVEROUTPUT ON SIZE UNLIMITED

DECLARE
  -- Associer un nom lisible a un code ORA-
  e_fk_enfant EXCEPTION;
  PRAGMA EXCEPTION_INIT(e_fk_enfant, -2292);  -- ORA-02292 : enfant existant

  e_fk_parent EXCEPTION;
  PRAGMA EXCEPTION_INIT(e_fk_parent, -2291);  -- ORA-02291 : parent inexistant

BEGIN
  -- Tenter de supprimer dept 10 (qui a des employes -> FK violation)
  DELETE FROM dept WHERE deptno = 10;

EXCEPTION
  WHEN e_fk_enfant THEN
    -- Nom explicite : on comprend immediatement sans connaitre -2292
    DBMS_OUTPUT.PUT_LINE('Impossible : des employes existent dans ce departement');
  WHEN e_fk_parent THEN
    DBMS_OUTPUT.PUT_LINE('Impossible : le parent reference n existe pas');
  WHEN OTHERS THEN
    DBMS_OUTPUT.PUT_LINE('Erreur : ' || SQLCODE || ' - ' || SQLERRM);
    RAISE;
END;
/
```

**Ce qu'il faut retenir :**
- `PRAGMA EXCEPTION_INIT` associe un nom PL/SQL a un code `ORA-XXXXX`
- Le nom est declare dans `DECLARE`, le `PRAGMA` juste apres
- Le code doit etre negatif : `PRAGMA EXCEPTION_INIT(e_mon_erreur, -2292)`

---

### Exemple 9-D : RAISE_APPLICATION_ERROR: erreurs metier

```sql
SET SERVEROUTPUT ON SIZE UNLIMITED

-- Codes d'erreur applicatifs centralises dans un package
CREATE OR REPLACE PACKAGE pkg_errors AS
  C_SAL_INVALIDE   CONSTANT NUMBER := -20001;
  C_EMP_INEXISTANT CONSTANT NUMBER := -20002;
  C_DEPT_INEXISTANT CONSTANT NUMBER := -20003;
END;
/

-- Procedure metier avec validations
CREATE OR REPLACE PROCEDURE augmenter_salaire (
  p_empno IN NUMBER,
  p_pct   IN NUMBER
) IS
  v_nb  NUMBER;
  v_sal emp.sal%TYPE;
BEGIN
  -- Validation du pourcentage
  IF p_pct <= 0 OR p_pct > 100 THEN
    RAISE_APPLICATION_ERROR(
      pkg_errors.C_SAL_INVALIDE,
      'Pourcentage invalide : ' || p_pct || '. Attendu : entre 0 et 100.'
    );
  END IF;

  -- Validation de l'employe
  SELECT COUNT(*) INTO v_nb FROM emp WHERE empno = p_empno;
  IF v_nb = 0 THEN
    RAISE_APPLICATION_ERROR(
      pkg_errors.C_EMP_INEXISTANT,
      'Employe ' || p_empno || ' introuvable.'
    );
  END IF;

  -- Appliquer l'augmentation
  UPDATE emp SET sal = sal * (1 + p_pct/100) WHERE empno = p_empno;
  SELECT sal INTO v_sal FROM emp WHERE empno = p_empno;
  DBMS_OUTPUT.PUT_LINE('Nouveau salaire ' || p_empno || ' : ' || v_sal);

EXCEPTION
  WHEN OTHERS THEN
    DBMS_OUTPUT.PUT_LINE('[augmenter_salaire] ' || SQLCODE || ' - ' || SQLERRM);
    RAISE;
END;
/

-- Tests
BEGIN augmenter_salaire(7839, 10); ROLLBACK; END; /   -- OK
BEGIN augmenter_salaire(7839, -5); END; /              -- Erreur -20001
BEGIN augmenter_salaire(9999, 10); END; /              -- Erreur -20002
```

**Ce qu'il faut retenir :**
- Les codes applicatifs sont dans la plage `-20000` a `-20999` (reservee par Oracle)
- Centraliser les codes dans un package (`pkg_errors`) evite les conflits et rend le code lisible
- Le message de `RAISE_APPLICATION_ERROR` remonte jusqu'au client Java/Python/NET

---

## Aide-memoire rapide

### Methodes des collections

| Methode | Description |
|---|---|
| `col.COUNT` | Nombre d'elements |
| `col.FIRST` / `col.LAST` | Premier / dernier indice |
| `col.EXISTS(i)` | TRUE si l'indice i existe |
| `col.EXTEND` | Ajoute 1 element NULL |
| `col.DELETE(i)` | Supprime l'element i |
| `col.NEXT(i)` | Prochain indice apres i |

### Attributs des curseurs

| Attribut | Signification |
|---|---|
| `%FOUND` | TRUE si le dernier FETCH a retourne une ligne |
| `%NOTFOUND` | TRUE si le dernier FETCH n'a rien retourne |
| `%ROWCOUNT` | Nombre de lignes lues depuis OPEN |
| `%ISOPEN` | TRUE si le curseur est ouvert |

### Sous-programmes DBMS_LOB essentiels

| Sous-programme | Role |
|---|---|
| `CREATETEMPORARY(lob, TRUE)` | Creer un LOB temporaire en memoire |
| `FREETEMPORARY(lob)` | Liberer la memoire (OBLIGATOIRE) |
| `WRITEAPPEND(lob, n, data)` | Ajouter `n` octets/car. a la fin |
| `GETLENGTH(lob)` | Taille en octets/caracteres |
| `READ(lob, n, offset, buf)` | Lire `n` octets a partir de `offset` |
| `SUBSTR(lob, n, offset)` | Extraire une sous-chaine |

### Rappels BULK COLLECT + FORALL

```sql
-- Lire en masse (toujours avec LIMIT)
FETCH c BULK COLLECT INTO v_col LIMIT 500;

-- DML en masse
FORALL i IN 1..v_col.COUNT
  INSERT INTO ... VALUES v_col(i);

-- DML en masse avec trous
FORALL i IN INDICES OF v_col
  UPDATE ... WHERE id = v_col(i);

-- DML en masse sans s'arreter sur les erreurs
FORALL i IN 1..v_col.COUNT SAVE EXCEPTIONS
  DELETE FROM ... WHERE id = v_col(i);
```

---

*Fin du guide de travaux pratiques: Jour 1 ORA-PLAV*
