# ORA-PLAV: Oracle PL/SQL Avancé
## Support etudiant: JOUR 2
### Schéma : TechInfo Solutions S.A.S

> **Activation** : `SET SERVEROUTPUT ON SIZE UNLIMITED`
> **Prérequis** : DDL/DML TechInfo chargé (`table_structure.sql` + données)
> **Révision Jour 1** : Démarrer par 10 min de révision orale (3 choses retenues)

---

## CONCEPT 1: Transactions autonomes

### Rappel théorique: Ce qu'il faut expliquer avant de coder

**Le problème de l'audit standard :**
Si une procédure logue un événement dans `TI_JOURNAL_AUDIT` dans la même transaction
que l'opération principale, et que cette transaction fait `ROLLBACK`,
**le log disparaît aussi**. En production, c'est inacceptable pour un journal d'audit.

**La solution : `PRAGMA AUTONOMOUS_TRANSACTION`**
Ce pragma dit à Oracle : "cette procédure ouvre sa propre transaction, indépendante
de la transaction parente. Son `COMMIT`/`ROLLBACK` n'affecte pas le parent."

```

**Règle absolue :** toute procédure `AUTONOMOUS_TRANSACTION` **doit** se terminer
par `COMMIT` ou `ROLLBACK`. Sinon : `ORA-06519`.

**Cas d'usage TechInfo :**
- Journaliser tout DML sur `TI_EMPLOYES` dans `TI_JOURNAL_AUDIT`
- Enregistrer les erreurs applicatives même quand la transaction principale échoue
- Compteurs persistants indépendants du traitement métier

---

### Exemple 1-A: Procédure d'audit autonome sur TI_JOURNAL_AUDIT

**Ce qu'on montre :**
1. Créer `pkg_audit` avec `PRAGMA AUTONOMOUS_TRANSACTION`
2. Prouver que le log survit au `ROLLBACK` de la transaction parente

```sql
-- ── Création du package d'audit autonome ─────────────────────────
CREATE OR REPLACE PACKAGE pkg_audit AS
  -- Journaliser un événement dans TI_JOURNAL_AUDIT
  -- PRAGMA AUTONOMOUS_TRANSACTION : le log est indépendant du contexte appelant
  PROCEDURE log_action (
    p_table_nom  IN VARCHAR2,
    p_action     IN VARCHAR2,
    p_old_val    IN VARCHAR2 DEFAULT NULL,
    p_new_val    IN VARCHAR2 DEFAULT NULL,
    p_details    IN VARCHAR2 DEFAULT NULL
  );
END pkg_audit;
/

