# ORA-PLAV: Oracle PL/SQL Avancé
## Support de Travaux Pratiques: Jour 1
### Schéma : TechInfo Solutions S.A.S

**Formation** : ORA-PLAV: Oracle PL/SQL Avancé
**Durée** : Jour 1 / 7 heures
**Environnement** : Oracle 21c XE · SQL Developer 24.x

---

### Avant de commencer

```sql
SET SERVEROUTPUT ON SIZE UNLIMITED
```
Sous SQL Developer : `View` → `DBMS Output` → `+` (vert)

**Schéma TechInfo Solutions :** préfixe `TI_`

| Table | Colonnes principales | Usage Jour 1 |
|---|---|---|
| `TI_DEPARTEMENTS` | `dept_id, code, nom, localite, budget, actif` | %TYPE, RECORD, Collections |
| `TI_EMPLOYES` | `emp_id, matricule, nom, prenom, poste, salaire, date_embauche, dept_id, mgr_id, actif` | Tous les concepts |
| `TI_CLIENTS` | `client_id, code_client, nom, email, ville, solde, actif` | Curseurs, BULK |
| `TI_PRODUITS` | `produit_id, reference, libelle, prix_ht, stock, stock_min, cat_id, fiche_technique CLOB` | LOBs |
| `TI_COMMANDES` | `cmd_id, numero, client_id, emp_id, date_cmd, statut, montant_ht, montant_ttc` | BULK COLLECT, FORALL |
| `TI_LIGNES_CMD` | `ligne_id, cmd_id, produit_id, quantite, prix_unitaire` | FORALL |
| `TI_HISTORIQUE_SAL` | `hist_id, emp_id, ancien_salaire, nouveau_salaire, motif` | Triggers (Jour 2) |
| `TI_JOURNAL_AUDIT` | `log_id, table_nom, action, utilisateur, date_action TIMESTAMP` | Jour 2 |

**Convention :**
- `-- ?` : ligne à compléter
- `-- A VOUS` : bloc entier à écrire
- ✏ : mini-exercice à réaliser seul

---

## CONCEPT 1: Structure d'un bloc PL/SQL

> Maîtriser les 3 sections, isoler les erreurs avec des blocs imbriqués.

---

### 1-A: Bloc minimal

```sql
BEGIN
  DBMS_OUTPUT.PUT_LINE('TechInfo Solutions: PL/SQL Avancé');
END;
/
```

---

### 1-B: Déclarer et utiliser des variables

```sql
DECLARE
  v_societe   VARCHAR2(100) := 'TechInfo Solutions S.A.S';
  v_nb_emp    NUMBER(6)     := 0;
  v_tva       NUMBER(5,4)   := 0.2000;
  v_date_ex   DATE          := SYSDATE;
  v_message   VARCHAR2(200);
BEGIN
  -- Compter les employés actifs dans TI_EMPLOYES
  SELECT COUNT(*) INTO v_nb_emp
    FROM ti_employes
   WHERE actif = 'O';

  v_message := v_societe || ': ' || v_nb_emp || ' employé(s) actif(s) au '
               || TO_CHAR(v_date_ex, 'DD/MM/YYYY');

  DBMS_OUTPUT.PUT_LINE(v_message);
  DBMS_OUTPUT.PUT_LINE('TVA : ' || (v_tva * 100) || ' %');
END;
/
```

**Points clés :**
- `:=` pour l'affectation (pas `=` comme en SQL)
- `||` pour la concaténation: toute opération avec `NULL` retourne `NULL`

---

### 1-C: La section EXCEPTION

```sql
-- Étape 1 : SANS EXCEPTION: observer l'arrêt du programme
BEGIN
  DBMS_OUTPUT.PUT_LINE('Recherche en cours...');
  DECLARE v_nom ti_employes.nom%TYPE;
  BEGIN
    SELECT nom INTO v_nom FROM ti_employes WHERE emp_id = 99999;
    DBMS_OUTPUT.PUT_LINE('Trouvé : ' || v_nom);
  END;
  DBMS_OUTPUT.PUT_LINE('Fin (jamais atteinte)');
END;
/
```

```sql
-- Étape 2 : AVEC EXCEPTION: récupération propre
DECLARE
  v_nom   ti_employes.nom%TYPE;
  v_found BOOLEAN := FALSE;
BEGIN
  BEGIN
    SELECT nom INTO v_nom FROM ti_employes WHERE emp_id = 99999;
    v_found := TRUE;
  EXCEPTION
    WHEN NO_DATA_FOUND THEN
      DBMS_OUTPUT.PUT_LINE('Employé 99999 introuvable.');
      DBMS_OUTPUT.PUT_LINE('Code  : ' || SQLCODE);
      DBMS_OUTPUT.PUT_LINE('Texte : ' || SQLERRM);
  END;

  DBMS_OUTPUT.PUT_LINE('Résultat : ' || CASE WHEN v_found THEN v_nom ELSE 'N/A' END);
  DBMS_OUTPUT.PUT_LINE('Fin du traitement sans plantage.');
END;
/
```

**`SQLCODE`** : code d'erreur Oracle (négatif)
**`SQLERRM`** : message complet

---

### 1-D: Blocs imbriqués : isoler les erreurs par étape

```sql
DECLARE
  v_client_nom  ti_clients.nom%TYPE;
  v_cmd_numero  ti_commandes.numero%TYPE  := 'AUCUNE';
  v_montant     ti_commandes.montant_ttc%TYPE := 0;
BEGIN
  DBMS_OUTPUT.PUT_LINE('=== Fiche client ===');

  -- Bloc 1 : charger le client (obligatoire)
  SELECT nom INTO v_client_nom FROM ti_clients WHERE client_id = 1;
  DBMS_OUTPUT.PUT_LINE('Client : ' || v_client_nom);

  -- Bloc 2 : dernière commande (optionnelle)
  BEGIN
    SELECT numero, montant_ttc
      INTO v_cmd_numero, v_montant
      FROM ti_commandes
     WHERE client_id = 1
       AND date_cmd  = (SELECT MAX(date_cmd) FROM ti_commandes WHERE client_id = 1)
       AND ROWNUM    = 1;
  EXCEPTION
    WHEN NO_DATA_FOUND THEN
      DBMS_OUTPUT.PUT_LINE('Aucune commande pour ce client.');
  END;

  DBMS_OUTPUT.PUT_LINE('Dernière cmd : ' || v_cmd_numero || ' (' || v_montant || ' EUR)');
END;
/
```

---

### ✏ Mini-exercice 1

Écrivez un bloc qui :
- Déclare `v_dept_id NUMBER := 10`
- Récupère le `nom` et la `localite` de ce département dans `TI_DEPARTEMENTS`
- Affiche : `"Département : IT: Paris"`
- Dans `EXCEPTION WHEN NO_DATA_FOUND` : affiche un message clair

```sql
-- A VOUS
DECLARE
  v_dept_id  NUMBER := 10;
  v_nom      ti_departements.nom%TYPE;
  v_localite ti_departements.localite%TYPE;
BEGIN
  -- ?
  DBMS_OUTPUT.PUT_LINE('Département : ' || v_nom || ': ' || NVL(v_localite, 'N/A'));
EXCEPTION
  -- ?
END;
/
```

---

## CONCEPT 2: Types de données avancés

> TIMESTAMP pour les horodatages précis, INTERVAL pour les durées, BOOLEAN pour la logique, PLS_INTEGER pour les compteurs rapides.

---

### 2-A: DATE vs TIMESTAMP sur TI_EMPLOYES

```sql
DECLARE
  v_date DATE      := SYSDATE;
  v_ts   TIMESTAMP := SYSTIMESTAMP;
BEGIN
  DBMS_OUTPUT.PUT_LINE('DATE      : ' || TO_CHAR(v_date, 'DD/MM/YYYY HH24:MI:SS'));
  DBMS_OUTPUT.PUT_LINE('TIMESTAMP : ' || TO_CHAR(v_ts,   'DD/MM/YYYY HH24:MI:SS.FF6'));

  DBMS_OUTPUT.PUT_LINE('');
  DBMS_OUTPUT.PUT_LINE('--- Dates embauche des employés ---');
  FOR r IN (SELECT nom, prenom, date_embauche FROM ti_employes ORDER BY date_embauche) LOOP
    DBMS_OUTPUT.PUT_LINE(
      RPAD(r.nom || ' ' || r.prenom, 25) ||
      TO_CHAR(r.date_embauche, 'DD/MM/YYYY')
    );
  END LOOP;
END;
/
```

**Règle :** `TI_JOURNAL_AUDIT.date_action` est de type `TIMESTAMP` (pas `DATE`)
pour éviter les collisions sur des événements rapprochés. Toujours utiliser
`SYSTIMESTAMP` dans les tables d'audit.

---

### 2-B: TIMESTAMP: chronométrer un traitement

```sql
DECLARE
  v_debut  TIMESTAMP(6);
  v_fin    TIMESTAMP(6);
  v_duree  INTERVAL DAY(0) TO SECOND(3);
  v_nb     PLS_INTEGER := 0;
  v_total  NUMBER := 0;
BEGIN
  v_debut := SYSTIMESTAMP;

  -- Calculer la masse salariale totale
  FOR r IN (SELECT salaire FROM ti_employes WHERE actif = 'O') LOOP
    v_total := v_total + r.salaire;
    v_nb    := v_nb + 1;
  END LOOP;

  v_fin   := SYSTIMESTAMP;
  v_duree := v_fin - v_debut;

  DBMS_OUTPUT.PUT_LINE('Employés actifs : ' || v_nb);
  DBMS_OUTPUT.PUT_LINE('Masse salariale : ' || TO_CHAR(v_total, '999G999G999D00') || ' EUR');
  DBMS_OUTPUT.PUT_LINE('Durée           : ' || v_duree);
END;
/
```

---

### 2-C: INTERVAL: délais sur TI_COMMANDES

```sql
DECLARE
  v_delai_std INTERVAL DAY(2) TO SECOND := INTERVAL '5 00:00:00' DAY TO SECOND;
BEGIN
  -- Ancienneté des employés
  DBMS_OUTPUT.PUT_LINE('--- Ancienneté du personnel ---');
  FOR r IN (
    SELECT nom, prenom, date_embauche,
           TRUNC(MONTHS_BETWEEN(SYSDATE, date_embauche) / 12) annees
      FROM ti_employes WHERE actif = 'O' ORDER BY date_embauche
  ) LOOP
    DBMS_OUTPUT.PUT_LINE(RPAD(r.nom || ' ' || r.prenom, 25) || r.annees || ' an(s)');
  END LOOP;

  -- Dates de livraison estimées
  DBMS_OUTPUT.PUT_LINE('');
  DBMS_OUTPUT.PUT_LINE('--- Livraisons estimées (délai std 5j) ---');
  FOR r IN (
    SELECT numero, date_cmd FROM ti_commandes
     WHERE statut IN ('NOUVEAU','EN_COURS') ORDER BY date_cmd
  ) LOOP
    DBMS_OUTPUT.PUT_LINE(
      RPAD(r.numero, 15) ||
      TO_CHAR(r.date_cmd,              'DD/MM/YYYY') || ' → ' ||
      TO_CHAR(r.date_cmd + v_delai_std,'DD/MM/YYYY')
    );
  END LOOP;
END;
/
```

---

### 2-D: BOOLEAN: règles métier sur TI_EMPLOYES

```sql
DECLARE
  v_salaire   ti_employes.salaire%TYPE;
  v_actif     ti_employes.actif%TYPE;
  v_senior    BOOLEAN;
  v_eligible  BOOLEAN;
BEGIN
  SELECT salaire, actif INTO v_salaire, v_actif
    FROM ti_employes WHERE emp_id = 1;

  v_senior   := (v_salaire > 4000);
  v_eligible := (v_actif = 'O') AND v_senior;

  DBMS_OUTPUT.PUT_LINE('Salaire   : ' || v_salaire);
  DBMS_OUTPUT.PUT_LINE('Senior    : ' || CASE WHEN v_senior   THEN 'OUI' ELSE 'NON' END);
  DBMS_OUTPUT.PUT_LINE('Éligible  : ' || CASE WHEN v_eligible THEN 'OUI' ELSE 'NON' END);

  -- BOOLEAN ne peut pas aller en colonne SQL → conversion obligatoire
  DBMS_OUTPUT.PUT_LINE('Flag CHAR : ' || CASE WHEN v_eligible THEN 'O' ELSE 'N' END);
END;
/
```

---

### 2-E: PLS_INTEGER: compteurs sur grands volumes

```sql
DECLARE
  v_debut_n  TIMESTAMP(6);
  v_debut_p  TIMESTAMP(6);
  v_fin      TIMESTAMP(6);
  v_count_n  NUMBER      := 0;
  v_count_p  PLS_INTEGER := 0;
BEGIN
  v_debut_n := SYSTIMESTAMP;
  FOR i IN 1..500000 LOOP v_count_n := v_count_n + 1; END LOOP;
  v_fin := SYSTIMESTAMP;
  DBMS_OUTPUT.PUT_LINE('NUMBER      : ' || (v_fin - v_debut_n));

  v_debut_p := SYSTIMESTAMP;
  FOR i IN 1..500000 LOOP v_count_p := v_count_p + 1; END LOOP;
  v_fin := SYSTIMESTAMP;
  DBMS_OUTPUT.PUT_LINE('PLS_INTEGER : ' || (v_fin - v_debut_p));

  DBMS_OUTPUT.PUT_LINE('=> Utiliser PLS_INTEGER pour tout compteur de boucle et indice de collection');
END;
/
```

---

### ✏ Mini-exercice 2

Chronométrez le calcul du montant TTC total de toutes les commandes de `TI_COMMANDES`.
Utilisez un compteur `PLS_INTEGER` pour compter les lignes, `SYSTIMESTAMP` pour la durée.
Affichez : nombre de commandes, total TTC, durée en secondes.

```sql
-- A VOUS
DECLARE
  v_debut  TIMESTAMP(6) := SYSTIMESTAMP;
  v_duree  INTERVAL DAY(0) TO SECOND(3);
  v_nb     PLS_INTEGER := 0;
  v_total  NUMBER      := 0;
BEGIN
  -- Parcourir TI_COMMANDES
  -- ?
  v_duree := SYSTIMESTAMP - v_debut;
  DBMS_OUTPUT.PUT_LINE('Commandes : ' || v_nb);
  DBMS_OUTPUT.PUT_LINE('Total TTC : ' || v_total || ' EUR');
  DBMS_OUTPUT.PUT_LINE('Durée     : ' || EXTRACT(SECOND FROM v_duree) || 's');
END;
/
```

---

## CONCEPT 3: %TYPE et %ROWTYPE

> Ancrer les variables sur le schéma pour un code robuste aux évolutions DDL.

---

### 3-A: Le problème sans %TYPE

```sql
-- Fragile : si ti_employes.nom passe de VARCHAR2(50) à VARCHAR2(100)
DECLARE
  v_nom_fragile VARCHAR2(30);   -- type en dur
BEGIN
  SELECT nom INTO v_nom_fragile FROM ti_employes WHERE emp_id = 1;
  DBMS_OUTPUT.PUT_LINE('Fragile : ' || v_nom_fragile);
END;
/

-- Robuste : ancrage sur la colonne
DECLARE
  v_nom ti_employes.nom%TYPE;   -- hérite du type réel de la colonne
BEGIN
  SELECT nom INTO v_nom FROM ti_employes WHERE emp_id = 1;
  DBMS_OUTPUT.PUT_LINE('Robuste : ' || v_nom);
END;
/
```

---

### 3-B: %TYPE sur une fiche employé complète

```sql
DECLARE
  v_emp_id    ti_employes.emp_id%TYPE;
  v_matricule ti_employes.matricule%TYPE;
  v_nom       ti_employes.nom%TYPE;
  v_prenom    ti_employes.prenom%TYPE;
  v_poste     ti_employes.poste%TYPE;
  v_salaire   ti_employes.salaire%TYPE;
  v_dept_id   ti_employes.dept_id%TYPE;
BEGIN
  SELECT emp_id, matricule, nom, prenom, poste, salaire, dept_id
    INTO v_emp_id, v_matricule, v_nom, v_prenom, v_poste, v_salaire, v_dept_id
    FROM ti_employes WHERE emp_id = 1;

  DBMS_OUTPUT.PUT_LINE('=== Fiche Employé ===');
  DBMS_OUTPUT.PUT_LINE('Matricule : ' || v_matricule);
  DBMS_OUTPUT.PUT_LINE('Nom       : ' || v_nom || ' ' || v_prenom);
  DBMS_OUTPUT.PUT_LINE('Poste     : ' || v_poste);
  DBMS_OUTPUT.PUT_LINE('Salaire   : ' || v_salaire || ' EUR');
  DBMS_OUTPUT.PUT_LINE('Dept ID   : ' || v_dept_id);
END;
/
```

---

### 3-C: %ROWTYPE sur TI_EMPLOYES et TI_DEPARTEMENTS

```sql
DECLARE
  r_emp  ti_employes%ROWTYPE;
  r_dept ti_departements%ROWTYPE;
BEGIN
  SELECT * INTO r_emp  FROM ti_employes    WHERE emp_id  = 1;
  SELECT * INTO r_dept FROM ti_departements WHERE dept_id = r_emp.dept_id;

  DBMS_OUTPUT.PUT_LINE('=== Fiche complète ===');
  DBMS_OUTPUT.PUT_LINE('Employé    : ' || r_emp.nom || ' ' || r_emp.prenom);
  DBMS_OUTPUT.PUT_LINE('Matricule  : ' || r_emp.matricule);
  DBMS_OUTPUT.PUT_LINE('Salaire    : ' || r_emp.salaire || ' EUR');
  DBMS_OUTPUT.PUT_LINE('Email      : ' || NVL(r_emp.email, 'Non renseigné'));
  DBMS_OUTPUT.PUT_LINE('Actif      : ' || CASE WHEN r_emp.actif = 'O' THEN 'Oui' ELSE 'Non' END);
  DBMS_OUTPUT.PUT_LINE('Département: ' || r_dept.nom || ' (' || r_dept.code || ')');
  DBMS_OUTPUT.PUT_LINE('Budget     : ' || NVL(TO_CHAR(r_dept.budget,'999G999D00'),'-') || ' EUR');
END;
/
```

---

### 3-D: %ROWTYPE sur curseur: plus économique

```sql
DECLARE
  -- Sélectionner seulement les colonnes nécessaires
  CURSOR c IS
    SELECT e.emp_id, e.nom, e.prenom, e.salaire, d.nom AS dept_nom
      FROM ti_employes    e
      JOIN ti_departements d ON d.dept_id = e.dept_id
     WHERE e.actif = 'O'
     ORDER BY e.salaire DESC;

  r c%ROWTYPE;   -- seulement 5 colonnes, pas les 12 de TI_EMPLOYES
BEGIN
  OPEN c;
  LOOP
    FETCH c INTO r;
    EXIT WHEN c%NOTFOUND;
    DBMS_OUTPUT.PUT_LINE(
      RPAD(r.nom || ' ' || r.prenom, 25) ||
      LPAD(r.salaire, 10) || ' EUR : ' || r.dept_nom
    );
  END LOOP;
  CLOSE c;
END;
/
```

---

### ✏ Mini-exercice 3

Utilisez `%ROWTYPE` pour charger l'employé `emp_id = 2`.
Récupérez séparément le nom du département (`TI_DEPARTEMENTS`) avec `%TYPE`.
Affichez : `"MARTIN Sophie est ANALYSTE dans INFORMATIQUE avec 3200 EUR"`

```sql
-- A VOUS
DECLARE
  r_emp    ti_employes%ROWTYPE;
  v_dnom   ti_departements.nom%TYPE;
BEGIN
  SELECT * INTO r_emp FROM ti_employes WHERE emp_id = 2;
  -- Récupérer le nom du département
  -- ?
  -- Afficher
  -- ?
END;
/
```

---

## CONCEPT 4: RECORD

> Créer des structures sur mesure combinant des colonnes de plusieurs tables.

---

### 4-A: RECORD simple

```sql
DECLARE
  TYPE t_contact IS RECORD (
    nom       VARCHAR2(100),
    email     VARCHAR2(150),
    telephone VARCHAR2(20),
    ville     VARCHAR2(50),
    actif     BOOLEAN DEFAULT TRUE
  );
  v_contact t_contact;
BEGIN
  v_contact.nom       := 'TechInfo Solutions';
  v_contact.email     := 'contact@techinfo.fr';
  v_contact.telephone := '01 23 45 67 89';
  v_contact.ville     := 'Paris';

  DBMS_OUTPUT.PUT_LINE('Contact : ' || v_contact.nom);
  DBMS_OUTPUT.PUT_LINE('Email   : ' || v_contact.email);
  DBMS_OUTPUT.PUT_LINE('Ville   : ' || v_contact.ville);
  DBMS_OUTPUT.PUT_LINE('Actif   : ' || CASE WHEN v_contact.actif THEN 'Oui' ELSE 'Non' END);
END;
/
```

---

### 4-B: RECORD avec %TYPE: fiche employé enrichie