CREATE OR REPLACE PACKAGE BODY pkg_audit AS

  PROCEDURE log_action (
    p_table_nom  IN VARCHAR2,
    p_action     IN VARCHAR2,
    p_old_val    IN VARCHAR2 DEFAULT NULL,
    p_new_val    IN VARCHAR2 DEFAULT NULL,
    p_details    IN VARCHAR2 DEFAULT NULL
  ) IS
    -- Ce pragma est la clé : ouvre une transaction séparée
    PRAGMA AUTONOMOUS_TRANSACTION;
  BEGIN
    INSERT INTO ti_journal_audit (
      log_id, table_nom, action, utilisateur,
      date_action, old_val, new_val, details
    ) VALUES (
      seq_audit.NEXTVAL,
      p_table_nom,
      p_action,
      SYS_CONTEXT('USERENV', 'SESSION_USER'),
      SYSTIMESTAMP,                              -- SYSTIMESTAMP, pas SYSDATE
      p_old_val,
      p_new_val,
      p_details
    );

    -- OBLIGATOIRE dans toute procédure autonome
    COMMIT;

  EXCEPTION
    WHEN OTHERS THEN
      -- En cas d'erreur dans l'audit lui-même : ROLLBACK de la transaction autonome
      ROLLBACK;
      -- Ne pas propager l'erreur d'audit vers la transaction parente
      -- (un échec du log ne doit pas bloquer l'opération principale)
      DBMS_OUTPUT.PUT_LINE('[pkg_audit.log_action] Erreur : ' || SQLERRM);
  END;

END pkg_audit;
/

-- ── Preuve : le log survit au ROLLBACK parent ────────────────────
DECLARE
  v_emp_id ti_employes.emp_id%TYPE := 1;
  v_sal_avant ti_employes.salaire%TYPE;
  v_sal_apres ti_employes.salaire%TYPE;
BEGIN
  SELECT salaire INTO v_sal_avant FROM ti_employes WHERE emp_id = v_emp_id;
  DBMS_OUTPUT.PUT_LINE('Salaire avant   : ' || v_sal_avant);

  -- Opération principale : augmentation de 20%
  UPDATE ti_employes SET salaire = salaire * 1.2 WHERE emp_id = v_emp_id;

  SELECT salaire INTO v_sal_apres FROM ti_employes WHERE emp_id = v_emp_id;
  DBMS_OUTPUT.PUT_LINE('Salaire pendant : ' || v_sal_apres);

  -- Appel du log autonome : s'exécute dans sa propre transaction
  pkg_audit.log_action(
    p_table_nom => 'TI_EMPLOYES',
    p_action    => 'UPDATE SALAIRE',
    p_old_val   => 'sal=' || v_sal_avant,
    p_new_val   => 'sal=' || v_sal_apres,
    p_details   => 'Augmentation 20% emp_id=' || v_emp_id
  );

  -- ROLLBACK de la transaction principale : annule l'UPDATE
  ROLLBACK;

  SELECT salaire INTO v_sal_apres FROM ti_employes WHERE emp_id = v_emp_id;
  DBMS_OUTPUT.PUT_LINE('Salaire apres ROLLBACK : ' || v_sal_apres || ' (restauré)');
END;
/

-- Vérifier que le log existe même après le ROLLBACK
SELECT log_id, table_nom, action, old_val, new_val, date_action
  FROM ti_journal_audit
 ORDER BY log_id DESC
 FETCH FIRST 3 ROWS ONLY;
/
```


---

### Exemple 1-B: Pattern de logging d'erreurs avec AUTONOMOUS_TRANSACTION

**Ce qu'on montre :** utiliser l'audit autonome pour enregistrer les erreurs,
même quand la procédure principale lève une exception.

```sql
-- Table de log d'erreurs (utilisation de TI_JOURNAL_AUDIT)
CREATE OR REPLACE PROCEDURE traiter_commande_batch (
  p_client_id IN NUMBER,
  p_montant   IN NUMBER
) IS
  v_nb NUMBER;
BEGIN
  -- Validation
  SELECT COUNT(*) INTO v_nb FROM ti_clients
   WHERE client_id = p_client_id AND actif = 'O';

  IF v_nb = 0 THEN
    -- Logger l'erreur avant de lever l'exception
    pkg_audit.log_action(
      p_table_nom => 'TI_COMMANDES',
      p_action    => 'ERREUR_VALIDATION',
      p_details   => 'Client ' || p_client_id || ' inactif ou inexistant'
    );
    RAISE_APPLICATION_ERROR(-20004, 'Client inactif : ' || p_client_id);
  END IF;

  -- Traitement normal
  INSERT INTO ti_commandes (cmd_id, numero, client_id, montant_ht)
  VALUES (seq_commande.NEXTVAL,
          'CMD-BATCH-' || TO_CHAR(SYSDATE,'YYYYMMDDHH24MI'),
          p_client_id, p_montant);

  pkg_audit.log_action(
    p_table_nom => 'TI_COMMANDES',
    p_action    => 'INSERT',
    p_new_val   => 'client=' || p_client_id || ' montant=' || p_montant
  );

  COMMIT;
  DBMS_OUTPUT.PUT_LINE('Commande créée pour client ' || p_client_id);

EXCEPTION
  WHEN OTHERS THEN
    ROLLBACK;
    DBMS_OUTPUT.PUT_LINE('Erreur : ' || SQLERRM);
    RAISE;
END;
/

-- Test avec un client valide et un client inexistant
BEGIN traiter_commande_batch(1, 1500); END;
/
BEGIN traiter_commande_batch(9999, 500); END;
/

-- Les deux tentatives sont loguées (succès ET erreur)
SELECT log_id, action, details, date_action
  FROM ti_journal_audit
 ORDER BY log_id DESC
 FETCH FIRST 5 ROWS ONLY;
/
```

---

## CONCEPT 2: SAVEPOINT

### Rappel théorique

**SAVEPOINT ≠ AUTONOMOUS_TRANSACTION :**
- `SAVEPOINT` = point de retour dans la **même** transaction
- `AUTONOMOUS_TRANSACTION` = **nouvelle** transaction indépendante

**Utilité des SAVEPOINTs :**
Quand un traitement se fait en plusieurs étapes et qu'une erreur à l'étape 3
ne doit pas annuler les étapes 1 et 2.

```
SAVEPOINT sp_etape1
  INSERT données de référence
SAVEPOINT sp_etape2
  INSERT données transactionnelles
  ← erreur ici → ROLLBACK TO sp_etape1 (annule étape 2 uniquement)
COMMIT (valide étape 1)
```

---

### Exemple 2-A: Import de commandes en plusieurs étapes

```sql
-- Simulation d'un import de commandes avec SAVEPOINTs
DECLARE
  v_cmd_id  NUMBER;
  v_err     VARCHAR2(200);
BEGIN
  -- Étape 1 : insérer l'en-tête de commande
  SELECT seq_commande.NEXTVAL INTO v_cmd_id FROM dual;

  INSERT INTO ti_commandes (cmd_id, numero, client_id, statut, montant_ht, montant_ttc)
  VALUES (v_cmd_id, 'CMD-IMP-' || v_cmd_id, 1, 'NOUVEAU', 0, 0);

  SAVEPOINT sp_entete;   -- point de retour après l'en-tête
  DBMS_OUTPUT.PUT_LINE('Étape 1 OK : cmd_id=' || v_cmd_id);

  -- Étape 2 : insérer les lignes de commande
  INSERT INTO ti_lignes_cmd (ligne_id, cmd_id, produit_id, quantite, prix_unitaire)
  VALUES (seq_ligne.NEXTVAL, v_cmd_id, 1, 2, 299.00);

  INSERT INTO ti_lignes_cmd (ligne_id, cmd_id, produit_id, quantite, prix_unitaire)
  VALUES (seq_ligne.NEXTVAL, v_cmd_id, 2, 1, 149.00);

  SAVEPOINT sp_lignes;   -- point de retour après les lignes
  DBMS_OUTPUT.PUT_LINE('Étape 2 OK : 2 lignes insérées');

  -- Étape 3 : mettre à jour le montant total (simulation d'une erreur)
  -- Simuler une erreur : produit_id=999 inexistant
  BEGIN
    INSERT INTO ti_lignes_cmd (ligne_id, cmd_id, produit_id, quantite, prix_unitaire)
    VALUES (seq_ligne.NEXTVAL, v_cmd_id, 99999, 1, 500.00);  -- FK violation

    SAVEPOINT sp_montant;
  EXCEPTION
    WHEN OTHERS THEN
      -- Erreur à l'étape 3 : revenir à l'état après l'étape 2
      ROLLBACK TO sp_lignes;
      DBMS_OUTPUT.PUT_LINE('Étape 3 KO (' || SQLERRM || ') → retour à sp_lignes');
  END;

  -- Mettre à jour le montant total avec ce qu'on a pu insérer
  UPDATE ti_commandes
     SET montant_ht  = (SELECT SUM(quantite * prix_unitaire) FROM ti_lignes_cmd WHERE cmd_id = v_cmd_id),
         montant_ttc = (SELECT SUM(quantite * prix_unitaire) * 1.2 FROM ti_lignes_cmd WHERE cmd_id = v_cmd_id)
   WHERE cmd_id = v_cmd_id;

  -- COMMIT : valide tout ce qui n'a pas été annulé
  COMMIT;
  DBMS_OUTPUT.PUT_LINE('Import terminé : cmd_id=' || v_cmd_id);

EXCEPTION
  WHEN OTHERS THEN
    ROLLBACK;  -- annule tout en cas d'erreur grave
    DBMS_OUTPUT.PUT_LINE('Erreur critique : ' || SQLERRM);
    RAISE;
END;
/

-- Vérifier le résultat
SELECT c.cmd_id, c.numero, c.montant_ht, c.montant_ttc,
       COUNT(l.ligne_id) nb_lignes
  FROM ti_commandes c
  JOIN ti_lignes_cmd l ON l.cmd_id = c.cmd_id
 WHERE c.numero LIKE 'CMD-IMP-%'
 GROUP BY c.cmd_id, c.numero, c.montant_ht, c.montant_ttc;
/
```

---

## CONCEPT 3: Droits DEFINER et INVOKER

### Rappel théorique

| | DEFINER (défaut) | INVOKER (`AUTHID CURRENT_USER`) |
|---|---|---|
| **Qui s'exécute ?** | Droits du créateur du programme | Droits de l'appelant courant |
| **Résolution des objets** | Dans le schéma du créateur | Dans le schéma de l'appelant |
| **Cas d'usage** | Applications multi-utilisateurs, ségrégation | Utilitaires génériques, multi-tenancy |
| **Sécurité** | L'appelant ne voit jamais les tables directement | L'appelant doit avoir ses propres droits |

**Cas TechInfo: DEFINER :**
`pkg_rh` est créé dans le schéma `APP_OWNER`. Les utilisateurs `USER1` et `USER2`
ont `EXECUTE ON pkg_rh` mais aucun `SELECT` direct sur `TI_EMPLOYES`.
Toutes les requêtes passent par le package.

---

### Exemple 3-A: DEFINER rights : ségrégation des accès

```sql
-- Procédure DEFINER (comportement par défaut)
-- Exécutée avec les droits du schéma qui l'a créée
CREATE OR REPLACE PROCEDURE get_masse_salariale_dept (
  p_dept_id  IN  NUMBER,
  p_total    OUT NUMBER,
  p_nb_emp   OUT NUMBER
) IS
  -- Accède à TI_EMPLOYES avec les droits du CRÉATEUR (APP_OWNER ou STAGIAIRE)
  -- L'appelant n'a pas besoin de SELECT sur TI_EMPLOYES
BEGIN
  SELECT SUM(salaire), COUNT(emp_id)
    INTO p_total, p_nb_emp
    FROM ti_employes
   WHERE dept_id = p_dept_id
     AND actif   = 'O';

  pkg_audit.log_action(
    p_table_nom => 'TI_EMPLOYES',
    p_action    => 'SELECT MASSE SAL',
    p_details   => 'dept_id=' || p_dept_id
  );
EXCEPTION
  WHEN OTHERS THEN
    DBMS_OUTPUT.PUT_LINE('Erreur : ' || SQLCODE || ': ' || SQLERRM);
    RAISE;
END;
/

-- Utilisation
DECLARE
  v_total  NUMBER;
  v_nb_emp NUMBER;
BEGIN
  -- L'appelant utilise la procédure sans accès direct à TI_EMPLOYES
  FOR r IN (SELECT dept_id, nom FROM ti_departements WHERE actif = 'O' ORDER BY code) LOOP
    get_masse_salariale_dept(r.dept_id, v_total, v_nb_emp);
    DBMS_OUTPUT.PUT_LINE(
      RPAD(r.nom, 20) ||
      LPAD(NVL(v_nb_emp, 0), 5) || ' emp.  ' ||
      LPAD(NVL(TO_CHAR(v_total,'999G999D00'),'0'), 12) || ' EUR'
    );
  END LOOP;
END;
/
```

---

### Exemple 3-B: INVOKER rights : utilitaire générique

```sql
-- Utilitaire INVOKER : s'adapte au schéma de l'appelant
-- Utile pour des outils de DBA ou des environnements multi-schémas
CREATE OR REPLACE PROCEDURE compter_objets_schema (
  p_type   IN  VARCHAR2 DEFAULT NULL,
  p_statut IN  VARCHAR2 DEFAULT 'VALID'
)
AUTHID CURRENT_USER   -- clé : droits de l'APPELANT
IS
  v_nb   NUMBER;
  v_sql  VARCHAR2(200);
BEGIN
  -- USER_OBJECTS est résolu dans le schéma de l'appelant
  -- (pas dans celui qui a créé la procédure)
  IF p_type IS NULL THEN
    SELECT COUNT(*) INTO v_nb FROM user_objects WHERE status = p_statut;
    DBMS_OUTPUT.PUT_LINE('Objets ' || p_statut || ' dans le schéma courant : ' || v_nb);
  ELSE
    SELECT COUNT(*) INTO v_nb FROM user_objects
     WHERE object_type = UPPER(p_type) AND status = p_statut;
    DBMS_OUTPUT.PUT_LINE(UPPER(p_type) || ' ' || p_statut || ' : ' || v_nb);
  END IF;

  -- Lister les objets invalides si demandé
  IF p_statut = 'INVALID' THEN
    DBMS_OUTPUT.PUT_LINE('Objets invalides :');
    FOR r IN (
      SELECT object_type, object_name, last_ddl_time
        FROM user_objects
       WHERE status = 'INVALID'
         AND (p_type IS NULL OR object_type = UPPER(p_type))
       ORDER BY object_type, object_name
    ) LOOP
      DBMS_OUTPUT.PUT_LINE('  ' || RPAD(r.object_type, 15) || r.object_name);
    END LOOP;
  END IF;
END;
/

-- Tests
BEGIN compter_objets_schema();                     END; /
BEGIN compter_objets_schema('PROCEDURE', 'VALID'); END; /
BEGIN compter_objets_schema(p_statut => 'INVALID'); END; /
```

---

## CONCEPT 4: Dictionnaire de données

### Rappel théorique: Les vues essentielles

| Vue | Contenu | Usage type |
|---|---|---|
| `USER_OBJECTS` | Tous les objets du schéma courant | Inventaire, vérifier `STATUS` |
| `USER_SOURCE` | Code source PL/SQL ligne par ligne | Retrouver du code perdu, diff |
| `USER_ERRORS` | Erreurs de compilation | Déboguer un objet `INVALID` |
| `USER_PROCEDURES` | Procédures et fonctions avec métadonnées | `AUTHID`, `DETERMINISTIC` |
| `USER_TRIGGERS` | Triggers du schéma | Statut, événement, table cible |
| `ALL_DEPENDENCIES` | Dépendances entre objets | Impact d'une modification |
| `USER_SEGMENTS` | Espace disque consommé | Tables et index volumineux |

---

### Exemple 4-A: Inventaire et maintenance des objets

```sql
-- ── 1. Lister tous les objets du schéma TechInfo ─────────────────
SELECT object_type, COUNT(*) nb
  FROM user_objects
 GROUP BY object_type
 ORDER BY object_type;
/

-- ── 2. Détecter les objets invalides ─────────────────────────────
SELECT object_type, object_name, status, last_ddl_time
  FROM user_objects
 WHERE status = 'INVALID'
 ORDER BY object_type, object_name;
-- Si des objets INVALID : ALTER PROCEDURE nom COMPILE;
/

-- ── 3. Voir le code source d'une procédure ────────────────────────
-- Utile si on a perdu le fichier source ou pour vérifier ce qui est en base
SELECT text
  FROM user_source
 WHERE name = 'PKG_AUDIT'
   AND type = 'PACKAGE BODY'
 ORDER BY line;
/

-- ── 4. Trouver toutes les dépendances d'un package ────────────────
-- Avant de modifier PKG_AUDIT, identifier ce qui dépend de lui
SELECT name, type, referenced_name, referenced_type
  FROM user_dependencies
 WHERE referenced_name = 'PKG_AUDIT'
 ORDER BY type, name;
/

-- ── 5. Voir les erreurs de compilation ────────────────────────────
-- Quand SHOW ERRORS ou SQL Developer indique des erreurs
SELECT line, position, text
  FROM user_errors
 WHERE name = 'PKG_AUDIT'
 ORDER BY sequence;
/

-- ── 6. Rechercher dans le code source ─────────────────────────────
-- Trouver tous les objets qui utilisent TI_EMPLOYES
SELECT DISTINCT name, type
  FROM user_source
 WHERE UPPER(text) LIKE '%TI_EMPLOYES%'
 ORDER BY type, name;
/
```

---

### Exemple 4-B: Script de recompilation des objets invalides

```sql
-- Générer les commandes de recompilation dans l'ordre correct
DECLARE
  v_sql VARCHAR2(200);
BEGIN
  -- D'abord les PACKAGE SPEC, puis PACKAGE BODY, puis PROCEDURE/FUNCTION
  FOR r IN (
    SELECT object_type, object_name
      FROM user_objects
     WHERE status = 'INVALID'
       AND object_type IN ('PACKAGE', 'PACKAGE BODY',
                           'PROCEDURE', 'FUNCTION', 'TRIGGER')
     ORDER BY CASE object_type
                WHEN 'PACKAGE'      THEN 1
                WHEN 'PACKAGE BODY' THEN 2
                WHEN 'PROCEDURE'    THEN 3
                WHEN 'FUNCTION'     THEN 4
                WHEN 'TRIGGER'      THEN 5
              END,
              object_name
  ) LOOP
    v_sql := 'ALTER ' || r.object_type || ' "' || r.object_name || '" COMPILE';
    DBMS_OUTPUT.PUT_LINE(v_sql || ';');

    BEGIN
      EXECUTE IMMEDIATE v_sql;
      DBMS_OUTPUT.PUT_LINE('  => OK');
    EXCEPTION
      WHEN OTHERS THEN
        DBMS_OUTPUT.PUT_LINE('  => ERREUR : ' || SQLERRM);
    END;
  END LOOP;

  -- Vérification finale
  DECLARE v_nb NUMBER;
  BEGIN
    SELECT COUNT(*) INTO v_nb FROM user_objects WHERE status = 'INVALID';
    DBMS_OUTPUT.PUT_LINE('');
    DBMS_OUTPUT.PUT_LINE('Objets invalides restants : ' || v_nb);
  END;
END;
/
```

---

## CONCEPT 5: Récursivité

### Rappel théorique

**Quand utiliser la récursivité PL/SQL ?**
- Logique trop complexe pour `CONNECT BY` (traitements métier à chaque niveau)
- La récursivité PL/SQL ouvre un curseur implicite à chaque niveau → surveiller `OPEN_CURSORS`
- Toujours ajouter un garde-fou contre la récursion infinie

**Alternative recommandée pour les hiérarchies SQL :**
`CONNECT BY PRIOR` est nativement optimisé par Oracle et ne risque pas de stack overflow.

---

### Exemple 5-A: Hiérarchie managériale dans TI_EMPLOYES

```sql
-- ── Approche 1 : CONNECT BY (recommandée pour les hiérarchies SQL) ──
-- Parcourir la hiérarchie manager → subordonnés
SELECT LPAD(' ', (LEVEL - 1) * 3) || e.nom || ' ' || e.prenom AS hierarchie,
       e.poste,
       e.salaire,
       LEVEL AS profondeur
  FROM ti_employes e
 WHERE actif = 'O'
 START WITH e.mgr_id IS NULL          -- sommet de la hiérarchie
 CONNECT BY PRIOR e.emp_id = e.mgr_id -- parent → enfants
 ORDER SIBLINGS BY e.nom;
/

-- ── Approche 2 : récursivité PL/SQL (quand la logique est complexe) ──
CREATE OR REPLACE FUNCTION get_equipe_manager (
  p_mgr_id    IN NUMBER,
  p_niveau    IN NUMBER DEFAULT 0,
  p_max_niv   IN NUMBER DEFAULT 5    -- garde-fou contre la récursion infinie
) RETURN VARCHAR2
IS
  v_resultat  VARCHAR2(4000) := '';
  v_nom_mgr   ti_employes.nom%TYPE;
BEGIN
  -- Garde-fou : arrêter si profondeur max atteinte
  IF p_niveau > p_max_niv THEN
    RETURN '  [Profondeur max atteinte]' || CHR(10);
  END IF;

  -- Parcourir les subordonnés directs
  FOR r IN (
    SELECT emp_id, nom, prenom, poste, salaire
      FROM ti_employes
     WHERE mgr_id = p_mgr_id AND actif = 'O'
     ORDER BY nom
  ) LOOP
    -- Ligne de cet employé avec indentation selon le niveau
    v_resultat := v_resultat ||
      LPAD(' ', p_niveau * 3) || '|-- ' ||
      r.nom || ' ' || r.prenom ||
      ' [' || NVL(r.poste, '?') || '] ' ||
      r.salaire || ' EUR' || CHR(10);

    -- Appel récursif pour les subordonnés de cet employé
    v_resultat := v_resultat ||
      get_equipe_manager(r.emp_id, p_niveau + 1, p_max_niv);
  END LOOP;

  RETURN v_resultat;
END;
/

-- Afficher la hiérarchie complète depuis le sommet
DECLARE
  v_hier VARCHAR2(4000);
  v_sommet_id ti_employes.emp_id%TYPE;
BEGIN
  -- Trouver le sommet (sans manager)
  SELECT emp_id INTO v_sommet_id
    FROM ti_employes
   WHERE mgr_id IS NULL AND actif = 'O'
     AND ROWNUM = 1;

  DBMS_OUTPUT.PUT_LINE('=== Organigramme TechInfo Solutions ===');
  v_hier := get_equipe_manager(v_sommet_id);
  DBMS_OUTPUT.PUT(v_hier);
END;
/
```

---

### Exemple 5-B: Calcul récursif : masse salariale cumulative

```sql
-- Calculer la masse salariale d'un manager + toute son équipe
CREATE OR REPLACE FUNCTION masse_salariale_equipe (
  p_mgr_id  IN NUMBER,
  p_niveau  IN NUMBER DEFAULT 0
) RETURN NUMBER
IS
  v_total NUMBER := 0;
  v_sal_propre ti_employes.salaire%TYPE := 0;
BEGIN
  -- Sécurité anti-boucle infinie
  IF p_niveau > 10 THEN RETURN 0; END IF;

  -- Salaire propre du manager
  BEGIN
    SELECT salaire INTO v_sal_propre
      FROM ti_employes WHERE emp_id = p_mgr_id AND actif = 'O';
    v_total := v_sal_propre;
  EXCEPTION
    WHEN NO_DATA_FOUND THEN v_total := 0;
  END;

  -- Additionner récursivement la masse de chaque subordonné
  FOR r IN (SELECT emp_id FROM ti_employes WHERE mgr_id = p_mgr_id AND actif = 'O') LOOP
    v_total := v_total + masse_salariale_equipe(r.emp_id, p_niveau + 1);
  END LOOP;

  RETURN v_total;
END;
/

-- Afficher la masse salariale par manager de premier niveau
BEGIN
  DBMS_OUTPUT.PUT_LINE('=== Masse salariale par équipe ===');
  FOR r IN (
    SELECT emp_id, nom, prenom, poste FROM ti_employes
     WHERE mgr_id IS NULL OR mgr_id IN (
       SELECT emp_id FROM ti_employes WHERE mgr_id IS NULL
     )
       AND actif = 'O'
     ORDER BY nom
  ) LOOP
    DBMS_OUTPUT.PUT_LINE(
      RPAD(r.nom || ' ' || r.prenom, 25) ||
      RPAD(NVL(r.poste,'?'), 15) ||
      TO_CHAR(masse_salariale_equipe(r.emp_id), '999G999D00') || ' EUR'
    );
  END LOOP;
END;
/
```

---

## CONCEPT 6: Fonctions PIPELINED

### Rappel théorique

**Fonction ordinaire vs PIPELINED :**

```
Fonction ordinaire :          Fonction PIPELINED :
  1. Calculer TOUT            1. Pour chaque ligne :
  2. Stocker en RAM              a. Calculer la ligne
  3. Retourner d'un coup         b. PIPE ROW → envoyé immédiatement
                                 c. Continuer
                              2. L'appelant SQL reçoit en flux
```

**Avantages :**
- Consommation mémoire constante (pas d'accumulation)
- L'appelant peut commencer à traiter avant la fin de la fonction
- Utilisable dans `FROM`, `JOIN`, `WHERE` comme une table

**Prérequis TechInfo :** les types `T_PRODUIT_OBJ` et `T_PRODUIT_TAB` sont déjà
définis dans `table_structure.sql`.

---

### Exemple 6-A: Fonction pipelined sur TI_PRODUITS avec logique métier

```sql
-- Les types sont déjà définis dans table_structure.sql :
-- T_PRODUIT_OBJ (produit_id, reference, libelle, prix_ht, stock, categorie)
-- T_PRODUIT_TAB TABLE OF T_PRODUIT_OBJ

-- ── Variante 1 : tous les produits actifs avec catégorie ──────────
CREATE OR REPLACE FUNCTION get_produits_actifs
  RETURN t_produit_tab
  PIPELINED   -- mot clé : active le mode streaming
IS
BEGIN
  FOR r IN (
    SELECT p.produit_id, p.reference, p.libelle, p.prix_ht,
           p.stock, c.libelle AS cat_libelle
      FROM ti_produits   p
      JOIN ti_categories c ON c.cat_id = p.cat_id
     WHERE p.actif = 'O'
     ORDER BY c.libelle, p.reference
  ) LOOP
    -- PIPE ROW : envoie cette ligne au flux et continue immédiatement
    -- L'appelant SQL la reçoit sans attendre la fin de la boucle
    PIPE ROW(t_produit_obj(
      r.produit_id, r.reference, r.libelle,
      r.prix_ht, r.stock, r.cat_libelle
    ));
  END LOOP;

  RETURN;  -- fin du flux (pas de valeur: c'est différent d'une fonction normale)
END;
/

-- ── Utilisation en SQL ────────────────────────────────────────────
-- C'est l'avantage principal : utilisable directement dans un SELECT
SELECT p.reference, p.libelle, p.prix_ht, p.stock, p.categorie
  FROM TABLE(get_produits_actifs()) p   -- TABLE() déploie la fonction en jeu de résultats
 WHERE p.prix_ht < 500
 ORDER BY p.prix_ht DESC;
/

-- Joindre avec une vraie table
SELECT p.reference, p.libelle, p.prix_ht,
       l.quantite, l.prix_unitaire,
       l.quantite * l.prix_unitaire AS montant_ligne
  FROM TABLE(get_produits_actifs()) p
  JOIN ti_lignes_cmd l ON l.produit_id = p.produit_id
 WHERE l.cmd_id = 1;
/
```

---

### Exemple 6-B: Fonction pipelined avec filtre paramétré

```sql
-- Variante 2 : avec catégorie ou seuil de stock optionnels
CREATE OR REPLACE FUNCTION get_produits_filtres (
  p_cat_code     IN VARCHAR2 DEFAULT NULL,
  p_stock_max    IN NUMBER   DEFAULT NULL,
  p_prix_max     IN NUMBER   DEFAULT NULL
)
  RETURN t_produit_tab
  PIPELINED
IS
BEGIN
  FOR r IN (
    SELECT p.produit_id, p.reference, p.libelle, p.prix_ht,
           p.stock, c.libelle AS cat_libelle
      FROM ti_produits   p
      JOIN ti_categories c ON c.cat_id = p.cat_id
     WHERE p.actif = 'O'
       -- Filtres optionnels : IS NULL = pas de filtre
       AND (p_cat_code  IS NULL OR c.code   = p_cat_code)
       AND (p_stock_max IS NULL OR p.stock  <= p_stock_max)
       AND (p_prix_max  IS NULL OR p.prix_ht <= p_prix_max)
     ORDER BY p.reference
  ) LOOP
    PIPE ROW(t_produit_obj(
      r.produit_id, r.reference, r.libelle,
      r.prix_ht, r.stock, r.cat_libelle
    ));
  END LOOP;
  RETURN;
END;
/

-- Utilisation avec différents filtres
-- Produits en alerte de stock (stock <= 5)
SELECT p.reference, p.stock, p.categorie
  FROM TABLE(get_produits_filtres(p_stock_max => 5)) p
 ORDER BY p.stock;
/

-- Produits informatiques à moins de 1000 EUR
SELECT p.reference, p.prix_ht, p.stock
  FROM TABLE(get_produits_filtres(p_cat_code => 'INFO', p_prix_max => 1000)) p
 ORDER BY p.prix_ht;
/
```

---

### Exemple 6-C: Fonction pipelined avec calcul analytique

```sql
-- Générer un rapport de lignes de commandes enrichi
-- Inclut des calculs impossibles à faire en SQL pur : logique PL/SQL + streaming
CREATE OR REPLACE TYPE t_ligne_rapport AS OBJECT (
  cmd_numero    VARCHAR2(20),
  produit_ref   VARCHAR2(20),
  quantite      NUMBER,
  prix_unitaire NUMBER,
  montant_ligne NUMBER,
  pct_commande  NUMBER   -- % du montant de cette ligne / total de la commande
);
/
CREATE OR REPLACE TYPE t_lignes_rapport_tab AS TABLE OF t_ligne_rapport;
/

CREATE OR REPLACE FUNCTION get_rapport_lignes (p_cmd_id IN NUMBER DEFAULT NULL)
  RETURN t_lignes_rapport_tab
  PIPELINED
IS
  v_total_cmd NUMBER;
BEGIN
  -- Pour chaque commande (ou une seule si p_cmd_id précisé)
  FOR r_cmd IN (
    SELECT c.cmd_id, c.numero, c.montant_ht
      FROM ti_commandes c
     WHERE (p_cmd_id IS NULL OR c.cmd_id = p_cmd_id)
       AND c.montant_ht > 0
     ORDER BY c.date_cmd DESC
  ) LOOP
    v_total_cmd := r_cmd.montant_ht;

    -- Lignes de cette commande avec pourcentage calculé en PL/SQL
    FOR r_lig IN (
      SELECT l.quantite, l.prix_unitaire, p.reference
        FROM ti_lignes_cmd l
        JOIN ti_produits   p ON p.produit_id = l.produit_id
       WHERE l.cmd_id = r_cmd.cmd_id
    ) LOOP
      PIPE ROW(t_ligne_rapport(
        r_cmd.numero,
        r_lig.reference,
        r_lig.quantite,
        r_lig.prix_unitaire,
        r_lig.quantite * r_lig.prix_unitaire,
        -- Calcul PL/SQL : impossible à faire avec une simple vue
        CASE WHEN v_total_cmd > 0
             THEN ROUND(r_lig.quantite * r_lig.prix_unitaire / v_total_cmd * 100, 1)
             ELSE 0
        END
      ));
    END LOOP;
  END LOOP;
  RETURN;
END;
/

-- Consommer comme une table ordinaire
SELECT cmd_numero, produit_ref, quantite,
       TO_CHAR(prix_unitaire,'999G999D00') AS p_u,
       TO_CHAR(montant_ligne,'999G999D00') AS montant,
       pct_commande || '%' AS pct
  FROM TABLE(get_rapport_lignes(1))
 ORDER BY montant_ligne DESC;
/
```

---

## CONCEPT 7: Surcharge et DETERMINISTIC

### Rappel théorique

**Surcharge (Overloading) :**
- Même nom, signatures différentes: **uniquement dans les packages**
- Oracle choisit la version selon le type des paramètres à l'appel
- Impossible de distinguer par le type de retour seul

**DETERMINISTIC :**
- Même entrée → toujours même sortie
- Oracle peut cacher le résultat dans une requête SQL
- Obligatoire pour les Function-Based Indexes

---

### Exemple 7-A: Surcharge dans pkg_format sur TechInfo

```sql
CREATE OR REPLACE PACKAGE pkg_format AS
  -- Trois versions de "formater" selon le type
  FUNCTION formater (p_val IN DATE)     RETURN VARCHAR2;
  FUNCTION formater (p_val IN NUMBER)   RETURN VARCHAR2;
  FUNCTION formater (p_val IN VARCHAR2) RETURN VARCHAR2;
  FUNCTION formater (p_val IN TIMESTAMP) RETURN VARCHAR2;  -- 4ème version
END pkg_format;
/

CREATE OR REPLACE PACKAGE BODY pkg_format AS

  FUNCTION formater (p_val IN DATE) RETURN VARCHAR2 IS
  BEGIN
    RETURN TO_CHAR(p_val, 'DD/MM/YYYY HH24:MI');
  END;

  FUNCTION formater (p_val IN NUMBER) RETURN VARCHAR2 IS
  BEGIN
    -- Format monétaire avec séparateur de milliers
    RETURN TO_CHAR(p_val, 'FM999G999G999D00') || ' EUR';
  END;

  FUNCTION formater (p_val IN VARCHAR2) RETURN VARCHAR2 IS
  BEGIN
    RETURN INITCAP(TRIM(p_val));
  END;

  FUNCTION formater (p_val IN TIMESTAMP) RETURN VARCHAR2 IS
  BEGIN
    -- Inclut les microsecondes pour les journaux d'audit
    RETURN TO_CHAR(p_val, 'DD/MM/YYYY HH24:MI:SS.FF3');
  END;

END pkg_format;
/

-- Utilisation : Oracle choisit automatiquement la bonne version
BEGIN
  DBMS_OUTPUT.PUT_LINE(pkg_format.formater(SYSDATE));               -- DATE
  DBMS_OUTPUT.PUT_LINE(pkg_format.formater(SYSTIMESTAMP));           -- TIMESTAMP
  DBMS_OUTPUT.PUT_LINE(pkg_format.formater(1500.75));                -- NUMBER
  DBMS_OUTPUT.PUT_LINE(pkg_format.formater('  techinfo solutions ')); -- VARCHAR2
END;
/

-- Utilisation dans un SELECT sur TI_EMPLOYES
SELECT e.nom || ' ' || e.prenom   AS employe,
       pkg_format.formater(e.salaire)       AS salaire_fmt,
       pkg_format.formater(e.date_embauche) AS embauche_fmt
  FROM ti_employes e WHERE actif = 'O' ORDER BY e.nom;
/
```

---

### Exemple 7-B: DETERMINISTIC sur les calculs sans accès à la base

```sql
-- Fonctions pures : même entrée → même sortie, pas d'accès DB
-- DETERMINISTIC permet à Oracle de les mettre en cache dans une requête SQL

CREATE OR REPLACE FUNCTION calc_tva (
  p_montant_ht IN NUMBER,
  p_taux       IN NUMBER DEFAULT 0.20
) RETURN NUMBER
DETERMINISTIC   -- indique qu'Oracle peut cacher le résultat
IS
BEGIN
  RETURN ROUND(p_montant_ht * p_taux, 2);
END;
/

CREATE OR REPLACE FUNCTION calc_ttc (
  p_montant_ht IN NUMBER,
  p_taux       IN NUMBER DEFAULT 0.20
) RETURN NUMBER
DETERMINISTIC
IS
BEGIN
  RETURN ROUND(p_montant_ht * (1 + p_taux), 2);
END;
/

-- Utilisation dans des requêtes SQL
-- Oracle appelle la fonction UNE seule fois par valeur distincte grâce à DETERMINISTIC
SELECT c.numero,
       c.montant_ht                             AS ht,
       calc_tva(c.montant_ht)                   AS tva_20,
       calc_ttc(c.montant_ht)                   AS ttc_calculé,
       c.montant_ttc                             AS ttc_stocké,
       calc_ttc(c.montant_ht) - c.montant_ttc   AS ecart
  FROM ti_commandes c
 WHERE ABS(calc_ttc(c.montant_ht) - c.montant_ttc) > 0.01
 ORDER BY ABS(calc_ttc(c.montant_ht) - c.montant_ttc) DESC;
/
```

---

## CONCEPT 8: RESULT_CACHE

### Rappel théorique

| | RESULT_CACHE | DETERMINISTIC | Associative Array |
|---|---|---|---|
| **Stockage** | SGA (partagé entre sessions) | Local à la requête SQL | Session courante |
| **Durée** | Survit entre sessions | Durée d'une requête SQL | Durée de la session |
| **Invalidation** | Auto sur DML (RELIES_ON) | N/A | Manuelle |
| **Cas d'usage** | Référentiels peu changeants | Calcul pur dans SELECT | Cache manuel de session |

---

### Exemple 8-A: RESULT_CACHE sur TI_PARAMETRES

```sql
-- Vérifier que RESULT_CACHE est activé
SELECT value FROM v$parameter WHERE name = 'result_cache_max_size';
/

-- Fonction avec RESULT_CACHE : mise en cache dans la SGA
-- Idéale pour les paramètres de configuration peu changeants
CREATE OR REPLACE FUNCTION get_parametre (
  p_cle IN VARCHAR2
) RETURN VARCHAR2
RESULT_CACHE RELIES_ON (ti_parametres)
-- RELIES_ON : invalide le cache si TI_PARAMETRES est modifié
IS
  v_val ti_parametres.valeur%TYPE;
BEGIN
  SELECT valeur INTO v_val
    FROM ti_parametres WHERE cle = p_cle;
  RETURN v_val;
EXCEPTION
  WHEN NO_DATA_FOUND THEN RETURN NULL;
END;
/

-- Préparer quelques paramètres
MERGE INTO ti_parametres USING dual ON (cle = 'TVA_STANDARD')
  WHEN MATCHED    THEN UPDATE SET valeur = '0.20'
  WHEN NOT MATCHED THEN INSERT (cle, valeur, description)
                        VALUES ('TVA_STANDARD','0.20','Taux TVA standard');

MERGE INTO ti_parametres USING dual ON (cle = 'DELAI_LIVRAISON_J')
  WHEN MATCHED    THEN UPDATE SET valeur = '5'
  WHEN NOT MATCHED THEN INSERT (cle, valeur, description)
                        VALUES ('DELAI_LIVRAISON_J','5','Délai livraison standard en jours');
COMMIT;
/

-- Premier appel : Oracle lit dans TI_PARAMETRES et met en cache
BEGIN
  DBMS_OUTPUT.PUT_LINE('TVA          : ' || get_parametre('TVA_STANDARD'));
  DBMS_OUTPUT.PUT_LINE('Délai livr.  : ' || get_parametre('DELAI_LIVRAISON_J') || ' j.');
END;
/

-- Appels suivants : Oracle sert depuis le cache SGA (0 accès disque)
-- Surveiller le cache
SELECT name, status, scan_count, invalidations,
       object_count, space_overhead
  FROM v$result_cache_objects
 WHERE name LIKE '%GET_PARAMETRE%';
/
```

---

## CONCEPT 9: Triggers composés (COMPOUND TRIGGER)

### Rappel théorique

**Problèmes résolus par le COMPOUND TRIGGER :**
1. **Mutating table** (`ORA-04091`) : un trigger `BEFORE EACH ROW` essaie
   de lire la table en cours de modification.
2. **Variables partagées** entre sections : stocker des données de `BEFORE EACH ROW`
   pour les utiliser dans `AFTER STATEMENT`.

**Les 4 sections et leurs rôles :**

```
COMPOUND TRIGGER trg_emp
  FOR INSERT OR UPDATE OF salaire ON ti_employes

  -- Section déclaration : variables persistantes pendant tout le DML
  v_ids    t_ids  := t_ids();     -- accumuler les IDs
  v_count  NUMBER := 0;

  BEFORE EACH ROW : validation ligne par ligne (accès à :NEW, :OLD)
  AFTER  EACH ROW : accumulation dans les collections
  AFTER  STATEMENT : traitement global (accès à la DB sans mutating table)
```

---

### Exemple 9-A: Trigger composé : audit des salaires dans TI_HISTORIQUE_SAL

```sql
-- Créer une collection type pour accumuler les données dans le trigger
CREATE OR REPLACE TYPE t_emp_sal_rec AS OBJECT (
  emp_id      NUMBER,
  ancien_sal  NUMBER,
  nouveau_sal NUMBER,
  motif       VARCHAR2(200)
);
/
CREATE OR REPLACE TYPE t_emp_sal_tab AS TABLE OF t_emp_sal_rec;
/

CREATE OR REPLACE TRIGGER trg_emp_sal_compound
  FOR INSERT OR UPDATE OF salaire ON ti_employes
  COMPOUND TRIGGER

  -- Variables partagées entre toutes les sections
  -- Persistent pendant toute la durée de l'opération DML
  v_historique  t_emp_sal_tab := t_emp_sal_tab();
  v_count       PLS_INTEGER := 0;

  -- ── BEFORE EACH ROW : validation avant chaque ligne ─────────────
  BEFORE EACH ROW IS
  BEGIN
    -- Vérification : le nouveau salaire doit être positif
    IF :NEW.salaire <= 0 THEN
      RAISE_APPLICATION_ERROR(-20002,
        'Salaire invalide : ' || :NEW.salaire ||
        '. Doit être > 0. Employé : ' || :NEW.emp_id);
    END IF;

    -- Vérification : pas plus de +50% d'augmentation d'un coup
    IF INSERTING THEN NULL;  -- pas de contrôle à l'insertion
    ELSIF :NEW.salaire > :OLD.salaire * 1.5 THEN
      RAISE_APPLICATION_ERROR(-20002,
        'Augmentation de ' ||
        ROUND((:NEW.salaire - :OLD.salaire) / :OLD.salaire * 100) ||
        '% trop importante (max 50%). Employé : ' || :NEW.emp_id);
    END IF;
  END BEFORE EACH ROW;

  -- ── AFTER EACH ROW : accumuler pour l'historique ─────────────────
  AFTER EACH ROW IS
  BEGIN
    v_count := v_count + 1;

    -- Stocker dans la collection (pas de requête sur ti_employes ici)
    v_historique.EXTEND;
    v_historique(v_count) := t_emp_sal_rec(
      :NEW.emp_id,
      CASE WHEN INSERTING THEN NULL ELSE :OLD.salaire END,
      :NEW.salaire,
      CASE WHEN INSERTING THEN 'Création employé'
           ELSE 'Mise à jour salaire'
      END
    );
  END AFTER EACH ROW;

  -- ── AFTER STATEMENT : insérer l'historique en masse ─────────────
  AFTER STATEMENT IS
  BEGIN
    -- Ici : accès DB possible sans risque de mutating table
    -- (l'opération DML sur TI_EMPLOYES est terminée)
    IF v_count > 0 THEN
      FORALL i IN 1..v_historique.COUNT
        INSERT INTO ti_historique_sal (
          hist_id, emp_id, ancien_salaire, nouveau_salaire, motif
        ) VALUES (
          seq_audit.NEXTVAL,
          v_historique(i).emp_id,
          v_historique(i).ancien_sal,
          v_historique(i).nouveau_sal,
          v_historique(i).motif
        );

      pkg_audit.log_action(
        p_table_nom => 'TI_EMPLOYES',
        p_action    => 'SALARY_CHANGE',
        p_details   => v_count || ' salaire(s) modifié(s)'
      );
    END IF;
  END AFTER STATEMENT;

END trg_emp_sal_compound;
/

-- Test : modifier plusieurs salaires d'un coup
UPDATE ti_employes SET salaire = salaire * 1.05 WHERE dept_id = 10 AND actif = 'O';
COMMIT;
/

-- Vérifier l'historique
SELECT h.emp_id, e.nom, h.ancien_salaire, h.nouveau_salaire,
       ROUND((h.nouveau_salaire - h.ancien_salaire) / h.ancien_salaire * 100, 1) AS pct,
       h.motif
  FROM ti_historique_sal h
  JOIN ti_employes e ON e.emp_id = h.emp_id
 ORDER BY h.hist_id DESC;
/

-- Test de validation : augmentation trop importante
BEGIN
  UPDATE ti_employes SET salaire = salaire * 2 WHERE emp_id = 1;
EXCEPTION
  WHEN OTHERS THEN DBMS_OUTPUT.PUT_LINE('Erreur attendue : ' || SQLERRM);
END;
/
```

---

## CONCEPT 10: Triggers DDL et système

### Rappel théorique

**Triggers DDL :** déclenchés sur `CREATE`, `ALTER`, `DROP`, `TRUNCATE`.
Fonctions disponibles : `ORA_SYSEVENT`, `ORA_DICT_OBJ_TYPE`, `ORA_DICT_OBJ_NAME`.

**Risque :** un trigger DDL qui plante **bloque l'opération DDL**.
Toujours protéger avec `EXCEPTION WHEN OTHERS` et une transaction autonome.

---

### Exemple 10-A: Audit des modifications DDL sur le schéma TechInfo

```sql
-- Trigger DDL : journaliser tout CREATE/ALTER/DROP dans TI_JOURNAL_AUDIT
CREATE OR REPLACE TRIGGER trg_audit_ddl
  AFTER DDL ON SCHEMA   -- se déclenche sur tous les DDL du schéma courant
DECLARE
  PRAGMA AUTONOMOUS_TRANSACTION;  -- le log ne doit pas être annulé si le DDL échoue
BEGIN
  -- Enregistrer l'événement DDL
  INSERT INTO ti_journal_audit (
    log_id, table_nom, action, utilisateur, date_action, details
  ) VALUES (
    seq_audit.NEXTVAL,
    ORA_DICT_OBJ_TYPE || ':' || ORA_DICT_OBJ_NAME,  -- ex : TABLE:TI_PRODUITS
    ORA_SYSEVENT,                                     -- CREATE, ALTER, DROP...
    SYS_CONTEXT('USERENV', 'SESSION_USER'),
    SYSTIMESTAMP,
    'DDL sur ' || ORA_DICT_OBJ_OWNER || '.' || ORA_DICT_OBJ_NAME
  );
  COMMIT;  -- obligatoire avec AUTONOMOUS_TRANSACTION
EXCEPTION
  WHEN OTHERS THEN
    -- Ne pas bloquer le DDL si l'audit échoue
    ROLLBACK;
END;
/

-- Tester : créer et supprimer une table temporaire
CREATE TABLE ti_test_ddl_audit (id NUMBER, val VARCHAR2(50));
/
ALTER TABLE ti_test_ddl_audit ADD (date_creation DATE DEFAULT SYSDATE);
/
DROP TABLE ti_test_ddl_audit;
/

-- Voir les événements DDL enregistrés
SELECT log_id, table_nom, action, utilisateur, date_action, details
  FROM ti_journal_audit
 WHERE action IN ('CREATE', 'ALTER', 'DROP')
 ORDER BY log_id DESC
 FETCH FIRST 5 ROWS ONLY;
/
```

---

### Exemple 10-B: Ordonnancement des triggers et FOLLOWS/PRECEDES

```sql
-- Deux triggers sur TI_EMPLOYES : contrôler l'ordre d'exécution
-- Trigger 1 : validation du poste
CREATE OR REPLACE TRIGGER trg_emp_valider_poste
  BEFORE INSERT OR UPDATE ON ti_employes
  FOR EACH ROW
BEGIN
  IF :NEW.poste IS NOT NULL THEN
    :NEW.poste := UPPER(TRIM(:NEW.poste));
  END IF;
END;
/

-- Trigger 2 : validation du matricule: doit s'exécuter APRÈS le trigger 1
CREATE OR REPLACE TRIGGER trg_emp_valider_matricule
  BEFORE INSERT OR UPDATE ON ti_employes
  FOR EACH ROW
  FOLLOWS trg_emp_valider_poste   -- exécuté APRÈS trg_emp_valider_poste
BEGIN
  IF :NEW.matricule IS NULL THEN
    :NEW.matricule := 'EMP-' || LPAD(seq_emp.NEXTVAL, 4, '0');
  END IF;

  IF :NEW.matricule NOT LIKE 'EMP-%' THEN
    RAISE_APPLICATION_ERROR(-20005,
      'Matricule invalide : ' || :NEW.matricule ||
      '. Format attendu : EMP-NNNN');
  END IF;
END;
/

-- Vérifier l'ordre des triggers
SELECT trigger_name, trigger_type, triggering_event, status
  FROM user_triggers
 WHERE table_name = 'TI_EMPLOYES'
 ORDER BY trigger_name;
/
```

---

## CONCEPT 11: Packages Oracle intégrés

### Exemple 11-A: DBMS_SCHEDULER : planifier la purge de TI_JOURNAL_AUDIT

```sql
-- Procédure métier à planifier
CREATE OR REPLACE PROCEDURE purger_journal_audit (
  p_retention_jours IN NUMBER DEFAULT 90
) IS
  v_nb_avant  NUMBER;
  v_nb_apres  NUMBER;
  v_cutoff    TIMESTAMP := SYSTIMESTAMP - NUMTODSINTERVAL(p_retention_jours, 'DAY');
BEGIN
  SELECT COUNT(*) INTO v_nb_avant FROM ti_journal_audit;

  DELETE FROM ti_journal_audit
   WHERE date_action < v_cutoff;

  v_nb_apres := v_nb_avant - SQL%ROWCOUNT;
  COMMIT;

  DBMS_OUTPUT.PUT_LINE(
    'Purge journal audit : ' || SQL%ROWCOUNT ||
    ' ligne(s) supprimée(s). ' || v_nb_apres || ' restante(s).'
  );

  -- Logger la purge elle-même (avec autonome)
  pkg_audit.log_action(
    p_table_nom => 'TI_JOURNAL_AUDIT',
    p_action    => 'PURGE',
    p_details   => 'Rétention ' || p_retention_jours || 'j. ' ||
                   SQL%ROWCOUNT || ' lignes supprimées'
  );
END;
/

-- Créer le job DBMS_SCHEDULER
BEGIN
  -- Supprimer si existe déjà
  BEGIN
    DBMS_SCHEDULER.DROP_JOB('JOB_PURGE_JOURNAL', force => TRUE);
  EXCEPTION WHEN OTHERS THEN NULL;
  END;

  DBMS_SCHEDULER.CREATE_JOB(
    job_name        => 'JOB_PURGE_JOURNAL',
    job_type        => 'STORED_PROCEDURE',
    job_action      => 'PURGER_JOURNAL_AUDIT',
    start_date      => SYSTIMESTAMP + NUMTODSINTERVAL(1, 'MINUTE'),  -- dans 1 min
    repeat_interval => 'FREQ=DAILY;BYHOUR=2;BYMINUTE=0;BYSECOND=0', -- tous les jours à 2h
    enabled         => TRUE,
    comments        => 'Purge quotidienne du journal d''audit TechInfo'
  );
  DBMS_OUTPUT.PUT_LINE('Job créé : JOB_PURGE_JOURNAL');
END;
/

-- Tester manuellement
BEGIN
  DBMS_SCHEDULER.RUN_JOB('JOB_PURGE_JOURNAL', use_current_session => TRUE);
END;
/

-- Surveiller le job
SELECT job_name, status, start_date, repeat_interval, next_run_date
  FROM user_scheduler_jobs
 WHERE job_name = 'JOB_PURGE_JOURNAL';
/

-- Historique d'exécution
SELECT log_id, job_name, status, actual_start_date, run_duration, error#
  FROM user_scheduler_job_log
 WHERE job_name = 'JOB_PURGE_JOURNAL'
 ORDER BY log_id DESC;
/
```

---

### Exemple 11-B: UTL_FILE : exporter TI_COMMANDES en CSV

```sql
-- Prérequis (exécuter en SYSDBA) :
-- CREATE OR REPLACE DIRECTORY EXPORT_DIR AS '/tmp';
-- GRANT READ, WRITE ON DIRECTORY EXPORT_DIR TO STAGIAIRE;

CREATE OR REPLACE PROCEDURE exporter_commandes_csv (
  p_dir       IN VARCHAR2 DEFAULT 'EXPORT_DIR',
  p_fichier   IN VARCHAR2 DEFAULT 'commandes_export.csv',
  p_statut    IN VARCHAR2 DEFAULT NULL
) IS
  v_file    UTL_FILE.FILE_TYPE;
  v_ligne   VARCHAR2(1000);
  v_nb      PLS_INTEGER := 0;
BEGIN
  -- Ouvrir le fichier en écriture
  -- Mode 'w' = write, MAX_LINESIZE = 1000
  v_file := UTL_FILE.FOPEN(p_dir, p_fichier, 'w', 1000);

  -- En-tête CSV
  UTL_FILE.PUT_LINE(v_file, 'CMD_ID;NUMERO;CLIENT;DATE_CMD;STATUT;MONTANT_HT;MONTANT_TTC');

  -- Données
  FOR r IN (
    SELECT c.cmd_id, c.numero, cl.nom AS client,
           c.date_cmd, c.statut, c.montant_ht, c.montant_ttc
      FROM ti_commandes c
      JOIN ti_clients   cl ON cl.client_id = c.client_id
     WHERE (p_statut IS NULL OR c.statut = p_statut)
     ORDER BY c.date_cmd
  ) LOOP
    v_ligne := r.cmd_id || ';' ||
               r.numero || ';' ||
               r.client || ';' ||
               TO_CHAR(r.date_cmd, 'DD/MM/YYYY') || ';' ||
               r.statut || ';' ||
               TO_CHAR(r.montant_ht, 'FM999999D99') || ';' ||
               TO_CHAR(r.montant_ttc, 'FM999999D99');
    UTL_FILE.PUT_LINE(v_file, v_ligne);
    v_nb := v_nb + 1;
  END LOOP;

  -- Toujours fermer le fichier dans le flux normal
  UTL_FILE.FCLOSE(v_file);
  DBMS_OUTPUT.PUT_LINE(v_nb || ' commande(s) exportée(s) dans ' || p_fichier);

EXCEPTION
  WHEN UTL_FILE.INVALID_PATH THEN
    DBMS_OUTPUT.PUT_LINE('Répertoire invalide : ' || p_dir);
    DBMS_OUTPUT.PUT_LINE('Créer : CREATE DIRECTORY ' || p_dir || ' AS ''/chemin/os'';');
  WHEN OTHERS THEN
    -- Fermer le fichier en cas d'erreur
    IF UTL_FILE.IS_OPEN(v_file) THEN UTL_FILE.FCLOSE(v_file); END IF;
    DBMS_OUTPUT.PUT_LINE('Erreur export : ' || SQLCODE || ': ' || SQLERRM);
    RAISE;
END;
/

BEGIN exporter_commandes_csv(); END;
/
```

---

### Exemple 11-C: DBMS_METADATA : extraire le DDL des tables TechInfo

```sql
-- Extraire le DDL d'un objet (utile pour les scripts de migration)
DECLARE
  v_ddl CLOB;
BEGIN
  -- Configurer le formatage de la sortie
  DBMS_METADATA.SET_TRANSFORM_PARAM(
    DBMS_METADATA.SESSION_TRANSFORM, 'SQLTERMINATOR', TRUE
  );
  DBMS_METADATA.SET_TRANSFORM_PARAM(
    DBMS_METADATA.SESSION_TRANSFORM, 'PRETTY', TRUE
  );

  -- Extraire le DDL de la table TI_EMPLOYES
  v_ddl := DBMS_METADATA.GET_DDL('TABLE', 'TI_EMPLOYES', USER);
  DBMS_OUTPUT.PUT_LINE('=== DDL TI_EMPLOYES ===');
  DBMS_OUTPUT.PUT_LINE(DBMS_LOB.SUBSTR(v_ddl, 4000, 1));
END;
/

-- Générer le script complet du schéma TechInfo
DECLARE
  v_ddl CLOB;
BEGIN
  DBMS_OUTPUT.PUT_LINE('-- Script DDL complet du schéma TechInfo');
  DBMS_OUTPUT.PUT_LINE('-- Généré le ' || TO_CHAR(SYSDATE,'DD/MM/YYYY HH24:MI'));
  DBMS_OUTPUT.PUT_LINE('');

  FOR r IN (
    SELECT object_name, object_type
      FROM user_objects
     WHERE object_type IN ('TABLE', 'SEQUENCE', 'VIEW', 'PROCEDURE', 'PACKAGE')
       AND object_name LIKE 'TI_%' OR object_name LIKE 'SEQ_%'
     ORDER BY CASE object_type
               WHEN 'SEQUENCE'  THEN 1
               WHEN 'TABLE'     THEN 2
               WHEN 'VIEW'      THEN 3
               WHEN 'PROCEDURE' THEN 4
               WHEN 'PACKAGE'   THEN 5
             END, object_name
  ) LOOP
    BEGIN
      v_ddl := DBMS_METADATA.GET_DDL(r.object_type, r.object_name, USER);
      DBMS_OUTPUT.PUT_LINE(DBMS_LOB.SUBSTR(v_ddl, 32767, 1));
    EXCEPTION
      WHEN OTHERS THEN
        DBMS_OUTPUT.PUT_LINE('-- ERREUR pour ' || r.object_type || ' ' || r.object_name);
    END;
  END LOOP;
END;
/
```

---

## CONCEPT 12: SQL dynamique : EXECUTE IMMEDIATE

### Rappel théorique

**Quand est-ce indispensable ?**
- DDL (`CREATE`, `ALTER`, `DROP`) : impossible en SQL statique dans PL/SQL
- Nom de table ou colonne dynamique
- Construction conditionnelle de la clause `WHERE`

**Règle d'or :** toujours des **bind variables** pour les valeurs.
Ne jamais concaténer des valeurs utilisateur directement dans le SQL dynamique
→ injection SQL + fragmentation du shared pool.

```sql
-- MAUVAIS (injection SQL possible) :
EXECUTE IMMEDIATE 'SELECT * FROM ti_employes WHERE nom = ''' || p_nom || '''';

-- BON (bind variable) :
EXECUTE IMMEDIATE 'SELECT COUNT(*) FROM ti_employes WHERE nom = :n' USING p_nom;
```

---

### Exemple 12-A: DDL dynamique et validation du nom de table

```sql
-- Procédure utilitaire : créer une table d'archive dynamiquement
CREATE OR REPLACE PROCEDURE creer_table_archive (
  p_table_source IN VARCHAR2,
  p_suffix       IN VARCHAR2 DEFAULT '_ARCHIVE'
) IS
  v_table_dest VARCHAR2(50);
  v_nb         NUMBER;
  v_sql        VARCHAR2(500);
BEGIN
  -- Validation : la table source existe-t-elle ?
  SELECT COUNT(*) INTO v_nb FROM user_objects
   WHERE object_name = UPPER(p_table_source) AND object_type = 'TABLE';

  IF v_nb = 0 THEN
    RAISE_APPLICATION_ERROR(-20001,
      'Table source introuvable : ' || UPPER(p_table_source));
  END IF;

  v_table_dest := UPPER(p_table_source) || UPPER(p_suffix);

  -- Supprimer si elle existe déjà
  SELECT COUNT(*) INTO v_nb FROM user_objects
   WHERE object_name = v_table_dest AND object_type = 'TABLE';

  IF v_nb > 0 THEN
    -- DDL : impossible sans EXECUTE IMMEDIATE
    EXECUTE IMMEDIATE 'DROP TABLE ' || v_table_dest || ' CASCADE CONSTRAINTS PURGE';
    DBMS_OUTPUT.PUT_LINE('Table existante supprimée : ' || v_table_dest);
  END IF;

  -- Créer la table d'archive avec la structure de la source
  v_sql := 'CREATE TABLE ' || v_table_dest ||
           ' AS SELECT * FROM ' || UPPER(p_table_source) || ' WHERE 1=0';

  EXECUTE IMMEDIATE v_sql;
  DBMS_OUTPUT.PUT_LINE('Table créée : ' || v_table_dest);

  pkg_audit.log_action(
    p_table_nom => v_table_dest,
    p_action    => 'CREATE',
    p_details   => 'Archive de ' || UPPER(p_table_source)
  );
END;
/

-- Test
BEGIN creer_table_archive('TI_EMPLOYES'); END;
/
BEGIN creer_table_archive('TABLE_INEXISTANTE'); END;   -- erreur gérée
/
```

---

### Exemple 12-B: DML dynamique avec bind variables

```sql
-- Mise à jour dynamique : la colonne à mettre à jour est un paramètre
-- (impossible en SQL statique)
CREATE OR REPLACE PROCEDURE maj_colonne_employe (
  p_emp_id    IN NUMBER,
  p_colonne   IN VARCHAR2,   -- nom de colonne dynamique
  p_valeur    IN VARCHAR2
) IS
  v_colonnes_autorisees CONSTANT VARCHAR2(200) :=
    ':POSTE:EMAIL:TELEPHONE:';   -- liste blanche
  v_sql  VARCHAR2(500);
  v_nb   NUMBER;
BEGIN
  -- Validation de la colonne (liste blanche obligatoire)
  IF INSTR(v_colonnes_autorisees, ':' || UPPER(p_colonne) || ':') = 0 THEN
    RAISE_APPLICATION_ERROR(-20001,
      'Colonne non autorisée : ' || p_colonne ||
      '. Colonnes autorisées : POSTE, EMAIL, TELEPHONE');
  END IF;

  -- Vérifier que l'employé existe
  SELECT COUNT(*) INTO v_nb FROM ti_employes WHERE emp_id = p_emp_id;
  IF v_nb = 0 THEN
    RAISE_APPLICATION_ERROR(pkg_errors.C_EMP_INTROUVABLE,
      'Employé ' || p_emp_id || ' introuvable');
  END IF;

  -- SQL dynamique avec bind variable pour la VALEUR (pas le nom de colonne)
  -- Le nom de colonne doit être concaténé (pas de bind possible pour les identifiants)
  v_sql := 'UPDATE ti_employes SET ' || UPPER(p_colonne) ||
           ' = :val WHERE emp_id = :id';

  EXECUTE IMMEDIATE v_sql USING p_valeur, p_emp_id;

  DBMS_OUTPUT.PUT_LINE(
    'Employé ' || p_emp_id || ' : ' || UPPER(p_colonne) || ' = ' || p_valeur
  );
  COMMIT;
END;
/

-- Tests
BEGIN maj_colonne_employe(1, 'poste', 'DIRECTEUR TECHNIQUE'); END;
/
BEGIN maj_colonne_employe(1, 'email', 'dt@techinfo.fr'); END;
/
BEGIN maj_colonne_employe(1, 'salaire', '9999'); END;   -- colonne non autorisée
/
```

---

### Exemple 12-C: SELECT dynamique avec REF CURSOR

```sql
-- Recherche multi-critères dynamique sur TI_EMPLOYES
-- La clause WHERE est construite selon les paramètres fournis
CREATE OR REPLACE PROCEDURE rechercher_employes_dyn (
  p_nom_like  IN VARCHAR2 DEFAULT NULL,
  p_dept_id   IN NUMBER   DEFAULT NULL,
  p_sal_min   IN NUMBER   DEFAULT NULL,
  p_sal_max   IN NUMBER   DEFAULT NULL,
  p_cur       OUT SYS_REFCURSOR
) IS
  v_sql     VARCHAR2(2000);
  v_where   VARCHAR2(1000) := ' WHERE actif = ''O''';
BEGIN
  -- Construire la clause WHERE dynamiquement
  IF p_nom_like IS NOT NULL THEN
    v_where := v_where || ' AND (nom LIKE :nom OR prenom LIKE :nom)';
  END IF;
  IF p_dept_id IS NOT NULL THEN
    v_where := v_where || ' AND dept_id = :dept';
  END IF;
  IF p_sal_min IS NOT NULL THEN
    v_where := v_where || ' AND salaire >= :sal_min';
  END IF;
  IF p_sal_max IS NOT NULL THEN
    v_where := v_where || ' AND salaire <= :sal_max';
  END IF;

  v_sql := 'SELECT emp_id, nom, prenom, poste, salaire, dept_id '
        || 'FROM ti_employes' || v_where
        || ' ORDER BY salaire DESC';

  -- Ouvrir le REF CURSOR avec EXECUTE IMMEDIATE implicite
  -- USING passe les bind variables dans l'ordre de leur apparition dans v_where
  CASE
    WHEN p_nom_like IS NOT NULL AND p_dept_id IS NOT NULL
     AND p_sal_min IS NOT NULL AND p_sal_max IS NOT NULL THEN
      OPEN p_cur FOR v_sql USING p_nom_like, p_nom_like, p_dept_id, p_sal_min, p_sal_max;
    WHEN p_nom_like IS NOT NULL AND p_dept_id IS NOT NULL
     AND p_sal_min IS NOT NULL THEN
      OPEN p_cur FOR v_sql USING p_nom_like, p_nom_like, p_dept_id, p_sal_min;
    WHEN p_nom_like IS NOT NULL THEN
      OPEN p_cur FOR v_sql USING p_nom_like, p_nom_like;
    WHEN p_dept_id IS NOT NULL AND p_sal_min IS NOT NULL THEN
      OPEN p_cur FOR v_sql USING p_dept_id, p_sal_min;
    WHEN p_dept_id IS NOT NULL THEN
      OPEN p_cur FOR v_sql USING p_dept_id;
    ELSE
      OPEN p_cur FOR v_sql;  -- pas de paramètres
  END CASE;
END;
/

-- Tests
DECLARE
  v_cur    SYS_REFCURSOR;
  v_emp_id NUMBER; v_nom VARCHAR2(50); v_prenom VARCHAR2(50);
  v_poste  VARCHAR2(50); v_sal NUMBER; v_dept NUMBER;
  v_nb     PLS_INTEGER := 0;

  PROCEDURE lire (p_titre VARCHAR2) IS
  BEGIN
    DBMS_OUTPUT.PUT_LINE('=== ' || p_titre || ' ===');
    v_nb := 0;
    LOOP
      FETCH v_cur INTO v_emp_id, v_nom, v_prenom, v_poste, v_sal, v_dept;
      EXIT WHEN v_cur%NOTFOUND;
      v_nb := v_nb + 1;
      DBMS_OUTPUT.PUT_LINE('  ' || RPAD(v_nom || ' ' || v_prenom, 25) || v_sal || ' EUR');
    END LOOP;
    DBMS_OUTPUT.PUT_LINE('  → ' || v_nb || ' résultat(s)');
    CLOSE v_cur;
  END;
BEGIN
  rechercher_employes_dyn(p_cur => v_cur);
  lire('Tous les employés actifs');

  rechercher_employes_dyn(p_dept_id => 10, p_cur => v_cur);
  lire('Département 10');

  rechercher_employes_dyn(p_sal_min => 3000, p_sal_max => 5000, p_cur => v_cur);
  lire('Salaire 3000-5000 EUR');
END;
/
```

---

## EXERCICE INTÉGRÉ: Architecture complète TechInfo Jour 2

> **Durée estimée : 40 à 50 minutes**
> Combine tous les concepts du Jour 2 dans une architecture cohérente.

### Objectif

Construire le **package `pkg_commandes_v2`**: version améliorée avec :
- Transactions autonomes (audit dans `TI_JOURNAL_AUDIT`)
- Trigger composé sur `TI_COMMANDES` (validation + historique)
- Fonction pipelined pour les rapports
- SQL dynamique pour la recherche multi-critères

### Cahier des charges

```sql
-- 1. pkg_commandes_v2 SPEC
CREATE OR REPLACE PACKAGE pkg_commandes_v2 AS

  -- Constantes
  C_TVA_STD CONSTANT NUMBER := 0.20;

  -- Types
  TYPE t_cmd_rec IS RECORD (
    cmd_id      ti_commandes.cmd_id%TYPE,
    numero      ti_commandes.numero%TYPE,
    client_nom  ti_clients.nom%TYPE,
    statut      ti_commandes.statut%TYPE,
    montant_ttc ti_commandes.montant_ttc%TYPE,
    nb_lignes   NUMBER
  );

  -- Créer une commande avec validation
  PROCEDURE creer_commande (
    p_client_id  IN  NUMBER,
    p_emp_id     IN  NUMBER DEFAULT NULL,
    p_cmd_id     OUT NUMBER,
    p_numero     OUT VARCHAR2
  );

  -- Passer au statut suivant
  PROCEDURE avancer_statut (p_cmd_id IN NUMBER);

  -- Annuler une commande
  PROCEDURE annuler_commande (p_cmd_id IN NUMBER, p_motif IN VARCHAR2 DEFAULT NULL);

  -- Recherche dynamique (retourne un REF CURSOR)
  PROCEDURE rechercher (
    p_statut    IN  VARCHAR2 DEFAULT NULL,
    p_client_id IN  NUMBER   DEFAULT NULL,
    p_cur       OUT SYS_REFCURSOR
  );

  -- Exception publique
  e_statut_invalide EXCEPTION;
  PRAGMA EXCEPTION_INIT(e_statut_invalide, -20011);

END pkg_commandes_v2;
/

-- 2. pkg_commandes_v2 BODY (à compléter)
CREATE OR REPLACE PACKAGE BODY pkg_commandes_v2 AS

  PROCEDURE creer_commande (
    p_client_id  IN  NUMBER,
    p_emp_id     IN  NUMBER DEFAULT NULL,
    p_cmd_id     OUT NUMBER,
    p_numero     OUT VARCHAR2
  ) IS
    v_nb   NUMBER;
    v_actif ti_clients.actif%TYPE;
  BEGIN
    -- Validation client
    -- ?

    -- Générer numéro et id
    SELECT seq_commande.NEXTVAL INTO p_cmd_id FROM dual;
    p_numero := 'CMD-' || TO_CHAR(SYSDATE,'YYYYMMDD') || '-' || LPAD(p_cmd_id, 5, '0');

    -- Insérer
    INSERT INTO ti_commandes (cmd_id, numero, client_id, emp_id, statut)
    VALUES (p_cmd_id, p_numero, p_client_id, p_emp_id, 'NOUVEAU');

    -- Audit autonome
    pkg_audit.log_action('TI_COMMANDES', 'INSERT', NULL, 'cmd=' || p_cmd_id,
      'Client=' || p_client_id);
    COMMIT;
    DBMS_OUTPUT.PUT_LINE('Commande créée : ' || p_numero);
  END;

  PROCEDURE avancer_statut (p_cmd_id IN NUMBER) IS
    v_statut_act ti_commandes.statut%TYPE;
    v_statut_suiv VARCHAR2(20);
  BEGIN
    SELECT statut INTO v_statut_act FROM ti_commandes WHERE cmd_id = p_cmd_id;

    -- Machine à états TechInfo
    v_statut_suiv := CASE v_statut_act
      WHEN 'NOUVEAU'   THEN 'EN_COURS'
      WHEN 'EN_COURS'  THEN 'EXPEDIE'
      WHEN 'EXPEDIE'   THEN 'LIVRE'
      ELSE NULL
    END;

    IF v_statut_suiv IS NULL THEN
      RAISE_APPLICATION_ERROR(pkg_errors.C_CMD_STATUT_INVALIDE,
        'Statut ' || v_statut_act || ' ne peut pas être avancé');
    END IF;

    UPDATE ti_commandes SET statut = v_statut_suiv WHERE cmd_id = p_cmd_id;
    pkg_audit.log_action('TI_COMMANDES', 'UPDATE STATUT',
      v_statut_act, v_statut_suiv, 'cmd=' || p_cmd_id);
    COMMIT;
    DBMS_OUTPUT.PUT_LINE('Statut avancé : ' || v_statut_act || ' → ' || v_statut_suiv);
  END;

  PROCEDURE annuler_commande (p_cmd_id IN NUMBER, p_motif IN VARCHAR2 DEFAULT NULL) IS
    v_statut ti_commandes.statut%TYPE;
  BEGIN
    SELECT statut INTO v_statut FROM ti_commandes WHERE cmd_id = p_cmd_id;
    IF v_statut = 'LIVRE' THEN
      RAISE_APPLICATION_ERROR(pkg_errors.C_CMD_STATUT_INVALIDE,
        'Impossible d''annuler une commande déjà LIVREE');
    END IF;
    UPDATE ti_commandes SET statut = 'ANNULE', commentaire = p_motif
     WHERE cmd_id = p_cmd_id;
    pkg_audit.log_action('TI_COMMANDES', 'ANNULE', v_statut, 'ANNULE', p_motif);
    COMMIT;
    DBMS_OUTPUT.PUT_LINE('Commande ' || p_cmd_id || ' annulée.');
  END;

  PROCEDURE rechercher (
    p_statut    IN  VARCHAR2 DEFAULT NULL,
    p_client_id IN  NUMBER   DEFAULT NULL,
    p_cur       OUT SYS_REFCURSOR
  ) IS
    v_sql VARCHAR2(1000) :=
      'SELECT c.cmd_id, c.numero, cl.nom, c.statut, c.montant_ttc, '
      || '(SELECT COUNT(*) FROM ti_lignes_cmd l WHERE l.cmd_id = c.cmd_id) '
      || 'FROM ti_commandes c JOIN ti_clients cl ON cl.client_id = c.client_id '
      || 'WHERE 1=1';
  BEGIN
    IF p_statut    IS NOT NULL THEN v_sql := v_sql || ' AND c.statut = :st'; END IF;
    IF p_client_id IS NOT NULL THEN v_sql := v_sql || ' AND c.client_id = :cli'; END IF;
    v_sql := v_sql || ' ORDER BY c.date_cmd DESC';

    IF    p_statut IS NOT NULL AND p_client_id IS NOT NULL THEN
      OPEN p_cur FOR v_sql USING p_statut, p_client_id;
    ELSIF p_statut IS NOT NULL THEN
      OPEN p_cur FOR v_sql USING p_statut;
    ELSIF p_client_id IS NOT NULL THEN
      OPEN p_cur FOR v_sql USING p_client_id;
    ELSE
      OPEN p_cur FOR v_sql;
    END IF;
  END;

END pkg_commandes_v2;
/

-- Test du package complet
DECLARE
  v_cmd_id NUMBER;
  v_numero VARCHAR2(50);
  v_cur    SYS_REFCURSOR;
  v_rec    pkg_commandes_v2.t_cmd_rec;
BEGIN
  -- Créer une commande
  pkg_commandes_v2.creer_commande(1, 1, v_cmd_id, v_numero);

  -- Avancer le statut
  pkg_commandes_v2.avancer_statut(v_cmd_id);
  pkg_commandes_v2.avancer_statut(v_cmd_id);

  -- Rechercher les commandes EXPEDIE
  pkg_commandes_v2.rechercher(p_statut => 'EXPEDIE', p_cur => v_cur);
  DBMS_OUTPUT.PUT_LINE('=== Commandes EXPEDIE ===');
  LOOP
    FETCH v_cur INTO v_rec.cmd_id, v_rec.numero, v_rec.client_nom,
                     v_rec.statut, v_rec.montant_ttc, v_rec.nb_lignes;
    EXIT WHEN v_cur%NOTFOUND;
    DBMS_OUTPUT.PUT_LINE('  ' || v_rec.numero || ': ' || v_rec.client_nom ||
      ': ' || v_rec.statut);
  END LOOP;
  CLOSE v_cur;
END;
/
```

---

## Récapitulatif Jour 2: Ce que vous savez faire

| Concept | Compétence |
|---|---|
| Transactions autonomes | `pkg_audit` indépendant, log survit au `ROLLBACK` |
| SAVEPOINT | Rollback partiel étape par étape |
| DEFINER / INVOKER | Ségrégation des droits, utilitaires multi-schémas |
| Dictionnaire de données | `USER_OBJECTS`, `USER_SOURCE`, `USER_ERRORS`, `ALL_DEPENDENCIES` |
| Récursivité | Hiérarchie `mgr_id`, garde-fou, alternative `CONNECT BY` |
| Fonctions PIPELINED | Streaming `PIPE ROW`, `TABLE()` dans SQL, filtres paramétrés |
| Surcharge | `pkg_format.formater()` multi-types, DETERMINISTIC |
| RESULT_CACHE | Cache SGA partagé sur `TI_PARAMETRES`, `RELIES_ON` |
| Triggers composés | 4 sections, résolution mutating table, `FOLLOWS`/`PRECEDES` |
| Triggers DDL | Audit `ORA_SYSEVENT`, `AUTONOMOUS_TRANSACTION` |
| Packages Oracle | `DBMS_SCHEDULER`, `UTL_FILE`, `DBMS_METADATA` |
| SQL dynamique | `EXECUTE IMMEDIATE` + bind variables, validation liste blanche |

---

## Bonnes pratiques Jour 2: Mémo formateur

**À insister absolument :**
- `PRAGMA AUTONOMOUS_TRANSACTION` = **COMMIT obligatoire** à la fin
- Trigger DDL avec `EXCEPTION WHEN OTHERS` : ne pas bloquer les DDL
- `EXECUTE IMMEDIATE` : toujours valider le nom de table via `user_objects`
- PIPELINED : `RETURN;` sans valeur = fin du flux (différent d'une fonction normale)
- RESULT_CACHE : vérifier `result_cache_max_size > 0` avant de l'utiliser

**Anti-patterns du Jour 2 :**
- `COMMIT` dans un trigger standard → `ORA-04092`
- `AUTONOMOUS_TRANSACTION` sans `COMMIT`/`ROLLBACK` → `ORA-06519`
- Concaténer des valeurs utilisateur dans `EXECUTE IMMEDIATE` → injection SQL
- Trigger LOGON qui plante → impossible de se connecter

---

*Fin du support etudiant: Jour 2: ORA-PLAV: TechInfo Solutions*