```sql
DECLARE
  TYPE t_fiche_emp IS RECORD (
    emp_id      ti_employes.emp_id%TYPE,
    nom_complet VARCHAR2(102),
    poste       ti_employes.poste%TYPE,
    salaire     ti_employes.salaire%TYPE,
    dept_nom    ti_departements.nom%TYPE,
    dept_code   ti_departements.code%TYPE,
    localite    ti_departements.localite%TYPE,
    manager     VARCHAR2(102)
  );
  v_fiche t_fiche_emp;
BEGIN
  SELECT e.emp_id,
         e.nom || ' ' || e.prenom,
         e.poste, e.salaire,
         d.nom, d.code, d.localite,
         NVL(m.nom || ' ' || m.prenom, 'Aucun manager')
    INTO v_fiche.emp_id, v_fiche.nom_complet, v_fiche.poste, v_fiche.salaire,
         v_fiche.dept_nom, v_fiche.dept_code, v_fiche.localite, v_fiche.manager
    FROM ti_employes    e
    JOIN ti_departements d ON d.dept_id = e.dept_id
    LEFT JOIN ti_employes m ON m.emp_id = e.mgr_id
   WHERE e.emp_id = 1;

  DBMS_OUTPUT.PUT_LINE('=== Fiche RH ===');
  DBMS_OUTPUT.PUT_LINE('Employé  : ' || v_fiche.nom_complet);
  DBMS_OUTPUT.PUT_LINE('Poste    : ' || v_fiche.poste);
  DBMS_OUTPUT.PUT_LINE('Salaire  : ' || v_fiche.salaire || ' EUR');
  DBMS_OUTPUT.PUT_LINE('Dept     : ' || v_fiche.dept_nom || ' (' || v_fiche.dept_code || ')');
  DBMS_OUTPUT.PUT_LINE('Localité : ' || NVL(v_fiche.localite,'N/A'));
  DBMS_OUTPUT.PUT_LINE('Manager  : ' || v_fiche.manager);
END;
/
```

---

### 4-C: RECORD comme paramètre de procédure

```sql
DECLARE
  TYPE t_nouvel_emp IS RECORD (
    nom     ti_employes.nom%TYPE,
    prenom  ti_employes.prenom%TYPE,
    poste   ti_employes.poste%TYPE,
    salaire ti_employes.salaire%TYPE,
    dept_id ti_employes.dept_id%TYPE,
    email   ti_employes.email%TYPE
  );

  PROCEDURE valider_candidat (p_emp IN t_nouvel_emp) IS
  BEGIN
    DBMS_OUTPUT.PUT_LINE('--- Dossier candidat ---');
    DBMS_OUTPUT.PUT_LINE('Nom      : ' || p_emp.nom || ' ' || p_emp.prenom);
    DBMS_OUTPUT.PUT_LINE('Poste    : ' || p_emp.poste);
    DBMS_OUTPUT.PUT_LINE('Salaire  : ' || p_emp.salaire || ' EUR');
    DBMS_OUTPUT.PUT_LINE('Dept ID  : ' || p_emp.dept_id);
    IF p_emp.salaire < 1500 THEN
      DBMS_OUTPUT.PUT_LINE('/!\ Salaire inférieur au minimum légal (1500 EUR)');
    END IF;
  END;

  v_candidat t_nouvel_emp;
BEGIN
  v_candidat.nom     := 'DUPONT';
  v_candidat.prenom  := 'Marie';
  v_candidat.poste   := 'ANALYSTE';
  v_candidat.salaire := 3800;
  v_candidat.dept_id := 10;
  v_candidat.email   := 'm.dupont@techinfo.fr';

  valider_candidat(v_candidat);
END;
/
```

---

### 4-D: RECORD dans une collection: stats par département

```sql
DECLARE
  TYPE t_stat_dept IS RECORD (
    dept_code  ti_departements.code%TYPE,
    dept_nom   ti_departements.nom%TYPE,
    nb_emp     PLS_INTEGER,
    sal_min    ti_employes.salaire%TYPE,
    sal_max    ti_employes.salaire%TYPE,
    sal_moyen  NUMBER(10,2)
  );
  TYPE t_stats IS TABLE OF t_stat_dept;
  v_stats t_stats := t_stats();
  v_idx   PLS_INTEGER := 0;
BEGIN
  FOR r IN (
    SELECT d.code, d.nom,
           COUNT(e.emp_id)    nb_emp,
           MIN(e.salaire)     sal_min,
           MAX(e.salaire)     sal_max,
           ROUND(AVG(e.salaire),2) sal_moyen
      FROM ti_departements d
      LEFT JOIN ti_employes e ON e.dept_id = d.dept_id AND e.actif = 'O'
     GROUP BY d.code, d.nom ORDER BY d.code
  ) LOOP
    v_idx := v_idx + 1;
    v_stats.EXTEND;
    v_stats(v_idx).dept_code := r.code;
    v_stats(v_idx).dept_nom  := r.nom;
    v_stats(v_idx).nb_emp    := r.nb_emp;
    v_stats(v_idx).sal_min   := r.sal_min;
    v_stats(v_idx).sal_max   := r.sal_max;
    v_stats(v_idx).sal_moyen := r.sal_moyen;
  END LOOP;

  DBMS_OUTPUT.PUT_LINE(RPAD('CODE',8) || RPAD('DÉPARTEMENT',20) || RPAD('EMP',5) ||
                       RPAD('MIN',10) || RPAD('MAX',10) || 'MOYEN');
  DBMS_OUTPUT.PUT_LINE(RPAD('-',60,'-'));
  FOR i IN 1..v_stats.COUNT LOOP
    DBMS_OUTPUT.PUT_LINE(
      RPAD(v_stats(i).dept_code, 8) ||
      RPAD(v_stats(i).dept_nom,  20) ||
      RPAD(NVL(TO_CHAR(v_stats(i).nb_emp),'0'),  5) ||
      RPAD(NVL(TO_CHAR(v_stats(i).sal_min),'-'), 10) ||
      RPAD(NVL(TO_CHAR(v_stats(i).sal_max),'-'), 10) ||
      NVL(TO_CHAR(v_stats(i).sal_moyen),'-')
    );
  END LOOP;
END;
/
```

---

### ✏ Mini-exercice 4

Créez un `TYPE RECORD t_cmd_resume` contenant :
`numero (ti_commandes.numero%TYPE)`, `client_nom (ti_clients.nom%TYPE)`,
`montant_ttc (ti_commandes.montant_ttc%TYPE)`, `statut (ti_commandes.statut%TYPE)`,
`nb_lignes NUMBER`.

Pour chaque commande de `TI_COMMANDES`, remplissez le RECORD via une jointure
avec `TI_CLIENTS` et un COUNT sur `TI_LIGNES_CMD`.
Affichez : `"CMD-1001  TechInfo SARL    1200.00 EUR  LIVRE  [3 lignes]"`

```sql
-- A VOUS
DECLARE
  TYPE t_cmd_resume IS RECORD (
    numero      ti_commandes.numero%TYPE,
    client_nom  ti_clients.nom%TYPE,
    montant_ttc ti_commandes.montant_ttc%TYPE,
    statut      ti_commandes.statut%TYPE,
    nb_lignes   NUMBER
  );
  v_cmd t_cmd_resume;
BEGIN
  FOR r IN (SELECT cmd_id FROM ti_commandes ORDER BY date_cmd) LOOP
    -- Remplir le RECORD via jointures
    -- ?
    -- Afficher
    -- ?
  END LOOP;
END;
/
```

---

## CONCEPT 5: Collections PL/SQL

> Nested TABLE pour les listes dynamiques, VARRAY pour les listes fixes, Associative Array pour les caches.

---

### 5-A: Nested TABLE: produits en alerte de stock

```sql
DECLARE
  TYPE t_prod_ref IS TABLE OF ti_produits.reference%TYPE;
  TYPE t_prod_lib IS TABLE OF ti_produits.libelle%TYPE;
  TYPE t_prod_stk IS TABLE OF ti_produits.stock%TYPE;

  v_refs t_prod_ref := t_prod_ref();
  v_libs t_prod_lib := t_prod_lib();
  v_stks t_prod_stk := t_prod_stk();
  v_idx  PLS_INTEGER := 0;
BEGIN
  FOR r IN (
    SELECT reference, libelle, stock FROM ti_produits
     WHERE stock <= stock_min AND actif = 'O' ORDER BY stock
  ) LOOP
    v_idx := v_idx + 1;
    v_refs.EXTEND; v_refs(v_idx) := r.reference;
    v_libs.EXTEND; v_libs(v_idx) := r.libelle;
    v_stks.EXTEND; v_stks(v_idx) := r.stock;
  END LOOP;

  DBMS_OUTPUT.PUT_LINE('=== Alerte stock: ' || v_idx || ' produit(s) ===');
  FOR i IN v_refs.FIRST..v_refs.LAST LOOP
    DBMS_OUTPUT.PUT_LINE(RPAD(v_refs(i), 15) || RPAD(v_libs(i), 30) || ' Stock: ' || v_stks(i));
  END LOOP;

  DBMS_OUTPUT.PUT_LINE('COUNT=' || v_refs.COUNT || '  FIRST=' || v_refs.FIRST || '  LAST=' || v_refs.LAST);
END;
/
```

---

### 5-B: Méthodes DELETE et parcours FIRST/NEXT

```sql
DECLARE
  TYPE t_cmd_ids IS TABLE OF ti_commandes.cmd_id%TYPE;
  v_cmds t_cmd_ids := t_cmd_ids();
  v_idx  PLS_INTEGER := 0;
BEGIN
  FOR r IN (SELECT cmd_id FROM ti_commandes ORDER BY cmd_id) LOOP
    v_idx := v_idx + 1;
    v_cmds.EXTEND;
    v_cmds(v_idx) := r.cmd_id;
  END LOOP;

  DBMS_OUTPUT.PUT_LINE('Avant DELETE : COUNT = ' || v_cmds.COUNT);
  v_cmds.DELETE(2);
  v_cmds.DELETE(4);
  DBMS_OUTPUT.PUT_LINE('Après DELETE(2) et (4) : COUNT = ' || v_cmds.COUNT);
  DBMS_OUTPUT.PUT_LINE('EXISTS(2) = ' || CASE WHEN v_cmds.EXISTS(2) THEN 'TRUE' ELSE 'FALSE' END);

  -- Parcours SÛUR avec FIRST/NEXT
  DBMS_OUTPUT.PUT_LINE('Parcours sûr :');
  DECLARE v_i PLS_INTEGER := v_cmds.FIRST;
  BEGIN
    WHILE v_i IS NOT NULL LOOP
      DBMS_OUTPUT.PUT_LINE('  indice ' || v_i || ' → cmd_id=' || v_cmds(v_i));
      v_i := v_cmds.NEXT(v_i);
    END LOOP;
  END;

  DBMS_OUTPUT.PUT_LINE('ATTENTION : FOR i IN 1..COUNT avec trou → exception !');
END;
/
```

---

### 5-C: VARRAY: workflow des statuts de commande

```sql
DECLARE
  TYPE t_statuts IS VARRAY(5) OF VARCHAR2(20);
  v_workflow t_statuts := t_statuts('NOUVEAU','EN_COURS','EXPEDIE','LIVRE','ANNULE');
BEGIN
  DBMS_OUTPUT.PUT_LINE('=== Workflow commandes TechInfo ===');
  FOR i IN 1..v_workflow.COUNT LOOP
    DECLARE v_nb NUMBER;
    BEGIN
      SELECT COUNT(*) INTO v_nb FROM ti_commandes WHERE statut = v_workflow(i);
      DBMS_OUTPUT.PUT_LINE('  ' || RPAD(v_workflow(i), 12) || ' : ' || v_nb || ' commande(s)');
    END;
  END LOOP;

  DBMS_OUTPUT.PUT_LINE('LIMIT = ' || v_workflow.LIMIT || '  COUNT = ' || v_workflow.COUNT);
END;
/
```

---

### 5-D: Associative Array: cache du référentiel catégories et prix

```sql
DECLARE
  TYPE t_cat_cache  IS TABLE OF ti_categories.libelle%TYPE INDEX BY PLS_INTEGER;
  TYPE t_prix_cache IS TABLE OF ti_produits.prix_ht%TYPE   INDEX BY PLS_INTEGER;

  v_cats t_cat_cache;
  v_prix t_prix_cache;
BEGIN
  -- Charger les référentiels UNE seule fois
  FOR r IN (SELECT cat_id, libelle FROM ti_categories) LOOP
    v_cats(r.cat_id) := r.libelle;
  END LOOP;
  FOR r IN (SELECT produit_id, prix_ht FROM ti_produits WHERE actif = 'O') LOOP
    v_prix(r.produit_id) := r.prix_ht;
  END LOOP;

  -- Utiliser les caches sans SELECT supplémentaire
  DBMS_OUTPUT.PUT_LINE('--- Produits (0 SELECT supplémentaire) ---');
  FOR r IN (SELECT produit_id, reference, libelle, cat_id FROM ti_produits
             WHERE actif = 'O' ORDER BY cat_id, reference)
  LOOP
    IF v_cats.EXISTS(r.cat_id) AND v_prix.EXISTS(r.produit_id) THEN
      DBMS_OUTPUT.PUT_LINE(
        RPAD(r.reference, 15) ||
        RPAD(r.libelle,   30) ||
        LPAD(v_prix(r.produit_id), 8) || ' EUR : ' || v_cats(r.cat_id)
      );
    END IF;
  END LOOP;
END;
/
```

---

### 5-E: Associative Array clé VARCHAR2: libellés de statuts

```sql
DECLARE
  TYPE t_libelle IS TABLE OF VARCHAR2(100) INDEX BY VARCHAR2(20);
  v_lib t_libelle;
BEGIN
  v_lib('NOUVEAU')   := 'Enregistrée, en attente de traitement';
  v_lib('EN_COURS')  := 'En cours de préparation';
  v_lib('EXPEDIE')   := 'Expédiée: suivi disponible';
  v_lib('LIVRE')     := 'Livrée et confirmée';
  v_lib('ANNULE')    := 'Annulée';

  FOR r IN (SELECT numero, statut, montant_ttc FROM ti_commandes ORDER BY date_cmd) LOOP
    IF v_lib.EXISTS(r.statut) THEN
      DBMS_OUTPUT.PUT_LINE(
        RPAD(r.numero, 15) || LPAD(r.montant_ttc, 10) || ' EUR : ' || v_lib(r.statut)
      );
    END IF;
  END LOOP;
END;
/
```

---

### ✏ Mini-exercice 5

En combinant `Nested TABLE` et `Associative Array` :
1. Créez un `RECORD t_prod_info` : `reference, libelle, prix_ht, cat_libelle, niveau_stock CHAR(1)`
2. Niveaux de stock : `stock = 0` → `'C'` (critique), `stock <= stock_min` → `'A'` (alerte), sinon `'N'` (normal)
3. Chargez tous les produits actifs dans une `Nested TABLE` de ce RECORD
4. Utilisez un `Associative Array` pour le cache des catégories
5. Affichez les produits triés par niveau de stock (C d'abord)

```sql
-- A VOUS
DECLARE
  TYPE t_prod_info IS RECORD (
    reference     ti_produits.reference%TYPE,
    libelle       ti_produits.libelle%TYPE,
    prix_ht       ti_produits.prix_ht%TYPE,
    cat_libelle   ti_categories.libelle%TYPE,
    niveau_stock  CHAR(1)
  );
  TYPE t_liste IS TABLE OF t_prod_info;
  TYPE t_cat   IS TABLE OF ti_categories.libelle%TYPE INDEX BY PLS_INTEGER;

  v_liste t_liste  := t_liste();
  v_cats  t_cat;
  v_idx   PLS_INTEGER := 0;
BEGIN
  -- Charger le cache des catégories
  -- ?

  -- Charger les produits
  FOR r IN (SELECT produit_id, reference, libelle, prix_ht, stock, stock_min, cat_id
              FROM ti_produits WHERE actif = 'O' ORDER BY stock) LOOP
    v_idx := v_idx + 1;
    v_liste.EXTEND;
    -- Remplir le RECORD
    -- ?
  END LOOP;

  -- Afficher
  FOR i IN 1..v_liste.COUNT LOOP
    -- Exemple : "[C] REF-001  Produit X  99.00 EUR  Informatique"
    -- ?
  END LOOP;
END;
/
```

---

## CONCEPT 6: Large Objects (LOBs)

> CREATETEMPORARY → WRITEAPPEND → READ (par chunks) → FREETEMPORARY.
> FREETEMPORARY est **obligatoire** dans le flux normal **et** dans EXCEPTION.

---

### 6-A: Écrire une fiche technique dans TI_PRODUITS

```sql
DECLARE
  v_clob CLOB;
  v_lig  VARCHAR2(300);
  v_ref  ti_produits.reference%TYPE;
  v_lib  ti_produits.libelle%TYPE;
  v_prix ti_produits.prix_ht%TYPE;
BEGIN
  SELECT reference, libelle, prix_ht INTO v_ref, v_lib, v_prix
    FROM ti_produits WHERE produit_id = 1;

  -- Récupérer le localisateur pour écriture
  UPDATE ti_produits SET fiche_technique = EMPTY_CLOB()
   WHERE produit_id = 1
  RETURNING fiche_technique INTO v_clob;

  -- Construire la fiche
  v_lig := '=== FICHE TECHNIQUE ===' || CHR(10);
  DBMS_LOB.WRITEAPPEND(v_clob, LENGTH(v_lig), v_lig);

  v_lig := 'Référence   : ' || v_ref || CHR(10);
  DBMS_LOB.WRITEAPPEND(v_clob, LENGTH(v_lig), v_lig);

  v_lig := 'Désignation : ' || v_lib || CHR(10);
  DBMS_LOB.WRITEAPPEND(v_clob, LENGTH(v_lig), v_lig);

  v_lig := 'Prix HT     : ' || TO_CHAR(v_prix,'999G999D00') || ' EUR' || CHR(10);
  DBMS_LOB.WRITEAPPEND(v_clob, LENGTH(v_lig), v_lig);

  v_lig := 'Mis à jour  : ' || TO_CHAR(SYSDATE,'DD/MM/YYYY') || CHR(10);
  DBMS_LOB.WRITEAPPEND(v_clob, LENGTH(v_lig), v_lig);

  COMMIT;
  DBMS_OUTPUT.PUT_LINE('Fiche enregistrée: taille : ' || DBMS_LOB.GETLENGTH(v_clob) || ' car.');
END;
/
```

---

### 6-B: Lire la fiche technique par chunks

```sql
DECLARE
  v_clob   CLOB;
  v_offset INTEGER := 1;
  v_amount INTEGER;
  v_buffer VARCHAR2(4000);
  v_len    INTEGER;
  v_ref    ti_produits.reference%TYPE;
BEGIN
  SELECT reference, fiche_technique INTO v_ref, v_clob
    FROM ti_produits WHERE produit_id = 1;

  v_len := DBMS_LOB.GETLENGTH(v_clob);
  IF v_len IS NULL OR v_len = 0 THEN
    DBMS_OUTPUT.PUT_LINE(v_ref || ' : aucune fiche technique.');
    RETURN;
  END IF;

  DBMS_OUTPUT.PUT_LINE('Produit : ' || v_ref || ': ' || v_len || ' car.');
  DBMS_OUTPUT.PUT_LINE('');

  -- Lecture par morceaux de 200 caractères
  WHILE v_offset <= v_len LOOP
    v_amount := LEAST(200, v_len - v_offset + 1);
    DBMS_LOB.READ(v_clob, v_amount, v_offset, v_buffer);
    DBMS_OUTPUT.PUT(v_buffer);
    v_offset := v_offset + v_amount;
  END LOOP;
END;
/
```

---

### 6-C: CLOB temporaire: rapport de commandes

```sql
DECLARE
  v_rapport CLOB;
  v_lig     VARCHAR2(300);
  v_offset  INTEGER := 1;
  v_amount  INTEGER;
  v_buffer  VARCHAR2(4000);
  v_len     INTEGER;
  v_total   NUMBER := 0;
  v_nb      PLS_INTEGER := 0;
BEGIN
  DBMS_LOB.CREATETEMPORARY(v_rapport, TRUE);

  -- En-tête
  v_lig := 'RAPPORT COMMANDES TECHINFO' || CHR(10) ||
           TO_CHAR(SYSDATE,'DD/MM/YYYY HH24:MI:SS') || CHR(10) ||
           RPAD('=',70,'=') || CHR(10) || CHR(10);
  DBMS_LOB.WRITEAPPEND(v_rapport, LENGTH(v_lig), v_lig);

  -- Lignes de commandes
  FOR r IN (
    SELECT c.numero, c.statut, c.date_cmd, c.montant_ttc, cl.nom AS client
      FROM ti_commandes c JOIN ti_clients cl ON cl.client_id = c.client_id
     ORDER BY c.date_cmd
  ) LOOP
    v_lig := RPAD(r.numero, 15) || RPAD(r.client, 25) ||
             TO_CHAR(r.date_cmd,'DD/MM/YY') || '  ' ||
             RPAD(r.statut, 12) ||
             LPAD(TO_CHAR(r.montant_ttc,'999G999D00'),12) || ' EUR' || CHR(10);
    DBMS_LOB.WRITEAPPEND(v_rapport, LENGTH(v_lig), v_lig);
    v_total := v_total + r.montant_ttc;
    v_nb    := v_nb + 1;
  END LOOP;

  -- Pied de page
  v_lig := CHR(10) || RPAD('-',70,'-') || CHR(10) ||
           v_nb || ' commande(s): CA TTC : ' || TO_CHAR(v_total,'999G999G999D00') || ' EUR' || CHR(10);
  DBMS_LOB.WRITEAPPEND(v_rapport, LENGTH(v_lig), v_lig);

  -- Affichage
  v_len := DBMS_LOB.GETLENGTH(v_rapport);
  DBMS_OUTPUT.PUT_LINE('Rapport : ' || v_len || ' car.');
  WHILE v_offset <= v_len LOOP
    v_amount := LEAST(4000, v_len - v_offset + 1);
    DBMS_LOB.READ(v_rapport, v_amount, v_offset, v_buffer);
    DBMS_OUTPUT.PUT(v_buffer);
    v_offset := v_offset + v_amount;
  END LOOP;

  DBMS_LOB.FREETEMPORARY(v_rapport);   -- chemin normal

EXCEPTION
  WHEN OTHERS THEN
    IF DBMS_LOB.ISTEMPORARY(v_rapport) = 1 THEN
      DBMS_LOB.FREETEMPORARY(v_rapport);   -- chemin erreur
    END IF;
    RAISE;
END;
/
```

---

### 6-D: FREETEMPORARY dans EXCEPTION: bonne pratique

```sql
-- Version INCORRECTE : fuite mémoire
DECLARE v_clob CLOB;
BEGIN
  DBMS_LOB.CREATETEMPORARY(v_clob, TRUE);
  RAISE_APPLICATION_ERROR(-20001, 'Erreur simulée');
  DBMS_LOB.FREETEMPORARY(v_clob);  -- jamais atteinte → FUITE
EXCEPTION
  WHEN OTHERS THEN
    DBMS_OUTPUT.PUT_LINE('Incorrect: LOB non libéré');
END;
/

-- Version CORRECTE
DECLARE v_clob CLOB;
BEGIN
  DBMS_LOB.CREATETEMPORARY(v_clob, TRUE);
  RAISE_APPLICATION_ERROR(-20001, 'Erreur simulée');
  DBMS_LOB.FREETEMPORARY(v_clob);  -- chemin normal
EXCEPTION
  WHEN OTHERS THEN
    IF DBMS_LOB.ISTEMPORARY(v_clob) = 1 THEN
      DBMS_LOB.FREETEMPORARY(v_clob);
      DBMS_OUTPUT.PUT_LINE('LOB libéré dans EXCEPTION: correct');
    END IF;
END;
/
```

---

### ✏ Mini-exercice 6

Générez dans un CLOB temporaire la liste complète de tous les produits actifs au format CSV :
`REFERENCE,LIBELLE,PRIX_HT,STOCK,CATEGORIE`

Exemple de ligne : `REF-001,Laptop Pro,1299.00,15,Informatique`

Affichez la taille du CLOB et les 3 premières lignes.
Garantissez `FREETEMPORARY` dans `EXCEPTION`.

```sql
-- A VOUS
DECLARE
  v_csv    CLOB;
  v_lig    VARCHAR2(500);
  v_offset INTEGER := 1;
  v_amount INTEGER;
  v_buffer VARCHAR2(4000);
  v_len    INTEGER;
  v_nb     PLS_INTEGER := 0;

  -- Cache catégories
  TYPE t_cat IS TABLE OF ti_categories.libelle%TYPE INDEX BY PLS_INTEGER;
  v_cats t_cat;
BEGIN
  DBMS_LOB.CREATETEMPORARY(v_csv, TRUE);

  -- Charger le cache catégories
  -- ?

  -- En-tête CSV
  v_lig := 'REFERENCE,LIBELLE,PRIX_HT,STOCK,CATEGORIE' || CHR(10);
  DBMS_LOB.WRITEAPPEND(v_csv, LENGTH(v_lig), v_lig);

  -- Données
  FOR r IN (SELECT produit_id, reference, libelle, prix_ht, stock, cat_id
              FROM ti_produits WHERE actif = 'O' ORDER BY reference) LOOP
    -- ?
    v_nb := v_nb + 1;
  END LOOP;

  -- Afficher infos + premières lignes
  v_len := DBMS_LOB.GETLENGTH(v_csv);
  DBMS_OUTPUT.PUT_LINE('Lignes : ' || v_nb || ': Taille : ' || v_len || ' car.');
  -- Lire les 300 premiers caractères pour aperçu
  -- ?

  DBMS_LOB.FREETEMPORARY(v_csv);
EXCEPTION
  WHEN OTHERS THEN
    -- ?
    RAISE;
END;
/
```

---

## CONCEPT 7: Curseurs explicites et REF CURSOR

> Cycle de vie DECLARE/OPEN/FETCH/CLOSE, 4 attributs, REF CURSOR pour l'interopérabilité.

---

### 7-A: Cycle de vie complet sur TI_EMPLOYES

```sql
DECLARE
  CURSOR c_emp IS
    SELECT e.emp_id, e.nom, e.prenom, e.salaire, d.code AS dept
      FROM ti_employes    e
      JOIN ti_departements d ON d.dept_id = e.dept_id
     WHERE e.actif = 'O' ORDER BY e.salaire DESC;

  r c_emp%ROWTYPE;
BEGIN
  -- OPEN
  OPEN c_emp;
  DBMS_OUTPUT.PUT_LINE('Ouvert : ' || CASE WHEN c_emp%ISOPEN THEN 'OUI' ELSE 'NON' END);

  -- FETCH
  LOOP
    FETCH c_emp INTO r;
    EXIT WHEN c_emp%NOTFOUND;
    DBMS_OUTPUT.PUT_LINE(
      LPAD(c_emp%ROWCOUNT, 3) || '. ' ||
      RPAD(r.nom || ' ' || r.prenom, 25) ||
      LPAD(r.salaire, 10) || ' EUR  [' || r.dept || ']'
    );
  END LOOP;
  DBMS_OUTPUT.PUT_LINE('Total : ' || c_emp%ROWCOUNT);

  -- CLOSE
  CLOSE c_emp;

EXCEPTION
  WHEN OTHERS THEN
    IF c_emp%ISOPEN THEN CLOSE c_emp; END IF;
    RAISE;
END;
/
```

---

### 7-B: Les quatre attributs : observer leur évolution

```sql
DECLARE
  CURSOR c IS
    SELECT cmd_id, numero, montant_ttc FROM ti_commandes
     WHERE statut = 'NOUVEAU' ORDER BY date_cmd;
  r c%ROWTYPE;
BEGIN
  DBMS_OUTPUT.PUT_LINE('AVANT OPEN : %ISOPEN = ' || CASE WHEN c%ISOPEN THEN 'TRUE' ELSE 'FALSE' END);

  OPEN c;
  DBMS_OUTPUT.PUT_LINE('APRES OPEN : %ISOPEN=' ||
    CASE WHEN c%ISOPEN THEN 'TRUE' ELSE 'FALSE' END || '  %ROWCOUNT=' || c%ROWCOUNT);

  FETCH c INTO r;
  DBMS_OUTPUT.PUT_LINE('APRES 1er FETCH (' || r.numero || ') :');
  DBMS_OUTPUT.PUT_LINE('  %FOUND    = ' || CASE WHEN c%FOUND    THEN 'TRUE' ELSE 'FALSE' END);
  DBMS_OUTPUT.PUT_LINE('  %NOTFOUND = ' || CASE WHEN c%NOTFOUND THEN 'TRUE' ELSE 'FALSE' END);
  DBMS_OUTPUT.PUT_LINE('  %ROWCOUNT = ' || c%ROWCOUNT);

  LOOP FETCH c INTO r; EXIT WHEN c%NOTFOUND; END LOOP;
  DBMS_OUTPUT.PUT_LINE('APRES DERNIER FETCH : %NOTFOUND=' ||
    CASE WHEN c%NOTFOUND THEN 'TRUE' ELSE 'FALSE' END || '  %ROWCOUNT=' || c%ROWCOUNT);

  CLOSE c;
END;
/
```

---

### 7-C: Curseur paramétré: employés par département

```sql
DECLARE
  CURSOR c_emp (p_dept_id NUMBER, p_sal_min NUMBER DEFAULT 0) IS
    SELECT emp_id, nom, prenom, salaire, poste
      FROM ti_employes
     WHERE dept_id = p_dept_id AND salaire >= p_sal_min AND actif = 'O'
     ORDER BY salaire DESC;

  r c_emp%ROWTYPE;

  PROCEDURE afficher (p_id NUMBER, p_seuil NUMBER DEFAULT 0) IS
  BEGIN
    DBMS_OUTPUT.PUT_LINE('--- Dept ' || p_id || ' (sal >= ' || p_seuil || ') ---');
    OPEN c_emp(p_id, p_seuil);
    LOOP
      FETCH c_emp INTO r; EXIT WHEN c_emp%NOTFOUND;
      DBMS_OUTPUT.PUT_LINE('  ' || RPAD(r.nom || ' ' || r.prenom, 25) ||
        LPAD(r.salaire, 8) || ' EUR  ' || r.poste);
    END LOOP;
    DBMS_OUTPUT.PUT_LINE('  Total : ' || c_emp%ROWCOUNT || ' employé(s)');
    CLOSE c_emp;
  END;
BEGIN
  afficher(10);
  afficher(10, 3000);
  afficher(20, 2500);
END;
/
```

---

### 7-D: REF CURSOR: commandes d'un client

```sql
CREATE OR REPLACE PROCEDURE get_commandes_client (
  p_client_id IN  NUMBER,
  p_statut    IN  VARCHAR2 DEFAULT NULL,
  p_cur       OUT SYS_REFCURSOR
) IS
BEGIN
  IF p_statut IS NULL THEN
    OPEN p_cur FOR
      SELECT c.numero, c.date_cmd, c.statut, c.montant_ttc,
             e.nom || ' ' || e.prenom AS commercial
        FROM ti_commandes c
        LEFT JOIN ti_employes e ON e.emp_id = c.emp_id
       WHERE c.client_id = p_client_id ORDER BY c.date_cmd DESC;
  ELSE
    OPEN p_cur FOR
      SELECT c.numero, c.date_cmd, c.statut, c.montant_ttc,
             e.nom || ' ' || e.prenom AS commercial
        FROM ti_commandes c
        LEFT JOIN ti_employes e ON e.emp_id = c.emp_id
       WHERE c.client_id = p_client_id AND c.statut = p_statut
       ORDER BY c.date_cmd DESC;
  END IF;
END;
/

DECLARE
  v_cur    SYS_REFCURSOR;
  v_numero ti_commandes.numero%TYPE;
  v_date   ti_commandes.date_cmd%TYPE;
  v_statut ti_commandes.statut%TYPE;
  v_ttc    ti_commandes.montant_ttc%TYPE;
  v_comm   VARCHAR2(102);
  v_nb     PLS_INTEGER := 0;
BEGIN
  get_commandes_client(1, NULL, v_cur);
  LOOP
    FETCH v_cur INTO v_numero, v_date, v_statut, v_ttc, v_comm;
    EXIT WHEN v_cur%NOTFOUND;
    v_nb := v_nb + 1;
    DBMS_OUTPUT.PUT_LINE(
      RPAD(v_numero, 15) || TO_CHAR(v_date,'DD/MM/YY') || '  ' ||
      RPAD(v_statut, 12) || LPAD(TO_CHAR(v_ttc,'999G999D00'),12) || ' EUR  ' ||
      NVL(v_comm, '(sans commercial)')
    );
  END LOOP;
  DBMS_OUTPUT.PUT_LINE('--- ' || v_nb || ' commande(s) ---');
  CLOSE v_cur;
END;
/
```

---

### 7-E: Curseurs implicites après DML

```sql
DECLARE
BEGIN
  -- Annuler les commandes NOUVEAU sans lignes depuis > 7 jours
  UPDATE ti_commandes SET statut = 'ANNULE'
   WHERE statut = 'NOUVEAU' AND date_cmd < SYSDATE - 7
     AND cmd_id NOT IN (SELECT cmd_id FROM ti_lignes_cmd);

  DBMS_OUTPUT.PUT_LINE('Annulées : ' || SQL%ROWCOUNT || '  FOUND=' ||
    CASE WHEN SQL%FOUND THEN 'TRUE' ELSE 'FALSE' END);
  ROLLBACK;

  -- Passer EN_COURS les NOUVEAU avec lignes
  UPDATE ti_commandes SET statut = 'EN_COURS'
   WHERE statut = 'NOUVEAU'
     AND cmd_id IN (SELECT cmd_id FROM ti_lignes_cmd);

  DBMS_OUTPUT.PUT_LINE('En cours : ' || SQL%ROWCOUNT);
  ROLLBACK;
END;
/
```

---

### ✏ Mini-exercice 7

Créez `rechercher_produits(p_critere VARCHAR2, p_valeur VARCHAR2, p_cur OUT SYS_REFCURSOR)` :
- `'CAT'` → produits de la catégorie (code)
- `'PRIX_MAX'` → produits avec `prix_ht <= p_valeur`
- `'ALERTE'` → produits avec `stock <= stock_min`

Depuis un bloc de test, appelez les 3 variantes et affichez `référence | libellé | prix | stock`.

```sql
CREATE OR REPLACE PROCEDURE rechercher_produits (
  p_critere IN VARCHAR2,
  p_valeur  IN VARCHAR2 DEFAULT NULL,
  p_cur     OUT SYS_REFCURSOR
) IS
BEGIN
  CASE p_critere
    WHEN 'CAT' THEN
      OPEN p_cur FOR
        SELECT p.reference, p.libelle, p.prix_ht, p.stock
          FROM ti_produits p JOIN ti_categories c ON c.cat_id = p.cat_id
         WHERE c.code = p_valeur AND p.actif = 'O';
    WHEN 'PRIX_MAX' THEN
      -- ?
    WHEN 'ALERTE' THEN
      -- ?
    ELSE
      RAISE_APPLICATION_ERROR(-20001, 'Critère inconnu : ' || p_critere);
  END CASE;
END;
/

-- Bloc de test
DECLARE
  v_cur  SYS_REFCURSOR;
  v_ref  ti_produits.reference%TYPE;
  v_lib  ti_produits.libelle%TYPE;
  v_prix ti_produits.prix_ht%TYPE;
  v_stk  ti_produits.stock%TYPE;

  PROCEDURE afficher_resultats (p_titre VARCHAR2, p_c IN OUT SYS_REFCURSOR) IS
  BEGIN
    DBMS_OUTPUT.PUT_LINE('=== ' || p_titre || ' ===');
    LOOP
      FETCH p_c INTO v_ref, v_lib, v_prix, v_stk;
      EXIT WHEN p_c%NOTFOUND;
      DBMS_OUTPUT.PUT_LINE(RPAD(v_ref,15) || RPAD(v_lib,30) || LPAD(v_prix,8) || ' EUR  stock=' || v_stk);
    END LOOP;
    CLOSE p_c;
  END;
BEGIN
  rechercher_produits('ALERTE', NULL, v_cur);
  afficher_resultats('Produits en alerte', v_cur);

  rechercher_produits('PRIX_MAX', '500', v_cur);
  afficher_resultats('Prix <= 500 EUR', v_cur);
END;
/
```

---

## CONCEPT 8: BULK COLLECT et FORALL

> BULK COLLECT réduit les context switches. Toujours utiliser LIMIT.
> FORALL envoie le DML en un seul appel. SAVE EXCEPTIONS en production.

---

### 8-A: Visualiser le context switch

```sql
DECLARE
  CURSOR c IS SELECT * FROM ti_employes WHERE actif = 'O';
  r      ti_employes%ROWTYPE;
  v_nb   PLS_INTEGER := 0;

  TYPE t_emp IS TABLE OF ti_employes%ROWTYPE;
  v_emps t_emp;

  v_debut TIMESTAMP(6);
  v_fin   TIMESTAMP(6);
BEGIN
  v_debut := SYSTIMESTAMP;
  OPEN c;
  LOOP FETCH c INTO r; EXIT WHEN c%NOTFOUND; v_nb := v_nb + 1; END LOOP;
  CLOSE c;
  v_fin := SYSTIMESTAMP;
  DBMS_OUTPUT.PUT_LINE('Fetch individuel  : ' || (v_fin - v_debut) || ' (' || v_nb || ' switches)');

  v_nb := 0;
  v_debut := SYSTIMESTAMP;
  OPEN c;
  LOOP
    FETCH c BULK COLLECT INTO v_emps LIMIT 10;
    EXIT WHEN v_emps.COUNT = 0;
    v_nb := v_nb + v_emps.COUNT;
    v_emps.DELETE;
  END LOOP;
  CLOSE c;
  v_fin := SYSTIMESTAMP;
  DBMS_OUTPUT.PUT_LINE('BULK COLLECT x10  : ' || (v_fin - v_debut) || ' (~' || CEIL(v_nb/10) || ' switches)');
END;
/
```

---

### 8-B: BULK COLLECT + FORALL: historique des salaires

```sql
DECLARE
  TYPE t_emp_id  IS TABLE OF ti_employes.emp_id%TYPE;
  TYPE t_sal_old IS TABLE OF ti_employes.salaire%TYPE;
  TYPE t_sal_new IS TABLE OF ti_employes.salaire%TYPE;

  CURSOR c IS SELECT emp_id, salaire FROM ti_employes WHERE actif = 'O' AND dept_id = 10;

  v_ids     t_emp_id;
  v_sal_old t_sal_old;
  v_sal_new t_sal_new;

  c_limit  CONSTANT PLS_INTEGER := 5;
  v_lot    PLS_INTEGER := 0;
  v_total  PLS_INTEGER := 0;
BEGIN
  OPEN c;
  LOOP
    FETCH c BULK COLLECT INTO v_ids, v_sal_old LIMIT c_limit;
    EXIT WHEN v_ids.COUNT = 0;
    v_lot := v_lot + 1;

    -- Calculer nouveaux salaires (+5%)
    v_sal_new := t_sal_new();
    v_sal_new.EXTEND(v_ids.COUNT);
    FOR i IN 1..v_ids.COUNT LOOP
      v_sal_new(i) := ROUND(v_sal_old(i) * 1.05, 2);
    END LOOP;

    -- Insérer dans TI_HISTORIQUE_SAL en masse
    FORALL i IN 1..v_ids.COUNT
      INSERT INTO ti_historique_sal (hist_id, emp_id, ancien_salaire, nouveau_salaire, motif)
      VALUES (seq_audit.NEXTVAL, v_ids(i), v_sal_old(i), v_sal_new(i), 'Augmentation annuelle +5%');

    DBMS_OUTPUT.PUT_LINE('Lot ' || v_lot || ' : ' || v_ids.COUNT || ' ligne(s)');
    v_total := v_total + v_ids.COUNT;
    COMMIT;
    v_ids.DELETE; v_sal_old.DELETE; v_sal_new.DELETE;
  END LOOP;
  CLOSE c;

  DBMS_OUTPUT.PUT_LINE('Total : ' || v_total || ' employé(s) historisé(s)');
END;
/
```

---

### 8-C: FORALL INDICES OF: après suppression d'éléments

```sql
DECLARE
  TYPE t_prod_ids IS TABLE OF ti_produits.produit_id%TYPE;
  v_ids t_prod_ids;
BEGIN
  SELECT produit_id BULK COLLECT INTO v_ids
    FROM ti_produits WHERE stock <= stock_min AND actif = 'O';

  DBMS_OUTPUT.PUT_LINE('Produits en alerte : ' || v_ids.COUNT);

  -- Variante 1 : IN 1..COUNT (indices consécutifs)
  FORALL i IN 1..v_ids.COUNT
    UPDATE ti_produits SET stock = stock + 50 WHERE produit_id = v_ids(i);
  DBMS_OUTPUT.PUT_LINE('IN 1..COUNT : ' || SQL%ROWCOUNT || ' lignes');
  ROLLBACK;

  -- Variante 2 : INDICES OF (gère les trous)
  v_ids.DELETE(2);   -- crée un trou
  FORALL i IN INDICES OF v_ids
    UPDATE ti_produits SET stock = stock + 50 WHERE produit_id = v_ids(i);
  DBMS_OUTPUT.PUT_LINE('INDICES OF  : ' || SQL%ROWCOUNT || ' lignes (trou ignoré)');
  ROLLBACK;
END;
/
```

---

### 8-D: FORALL SAVE EXCEPTIONS: import de commandes

```sql
DECLARE
  TYPE t_numero IS TABLE OF ti_commandes.numero%TYPE;
  TYPE t_cli_id IS TABLE OF ti_commandes.client_id%TYPE;
  TYPE t_emp_id IS TABLE OF ti_commandes.emp_id%TYPE;

  -- Données avec erreurs : client 9999 n'existe pas
  v_nums t_numero := t_numero('CMD-X001','CMD-X002','CMD-X003','CMD-X004','CMD-X005');
  v_clis t_cli_id := t_cli_id(1,          2,         9999,       1,         2       );
  v_emps t_emp_id := t_emp_id(1,          1,         1,          NULL,      1       );

  e_bulk EXCEPTION;
  PRAGMA EXCEPTION_INIT(e_bulk, -24381);

  v_ok NUMBER := 0;
  v_ko NUMBER := 0;
BEGIN
  DBMS_OUTPUT.PUT_LINE('Import de ' || v_nums.COUNT || ' commandes :');

  FORALL i IN 1..v_nums.COUNT SAVE EXCEPTIONS
    INSERT INTO ti_commandes (cmd_id, numero, client_id, emp_id)
    VALUES (seq_commande.NEXTVAL, v_nums(i), v_clis(i), v_emps(i));

  v_ok := SQL%ROWCOUNT;
  DBMS_OUTPUT.PUT_LINE('Toutes réussies : ' || v_ok);

EXCEPTION
  WHEN e_bulk THEN
    v_ko := SQL%BULK_EXCEPTIONS.COUNT;
    v_ok := v_nums.COUNT - v_ko;
    DBMS_OUTPUT.PUT_LINE('Réussites : ' || v_ok || '  Échecs : ' || v_ko);
    FOR j IN 1..v_ko LOOP
      DBMS_OUTPUT.PUT_LINE(
        '  ERREUR iter ' || SQL%BULK_EXCEPTIONS(j).ERROR_INDEX ||
        ' (' || v_nums(SQL%BULK_EXCEPTIONS(j).ERROR_INDEX) || ')' ||
        ' ORA-' || SQL%BULK_EXCEPTIONS(j).ERROR_CODE
      );
    END LOOP;
    ROLLBACK;
END;
/
```

---

### ✏ Mini-exercice 8

Contexte : migration des produits actifs de `TI_PRODUITS` vers une table archive.

```sql
CREATE TABLE ti_produits_archive AS SELECT * FROM ti_produits WHERE 1=0;
ALTER TABLE ti_produits_archive ADD archive_le DATE;
/
```

Écrivez un bloc qui :
1. Charge les produits actifs avec `BULK COLLECT` (tous, sans LIMIT: table petite)
2. Insère dans `TI_PRODUITS_ARCHIVE` avec `FORALL SAVE EXCEPTIONS`
3. Affiche réussites et erreurs éventuelles
4. COMMIT si succès, ROLLBACK en cas d'erreur totale
5. `DROP TABLE ti_produits_archive;` à la fin

```sql
-- A VOUS
DECLARE
  TYPE t_prod IS TABLE OF ti_produits%ROWTYPE;
  v_prods t_prod;
  e_bulk  EXCEPTION;
  PRAGMA EXCEPTION_INIT(e_bulk, -24381);
BEGIN
  SELECT * BULK COLLECT INTO v_prods FROM ti_produits WHERE actif = 'O';
  DBMS_OUTPUT.PUT_LINE('Produits à archiver : ' || v_prods.COUNT);

  FORALL i IN 1..v_prods.COUNT SAVE EXCEPTIONS
    INSERT INTO ti_produits_archive
    VALUES (v_prods(i).produit_id, v_prods(i).reference, v_prods(i).libelle,
            v_prods(i).prix_ht, v_prods(i).stock, v_prods(i).stock_min,
            v_prods(i).cat_id, v_prods(i).fiche_technique, v_prods(i).actif, SYSDATE);

  -- ?
  COMMIT;
EXCEPTION
  WHEN e_bulk THEN
    -- ?
    ROLLBACK;
END;
/

SELECT produit_id, reference, archive_le FROM ti_produits_archive ORDER BY produit_id;
/
DROP TABLE ti_produits_archive;
/
```

---

## CONCEPT 9: Gestion des erreurs

> Exceptions prédéfinies, PRAGMA EXCEPTION_INIT, RAISE_APPLICATION_ERROR, pkg_errors.

---

### 9-A: Exceptions prédéfinies sur TechInfo

```sql
DECLARE
  v_client ti_clients%ROWTYPE;
BEGIN
  -- NO_DATA_FOUND
  BEGIN
    SELECT * INTO v_client FROM ti_clients WHERE client_id = 99999;
  EXCEPTION
    WHEN NO_DATA_FOUND THEN
      DBMS_OUTPUT.PUT_LINE('Client 99999 introuvable: traitement ignoré');
  END;

  -- TOO_MANY_ROWS
  BEGIN
    DECLARE v_nom ti_clients.nom%TYPE;
    BEGIN
      SELECT nom INTO v_nom FROM ti_clients WHERE actif = 'O';  -- plusieurs lignes
    END;
  EXCEPTION
    WHEN TOO_MANY_ROWS THEN
      DBMS_OUTPUT.PUT_LINE('Plusieurs clients actifs: utiliser un curseur');
  END;

  -- DUP_VAL_ON_INDEX
  BEGIN
    INSERT INTO ti_clients (client_id, code_client, nom) VALUES (1,'DUP','Test');
  EXCEPTION
    WHEN DUP_VAL_ON_INDEX THEN
      DBMS_OUTPUT.PUT_LINE('client_id=1 existe déjà: DUP_VAL_ON_INDEX');
  END;

  DBMS_OUTPUT.PUT_LINE('Tous les cas traités.');
END;
/
```

---

### 9-B: WHEN OTHERS : bon usage vs mauvais

```sql
-- MAUVAIS : WHEN OTHERS THEN NULL (interdit en production)
DECLARE v_client ti_clients%ROWTYPE;
BEGIN
  SELECT * INTO v_client FROM ti_clients WHERE client_id = 99999;
EXCEPTION
  WHEN OTHERS THEN NULL;   -- bug invisible !
END;
/

-- BON : log + RAISE
DECLARE v_client ti_clients%ROWTYPE;
BEGIN
  SELECT * INTO v_client FROM ti_clients WHERE client_id = 99999;
EXCEPTION
  WHEN NO_DATA_FOUND THEN
    DBMS_OUTPUT.PUT_LINE('Client absent: valeur par défaut');
  WHEN OTHERS THEN
    DBMS_OUTPUT.PUT_LINE('Inattendu : ' || SQLCODE || ': ' || SQLERRM);
    RAISE;   -- TOUJOURS
END;
/
```

**Règle d'or :** `WHEN OTHERS` → logguer `SQLCODE`+`SQLERRM` → `RAISE`

---

### 9-C: PRAGMA EXCEPTION_INIT: nommer les erreurs FK du schéma

```sql
DECLARE
  e_fk_emp_dept  EXCEPTION;
  PRAGMA EXCEPTION_INIT(e_fk_emp_dept, -2292);   -- ORA-02292 : enfants existants

  e_fk_cmd_cli   EXCEPTION;
  PRAGMA EXCEPTION_INIT(e_fk_cmd_cli,  -2291);   -- ORA-02291 : parent inexistant

  e_ck_violation EXCEPTION;
  PRAGMA EXCEPTION_INIT(e_ck_violation, -2290);  -- ORA-02290 : CHECK violé
BEGIN
  -- Test : supprimer un département qui a des employés
  BEGIN
    DELETE FROM ti_departements WHERE dept_id = 10;
  EXCEPTION
    WHEN e_fk_emp_dept THEN
      DBMS_OUTPUT.PUT_LINE('Dept 10 : des employés y sont rattachés');
  END;

  -- Test : insérer un employé avec salaire négatif
  BEGIN
    INSERT INTO ti_employes (emp_id, matricule, nom, prenom, salaire, dept_id)
    VALUES (seq_emp.NEXTVAL,'EMP-9999','TEST','Test',-500, 10);
  EXCEPTION
    WHEN e_ck_violation THEN
      DBMS_OUTPUT.PUT_LINE('Salaire négatif interdit: CHECK salaire > 0');
  END;
END;
/
```

---

### 9-D: RAISE_APPLICATION_ERROR et package pkg_errors

```sql
-- Package centralisé TechInfo
CREATE OR REPLACE PACKAGE pkg_errors AS
  C_EMP_INTROUVABLE     CONSTANT NUMBER := -20001;
  C_SAL_INVALIDE        CONSTANT NUMBER := -20002;
  C_DEPT_INTROUVABLE    CONSTANT NUMBER := -20003;
  C_CLIENT_INACTIF      CONSTANT NUMBER := -20004;
  C_STOCK_INSUFFISANT   CONSTANT NUMBER := -20010;
  C_CMD_STATUT_INVALIDE CONSTANT NUMBER := -20011;
END pkg_errors;
/

-- Procédure métier : créer une commande
CREATE OR REPLACE PROCEDURE passer_commande (
  p_client_id IN NUMBER,
  p_emp_id    IN NUMBER,
  p_numero    IN VARCHAR2
) IS
  v_actif ti_clients.actif%TYPE;
  v_nb    NUMBER;
BEGIN
  BEGIN
    SELECT actif INTO v_actif FROM ti_clients WHERE client_id = p_client_id;
  EXCEPTION
    WHEN NO_DATA_FOUND THEN
      RAISE_APPLICATION_ERROR(pkg_errors.C_EMP_INTROUVABLE,
        'Client ' || p_client_id || ' introuvable');
  END;

  IF v_actif = 'N' THEN
    RAISE_APPLICATION_ERROR(pkg_errors.C_CLIENT_INACTIF,
      'Client ' || p_client_id || ' inactif: commande refusée');
  END IF;

  SELECT COUNT(*) INTO v_nb FROM ti_employes WHERE emp_id = p_emp_id AND actif = 'O';
  IF v_nb = 0 THEN
    RAISE_APPLICATION_ERROR(pkg_errors.C_EMP_INTROUVABLE,
      'Commercial ' || p_emp_id || ' introuvable');
  END IF;

  INSERT INTO ti_commandes (cmd_id, numero, client_id, emp_id)
  VALUES (seq_commande.NEXTVAL, p_numero, p_client_id, p_emp_id);
  DBMS_OUTPUT.PUT_LINE('Commande ' || p_numero || ' créée.');
  ROLLBACK;
EXCEPTION
  WHEN OTHERS THEN
    DBMS_OUTPUT.PUT_LINE('Erreur : ' || SQLCODE || ': ' || SQLERRM);
    RAISE;
END;
/

-- Tests
BEGIN passer_commande(1,    1, 'CMD-TEST-001'); END; /   -- OK
BEGIN passer_commande(9999, 1, 'CMD-TEST-002'); END; /   -- -20001
BEGIN passer_commande(1, 9999, 'CMD-TEST-003'); END; /   -- -20001
```

---

### ✏ Mini-exercice 9

Créez `valider_produit(p_prod_id NUMBER, p_qte NUMBER)` qui :
1. Vérifie que le produit existe et est actif (sinon `pkg_errors.C_EMP_INTROUVABLE`)
2. Vérifie que `p_qte > 0` (sinon `pkg_errors.C_SAL_INVALIDE`)
3. Vérifie que `p_qte <= stock` (sinon `pkg_errors.C_STOCK_INSUFFISANT`)
4. Affiche : `"Commande validée : REF-001 x 5: stock restant : 10"`
5. Dans `WHEN OTHERS` : affiche le code + message, puis `RAISE`

Testez avec : produit 1 qté 3 (OK), produit 1 qté 999 (insuffisant), produit 99999 qté 1 (inexistant).

```sql
CREATE OR REPLACE PROCEDURE valider_produit (
  p_prod_id IN NUMBER,
  p_qte     IN NUMBER
) IS
  v_ref   ti_produits.reference%TYPE;
  v_stock ti_produits.stock%TYPE;
  v_actif ti_produits.actif%TYPE;
BEGIN
  -- Vérifier existence et statut
  -- ?

  -- Vérifier quantité > 0
  -- ?

  -- Vérifier stock suffisant
  -- ?

  -- Succès
  DBMS_OUTPUT.PUT_LINE('Validé : ' || v_ref || ' x ' || p_qte ||
    ': stock restant : ' || (v_stock - p_qte));
EXCEPTION
  WHEN OTHERS THEN
    -- ?
END;
/

BEGIN valider_produit(1,    3);  END; /
BEGIN valider_produit(1,  999);  END; /
BEGIN valider_produit(99999, 1); END; /
```

---

## EXERCICE INTÉGRÉ: Rapport de performance TechInfo

> **Durée estimée : 35 à 45 minutes**
> Combine les 9 concepts du Jour 1 avec le schéma TechInfo Solutions.

### Cahier des charges

```
Étape 1: Structures
  TYPE RECORD t_perf_dept :
    dept_code, dept_nom, nb_emp PLS_INTEGER,
    sal_total NUMBER(12,2), nb_cmds PLS_INTEGER, ca_total NUMBER(12,2)
  TYPE TABLE t_perfs IS TABLE OF t_perf_dept
  Associative Array t_dept_cache (dept_id → nom)

Étape 2: Chargement
  Curseur sur TI_DEPARTEMENTS actifs
  Pour chaque département :
    BULK COLLECT des salaires depuis TI_EMPLOYES
    Agrégat commandes depuis TI_COMMANDES + TI_EMPLOYES

Étape 3: Construction du CLOB
  En-tête : "RAPPORT PERFORMANCE: DD/MM/YYYY"
  Ligne par département : CODE | NOM | EMP | MASSE SAL. | CMDS | CA TTC
  Pied de page : totaux globaux + durée de traitement (SYSTIMESTAMP)

Étape 4: Erreurs
  WHEN OTHERS : FREETEMPORARY + fermeture curseur + RAISE

Étape 5: Affichage
  Lire le CLOB par chunks de 250 car.
```

```sql
-- A VOUS
DECLARE
  -- Étape 1 : structures
  TYPE t_perf_dept IS RECORD (
    dept_code  ti_departements.code%TYPE,
    dept_nom   ti_departements.nom%TYPE,
    nb_emp     PLS_INTEGER,
    sal_total  NUMBER(12,2),
    nb_cmds    PLS_INTEGER,
    ca_total   NUMBER(12,2)
  );
  TYPE t_perfs     IS TABLE OF t_perf_dept;
  TYPE t_dept_cache IS TABLE OF ti_departements.nom%TYPE INDEX BY PLS_INTEGER;

  v_perfs   t_perfs := t_perfs();
  v_cache   t_dept_cache;
  v_idx     PLS_INTEGER := 0;

  -- LOB
  v_rapport CLOB;
  v_lig     VARCHAR2(300);
  v_offset  INTEGER := 1;
  v_amount  INTEGER;
  v_buffer  VARCHAR2(4000);
  v_len     INTEGER;

  -- Timing
  v_debut   TIMESTAMP(6) := SYSTIMESTAMP;

  -- Curseur
  CURSOR c_depts IS
    SELECT dept_id, code, nom FROM ti_departements WHERE actif = 'O' ORDER BY code;

BEGIN
  DBMS_LOB.CREATETEMPORARY(v_rapport, TRUE);

  -- Charger le cache des noms de département
  FOR r IN (SELECT dept_id, nom FROM ti_departements) LOOP
    v_cache(r.dept_id) := r.nom;
  END LOOP;

  -- En-tête
  v_lig := 'RAPPORT PERFORMANCE TECHINFO SOLUTIONS' || CHR(10) ||
           TO_CHAR(SYSDATE,'DD/MM/YYYY HH24:MI') || CHR(10) ||
           RPAD('=',65,'=') || CHR(10) || CHR(10);
  DBMS_LOB.WRITEAPPEND(v_rapport, LENGTH(v_lig), v_lig);

  -- Parcourir les départements
  FOR r_dept IN c_depts LOOP
    DECLARE
      TYPE t_sal IS TABLE OF ti_employes.salaire%TYPE;
      v_sals   t_sal;
      v_nb_cmd NUMBER := 0;
      v_ca     NUMBER := 0;
    BEGIN
      -- BULK COLLECT des salaires
      SELECT salaire BULK COLLECT INTO v_sals
        FROM ti_employes WHERE dept_id = r_dept.dept_id AND actif = 'O';

      -- Agrégat commandes
      SELECT COUNT(DISTINCT c.cmd_id), NVL(SUM(c.montant_ttc), 0)
        INTO v_nb_cmd, v_ca
        FROM ti_commandes c
        JOIN ti_employes  e ON e.emp_id = c.emp_id
       WHERE e.dept_id = r_dept.dept_id;

      -- Remplir le RECORD de performance
      v_idx := v_idx + 1;
      v_perfs.EXTEND;
      v_perfs(v_idx).dept_code := r_dept.code;
      v_perfs(v_idx).dept_nom  := r_dept.nom;
      v_perfs(v_idx).nb_emp    := v_sals.COUNT;
      -- ? (calculer sal_total, nb_cmds, ca_total)
    END;
  END LOOP;

  -- En-tête du tableau
  v_lig := RPAD('CODE',8) || RPAD('DÉPARTEMENT',20) || RPAD('EMP',5) ||
           RPAD('MASSE SAL.',14) || RPAD('CMDS',6) || 'CA TTC' || CHR(10) ||
           RPAD('-',65,'-') || CHR(10);
  DBMS_LOB.WRITEAPPEND(v_rapport, LENGTH(v_lig), v_lig);

  -- Lignes et totaux
  DECLARE
    v_tot_emp  PLS_INTEGER := 0;
    v_tot_sal  NUMBER := 0;
    v_tot_cmd  PLS_INTEGER := 0;
    v_tot_ca   NUMBER := 0;
  BEGIN
    FOR i IN 1..v_perfs.COUNT LOOP
      -- Construire la ligne
      -- ?
      -- Accumuler les totaux
      -- ?
    END LOOP;

    -- Pied de page
    -- ?
  END;

  -- Durée
  v_lig := CHR(10) || 'Généré en ' ||
    ROUND(EXTRACT(SECOND FROM (SYSTIMESTAMP - v_debut)), 3) || 's' || CHR(10);
  DBMS_LOB.WRITEAPPEND(v_rapport, LENGTH(v_lig), v_lig);

  -- Affichage par chunks de 250 car.
  v_len := DBMS_LOB.GETLENGTH(v_rapport);
  DBMS_OUTPUT.PUT_LINE('Rapport : ' || v_len || ' car.');
  DBMS_OUTPUT.PUT_LINE('');
  WHILE v_offset <= v_len LOOP
    v_amount := LEAST(250, v_len - v_offset + 1);
    DBMS_LOB.READ(v_rapport, v_amount, v_offset, v_buffer);
    DBMS_OUTPUT.PUT(v_buffer);
    v_offset := v_offset + v_amount;
  END LOOP;

  DBMS_LOB.FREETEMPORARY(v_rapport);

EXCEPTION
  WHEN OTHERS THEN
    IF DBMS_LOB.ISTEMPORARY(v_rapport) = 1 THEN DBMS_LOB.FREETEMPORARY(v_rapport); END IF;
    IF c_depts%ISOPEN THEN CLOSE c_depts; END IF;
    DBMS_OUTPUT.PUT_LINE('Erreur : ' || SQLCODE || ': ' || SQLERRM);
    RAISE;
END;
/
```

---

## Récapitulatif: Ce que vous savez faire après le Jour 1

| Concept | Compétence acquise |
|---|---|
| Bloc PL/SQL | DECLARE/BEGIN/EXCEPTION, blocs imbriqués, SQLCODE/SQLERRM |
| Types avancés | TIMESTAMP, INTERVAL, BOOLEAN, PLS_INTEGER |
| %TYPE / %ROWTYPE | Ancrage sur le schéma TechInfo, %ROWTYPE sur curseur |
| RECORD | Structures multi-tables, passage en paramètre |
| Collections | Nested TABLE, VARRAY, Associative Array (cache) |
| LOBs | CLOB persistant (TI_PRODUITS), CLOB temporaire, FREETEMPORARY garanti |
| Curseurs | Cycle de vie, 4 attributs, paramétré, REF CURSOR pour TI_COMMANDES |
| BULK COLLECT | LIMIT obligatoire, pattern lot + COMMIT, libération PGA |
| FORALL | IN 1..COUNT, INDICES OF, SAVE EXCEPTIONS + SQL%BULK_EXCEPTIONS |
| Erreurs | Exceptions prédéfinies, PRAGMA, pkg_errors, WHEN OTHERS bon usage |

---

## Bonnes pratiques: Mémo

**Toujours :**
- `colonne%TYPE`: jamais de types en dur
- `LIMIT` avec chaque `BULK COLLECT`
- `CLOSE curseur` dans `EXCEPTION WHEN OTHERS`
- `FREETEMPORARY` dans `EXCEPTION` aussi
- `RAISE` après le log dans `WHEN OTHERS`

**Jamais :**
- `WHEN OTHERS THEN NULL`
- `BULK COLLECT` sans `LIMIT` sur une grande table
- Codes `-20xxx` en dur: utiliser `pkg_errors`

---

*Fin du support Jour 1: ORA-PLAV: TechInfo Solutions S.A.S*
*Jour 2 : Transactions autonomes · TI_JOURNAL_AUDIT · Triggers composés · Packages Oracle intégrés · SQL dynamique*
