# Objectif

Automatiser les tâches répétitives au sein d'une boîte de production artistique française liées à l'embauche :

- Édition et envoi des contrats de travail
- Préparation des données de paie
- Envoi des bulletins de paie
- Registre des montants des paies (masse, brut, net, etc.)
- Suivi des étapes de l'embauche, de la DPAE jusqu'à la paie

**Contexte spécifique :** Contrats à durée déterminée d'usage (CDDU) pour trois types de salariés :

- **Intermittents artistes (annexe 10)** (régime des intermittents, prestations artistiques)
- **Intermittents techniciens (annexe 8)** (régime des intermittents, prestations techniques)
- **Régime général** (formateurs, intervenants, etc.)

**Problème actuel :** L'embauche en CDDU est une tâche aussi importante que répétitive, chronophage et soumise à erreurs. Elle compte plusieurs étapes qui doivent être exécutées dans un ordre précis (déclarations préalables, édition des contrats, archivage, émission et envoi des bulletins de paie, virements des salaires) et dans le respect d'un calendrier et d'échéances strictes.

**Solution proposée :** Une application qui, à partir de données saisies dans Google Sheets, génère automatiquement les contrats, crée les brouillons d'emails pour envoi aux salariés, prépare les données de paie et intègre les montants des bulletins dans le tableur, tout en assurant le suivi des étapes.

# Utilisateur cible

Administrateur / producteur de compagnie artistique (1-3 personnes).

**Profil :**

- À l'aise avec Google Sheets
- Responsable des embauches et de la paie
- Traite typiquement 5-20 contrats par mois

# Contraintes permanentes

Ces règles s'appliquent à toutes les versions de l'application. Elles reflètent les pratiques réelles de la compagnie et les contraintes légales du secteur.

## Structure des contrats

- **Une ligne = un contrat**
- **Un contrat = un salarié** (pas de contrats multi-salariés)
- **Un contrat = un emploi** (pas de contrats multi-emplois)
- **Un contrat = une ou plusieurs prestations** (journées de travail, heures d'enseignement, répétitions, représentations)
- Un salarié peut avoir plusieurs contrats simultanément, pour un ou plusieurs spectacles
- **Contrats intermittents (A8 et A10) :** toujours associés à un spectacle
- **Contrats régime général (RG) :** pas de spectacle associé

## Bulletins de paie

- Chaque bulletin de paie (BdP) est lié à un seul contrat et à un seul mois civil
- Un contrat dont toutes les prestations tombent dans le même mois donne lieu à un seul BdP
- Un contrat qui s'étend sur plusieurs mois donne lieu à un BdP par mois concerné
- Les BdPs ne sont jamais regroupés entre plusieurs contrats

## Signature employeur

- La signature de l'employeur est intégrée au template
- Le contrat est considéré signé par l'employeur dès sa génération
- Date de signature = date de génération du document

# V1 = Première version fonctionnelle

## Simplifications et garde-fous en V1

Ces contraintes s'appliquent uniquement en V1 et seront levées dans les versions ultérieures :

1. **Un contrat = un mois**
   - Les contrats ne chevauchent jamais deux mois civils
   - Des missions dans des mois différents donnent lieu à des contrats séparés
   - Conséquence directe : chaque contrat V1 donne lieu à exactement un BdP

2. **Prestations uniformes par type**
   - Toutes les prestations d'un même type dans un contrat partagent la même durée et le même salaire brut unitaire
   - Exemples : 3 répétitions de 4h à 80€ chacune + 2 représentations à 240€ chacune ; 5 journées de 4h à 120€ chacune
   - En V2 : chaque prestation pourra avoir sa propre durée et son propre salaire

3. **Templates fixes**
   - En V1 : trois templates fixes, un par type de contrat (A10, A8, RG)
   - En V2 : templates configurables, éditeur à tiroirs

4. **Envoi manuel uniquement**
   - Un brouillon Gmail est créé pour relecture et ajustement
   - L'utilisateur envoie manuellement après validation

## Structure des données en V1

Le fichier Google Sheets "BD Embauches" contient quatre feuilles :

- **Salaries** : données personnelles, coordonnées, informations sociales
- **Spectacles** : titre, code, numéro d'objet
- **Emplois** : règles métier par emploi (voir ci-dessous)
- **Contrats** : une ligne par contrat

### Feuille Emplois

La feuille Emplois est introduite dès V1. Elle porte les règles métier propres à chaque emploi, utilisées pour la génération et la validation des contrats.

**Emplois disponibles en V1 :**

| Emploi                  | Type de contrat | Type de répétition    |
| ----------------------- | --------------- | --------------------- |
| Artiste musicien        | A10             | Cachet de répétition  |
| Artiste chorégraphique  | A10             | Service de répétition |
| Chargé de production    | A8              | —                     |
| Intervenant occasionnel | RG              | —                     |

D'autres emplois seront ajoutés au fil des besoins sans impact majeur sur l'architecture.

En V2+, la feuille Emplois sera enrichie de règles supplémentaires : salaires minimaux par prestation, règles de validation des quantités et durées, etc. Elle pourra migrer vers une base de données relationnelle avec le reste des données.

## Types de contrats en V1

### A10 — Intermittent artiste

- Associé à un spectacle
- `statut` = "artiste"
- Prestations :
  - **Répétitions** : jusqu'à 20 dates, même durée (heures), même brut unitaire pour toutes
    - Le type de répétition ("service de répétition" ou "cachet de répétition") est déterminé automatiquement par l'emploi (ex : artiste musicien → cachet ; artiste chorégraphique → service)
  - **Représentations** : jusqu'à 12 dates, chacune avec son propre lieu et son propre brut
- Contrainte : au moins 1 répétition OU 1 représentation requise

### A8 — Intermittent technicien

- Associé à un spectacle
- `statut` = "cadre" ou "non-cadre"
- Prestations :
  - **Journées** : jusqu'à 20 dates, chacune avec son propre lieu, même durée (heures) et même brut unitaire pour toutes
- Contrainte : au moins 1 journée requise
- Note V2 : il sera possible de préciser la nature de chaque journée (montage, production, répétition, représentation, etc.)

### RG — Régime général

- Pas de spectacle associé
- `statut` = "cadre" ou "non-cadre"
- Prestation unique et permanente (pas de changement prévu en V2) :
  - date de début, date de fin, nature, lieu, brut convenu
- Le salaire bénéficie de deux indemnités légales (prime de précarité et congés payés), chacune multipliant le brut par 1.1, soit un coefficient total de 1.21. Ce coefficient (`coef_indemnites`) est défini dans la feuille Emplois et appliqué automatiquement. Le montant résultant (`prestation_brut_indemnites`) est calculé par formule dans la feuille Contrats.

## Champs ignorés selon le type de contrat

Certains champs de la feuille "Contrats" ne sont pertinents que pour certains types. En V1, si un champ non applicable à un type est renseigné, l'application affiche un avertissement dans la console mais continue le traitement normalement.

En V2 (avec front-end), les champs seront affichés ou masqués dynamiquement selon le type de contrat.

| Champ                                                                                                                       | A10    | A8     | RG     |
| --------------------------------------------------------------------------------------------------------------------------- | ------ | ------ | ------ |
| spectacle_id                                                                                                                | ✓      | ✓      | ignoré |
| date_repetition1…20, duree_repetitions, brut_par_repetition                                                                 | ✓      | ignoré | ignoré |
| date_representation1…12, lieu_representation1…12, brut_representation1…12                                                   | ✓      | ignoré | ignoré |
| date_journee1…20, lieu_journee1…20, duree_journees, brut_par_journee                                                        | ignoré | ✓      | ignoré |
| prestation_date_debut, prestation_date_fin, prestation_nature, prestation_lieu, prestation_brut, prestation_brut_indemnites | ignoré | ignoré | ✓      |

## Schéma des feuilles Google Sheets

### Salaries

| Colonne             | Description                                                                    |
| ------------------- | ------------------------------------------------------------------------------ |
| `salarie_id`        | Identifiant unique technique                                                   |
| `sobriquet`         | Identifiant unique créé par l'utilisateur                                      |
| `nom`               | Nom de famille                                                                 |
| `prenom`            | Premier prénom ou prenoms d'usage                                              |
| `prenom_autres`     | Prénoms suivants                                                               |
| `nom_docs`          | Nom affiché sur les documents et nommage des fichiers (doit être unique)       |
| `preferred_gender`  | "homme" ou "femme" — utilisé pour accords grammaticaux dans emails et contrats |
| `date_naissance`    | Date de naissance                                                              |
| `ville_naissance`   | Ville de naissance                                                             |
| `dep_naissance`     | Département de naissance (nom)                                                 |
| `num_dep_naissance` | Numéro de département (doit être cohérent avec INSEE)                          |
| `pays_naissance`    | Pays de naissance                                                              |
| `nationalite`       | Nationalité                                                                    |
| `insee`             | Numéro de sécurité sociale                                                     |
| `numero_cs`         | Immatriculation Congés Spectacles                                              |
| `telephone`         | Téléphone                                                                      |
| `email`             | Adresse email                                                                  |
| `adresse_ligne1`    | Adresse postale — ligne 1                                                      |
| `adresse_ligne2`    | Ligne 2                                                                        |
| `adresse_ligne3`    | Ligne 3                                                                        |
| `adresse_ligne4`    | Ligne 4                                                                        |
| `adresse_ligne5`    | Ligne 5                                                                        |
| `IBAN`              | IBAN pour virement                                                             |
| `BIC`               | BIC pour virement                                                              |
| `regime_actuel`     | Régime d'indemnisation en cours : A8 / A10 / RG / autre / vide _(V2)_          |
| `fin_de_droits`     | Fin des droits du dossier en cours _(V2)_                                      |

**Notes :**

- `num_dep_naissance` et `insee` doivent être cohérents (les deux premiers chiffres du numéro INSEE encodent le département). Une validation croisée sera implémentée après V1, incluant la cohérence avec `date_naissance`.
- `nom_docs` doit être unique. En V1, l'unicité est assurée par l'utilisateur. Une validation automatique sera ajoutée lors de la migration vers une base de données.
- Les champs marqués _(V2)_ ne sont ni lus ni écrits par l'application avant V2, mais ils sont créés dès V1 (colonnes vides) pour éviter une migration de schéma ultérieure.

### Spectacles

| Colonne                  | Description                                                                    |
| ------------------------ | ------------------------------------------------------------------------------ |
| `spectacle_id`           | Identifiant unique technique                                                   |
| `code`                   | Identifiant court créé par l'utilisateur (ex: "RJ")                            |
| `titre`                  | Titre complet                                                                  |
| `numero_objet`           | Numéro d'objet (fourni par France Travail)                                     |
| `nombre_representations` | Nombre de représentations _(V2 — calculé automatiquement depuis les contrats)_ |

### Emplois

| Colonne                 | Description                                                                        |
| ----------------------- | ---------------------------------------------------------------------------------- |
| `emploi_id`             | Identifiant unique technique                                                       |
| `code`                  | Identifiant court créé par l'utilisateur                                           |
| `intitule_masculin`     | Nom de l'emploi, accord masculin (ex: "Artiste musicien")                          |
| `intitule_feminin`      | Nom de l'emploi, accord feminin (ex: "Artiste musicienne")                         |
| `type_contrat`          | A10 / A8 / RG                                                                      |
| `type_repetition`       | "service" / "cachet" / vide si non applicable                                      |
| `statut_emploi_default` | cadre / non-cadre — valeur par défaut, non appliquée automatiquement en V1         |
| `coef_indemnites`       | Multiplicateur d'indemnités (ex: `1.21` pour RG, vide sinon) _(extensible en V2+)_ |
| `convention_collective` | Convention collective applicable (ex: "CCNEAC") _(V2)_                             |

**Note :** Le champ `convention_collective` permettra à terme de gérer des règles métier propres à chaque convention (ex: type de répétition, salaires minimaux). Une configuration externe (type `idcc_config.json`) pourra être introduite en V2+ pour centraliser ces règles. Voir `spec.md` pour les détails.

### Contrats

**Identité et référence**

| Colonne         | Description                                                                                                                          |
| --------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| `contrat_id`    | Identifiant unique technique, calculé par formule Google Sheets en V1                                                                |
| `salarie_id`    | Référence Salaries                                                                                                                   |
| `emploi_id`     | Référence Emplois                                                                                                                    |
| `spectacle_id`  | Référence Spectacles (ignoré si RG)                                                                                                  |
| `type_contrat`  | A10 / A8 / RG — dérivé de `emploi_id` par formule                                                                                    |
| `statut_emploi` | cadre / non-cadre — défini par l'utilisateur (indépendant de `statut_emploi_default` en V1)                                          |
| `date_dpae`     | Date à laquelle l'employeur a effectué la DPAE auprès de l'Urssaf                                                                    |
| `fichier_dpae`  | Nom à donner au recépissé de DPAE pour archivage, calculé par formule — ex: "DPAE `[nom_docs] [date_debut]-[date_fin] [contrat_id]`" |

**Prestations A10 — répétitions**

| Colonne                                  | Description                             |
| ---------------------------------------- | --------------------------------------- |
| `date_repetition1` … `date_repetition20` | Dates des répétitions (20 colonnes)     |
| `duree_repetitions`                      | Durée en heures (commune à toutes)      |
| `brut_par_repetition`                    | Montant brut unitaire (commun à toutes) |

**Prestations A10 — représentations**

| Colonne                                          | Description                  |
| ------------------------------------------------ | ---------------------------- |
| `date_representation1` … `date_representation12` | Dates (12 colonnes)          |
| `lieu_representation1` … `lieu_representation12` | Lieux (12 colonnes)          |
| `brut_representation1` … `brut_representation12` | Montants bruts (12 colonnes) |

**Prestations A8 — journées**

| Colonne                            | Description                             |
| ---------------------------------- | --------------------------------------- |
| `date_journee1` … `date_journee20` | Dates des journées (20 colonnes)        |
| `lieu_journee1` … `lieu_journee20` | Lieux (20 colonnes)                     |
| `duree_journees`                   | Durée en heures (commune à toutes)      |
| `brut_par_journee`                 | Montant brut unitaire (commun à toutes) |

**Prestation RG**

| Colonne                      | Description                                    |
| ---------------------------- | ---------------------------------------------- |
| `prestation_date_debut`      | Date de début                                  |
| `prestation_date_fin`        | Date de fin                                    |
| `prestation_nature`          | Description de la mission                      |
| `prestation_lieu`            | Lieu                                           |
| `prestation_brut`            | Montant brut convenu                           |
| `prestation_brut_indemnites` | Brut × `coef_indemnites` — calculé par formule |

**Totaux (formules Sheets)**

| Colonne       | Description                                                         |
| ------------- | ------------------------------------------------------------------- |
| `brut_total`  | Calculé automatiquement (inclut `prestation_brut_indemnites` si RG) |
| `date_debut`  | Première prestation (calculé)                                       |
| `date_fin`    | Dernière prestation (calculé)                                       |
| `nom_fichier` | Généré automatiquement par formule                                  |

**Frais et paie**

| Colonne                 | Description                                                                                                              |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| `defraiements`          | Montant des frais                                                                                                        |
| `defraiements_details`  | Détail des frais — usage interne                                                                                         |
| `cout_bdp`              | Coût du bulletin de paie                                                                                                 |
| `masse_salariale`       | Saisi après réception BdP                                                                                                |
| `net_avant_pas`         | Net avant prélèvement à la source (aussi appelé "Net social") — saisi après réception BdP                                |
| `montant_net_imposable` | Montant net imposable — saisi après réception BdP                                                                        |
| `net_a_payer`           | Net à payer — saisi après réception BdP                                                                                  |
| `prelevement_source`    | Prélèvement à la source — saisi après réception BdP                                                                      |
| `ref_virement`          | Usage interne — ex: `SALAIRE DUPONT RJ MAI 2026 - NET 450.00 - FRAIS 18 - [contrat_id]`                                  |
| `message_virement`      | Pour le salarié — ex: `SALAIRE COMEDIENNE RJ MAI 2026 - NET 450.00 - FRAIS 18 - CONTRAT [contrat_id]`                    |
| `nom_bdp`               | Nom du fichier BdP — ex: `2026-05 SALAIRE [nom_docs] [spectacle_id] [intitule] [contrat_id]` (`spectacle_id` omis si RG) |

**Tracking**

| Colonne          | Description                                                                                                                                                                                                                                                                                                  |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `url_doc`        | Lien vers le Google Doc                                                                                                                                                                                                                                                                                      |
| `url_mail`       | Lien vers le brouillon Gmail _(V2 : lien vers l'email envoyé)_                                                                                                                                                                                                                                               |
| `status`         | `success` / `error` / vide                                                                                                                                                                                                                                                                                   |
| `log`            | JSON avec détails (`tasks_status` (clés possibles : `doc`, `pdf`, `contract_email_draft`, `contract_email`, `bdp_parsing`, `bdp_email_draft`, `bdp_email_sent`) + `errors` si applicable) **ou** `checksum` si `status` = `success` (voir ci-dessous: "Protection contre les modifications post-génération") |
| `date_processed` | Horodatage du dernier traitement                                                                                                                                                                                                                                                                             |

## Idempotence

Le traitement d'un contrat est considéré comme déjà effectué si et seulement si `status` = `success`. Dans tous les autres cas (`status` = `error`, traitement partiel), la ligne est retraitée et les champs de tracking sont écrasés.

Ce comportement gère notamment les interruptions partielles : si un Google Doc a été créé mais que le brouillon Gmail n'a pas pu être généré, la ligne sera retraitée intégralement au prochain lancement.

## Protection contre les modifications post-génération

Une fois un contrat généré et envoyé au salarié, modifier les données source dans le tableur crée un risque de discordance entre le document signé et les données de référence. Deux mécanismes complémentaires sont mis en place :

**A — Détection par checksum**
Lors du traitement d'une ligne, le script calcule un hash des champs contractuels (informations du salarié, prestations, identités, montants, etc.) et le stocke dans le champ `log`. À chaque relance, le script recalcule ce hash et le compare à la valeur stockée. Si `status = success` mais que le hash a changé, le script affiche un avertissement explicite; la ligne sera traitée à nouveau suelement si l'app est lancée avec le flag `--force`. Ce mécanisme détecte toute modification silencieuse intervenue après génération.

**B — Protection native Google Sheets**
Après traitement réussi, le script applique via l'API Sheets une protection sur la ligne concernée, restreignant les modifications aux propriétaires du fichier. Cette protection crée une friction visible directement dans l'interface Sheets et prévient les éditions accidentelles. Elle reste levable manuellement pour les corrections intentionnelles.

_Accessoirement, un méchanisme de distinction visuelle peut être mis en place : les lignes traitées peuvent être légèrement grisées._

Les champs de tracking (`url_doc`, `url_mail`, `status`, `log`, `date_processed`) et les champs de paie (`masse_salariale`, `net_a_payer`, etc.) restent toujours éditables et sont exclus de la protection.

Les détails d'implémentation (champs inclus dans le hash, comportement du flag `--force`, appel API pour la protection) sont décrits dans `spec.md`.

## Authentification

L'application utilise OAuth 2.0 via un fichier `oauth-credentials.json` stocké dans le dossier de l'application. Au premier lancement, un navigateur s'ouvre pour le consentement OAuth ; le token résultant est stocké dans `token.json` et réutilisé automatiquement. Les deux fichiers doivent figurer dans `.gitignore`.

**Scopes requis :**

- Google Sheets (lecture/écriture)
- Google Drive (création/copie de fichiers)
- Google Docs (édition)
- Gmail (création de brouillons)

## Périmètre V1

**Fonctionnalités :**

- Génération de contrats Google Doc depuis un template (un par type : A10, A8, RG)
- Export automatique en PDF
- Archivage des documents Google Doc et PDF selon arboresence par date de début du contrat
- Création de brouillons Gmail avec PDF en pièce jointe
- Traitement par lot de plusieurs contrats
- Reprise en cas d'interruption (idempotence)
- Saisie manuelle des montants de paie après réception des bulletins

**Interface :**

- Saisie des données : Google Sheets
- Exécution : ligne de commande Node.js
- Pas de front-end web ni de backoffice

## Commande

```bash
node index.js [rangées] --force
```

**Exemples :**

- `node index.js` → Traite tous les contrats pas encore traités
- `node index.js 3` → Traite uniquement la rangée 3
- `node index.js 3 --force` → Traite uniquement la rangée 3, même si elle à déjà été traitée ou si le sumcheck ne match pas les valeurs actuels
- `node index.js 3-5 7` → Traite les rangées 3, 4, 5 et 7

## Gestion des erreurs

Si une erreur survient sur un contrat :

- L'erreur est affichée dans la console
- Le traitement continue avec les contrats suivants
- Un résumé final indique le nombre de succès et d'erreurs
- L'erreur est renseginé dans le champ `log` de "Contrats".

# Exemple d'utilisation

## Contexte

La compagnie "Les Bateleurs" embauche Marie Dupont (comédienne, A10) pour 3 répétitions et 1 représentation du spectacle "Roméo et Juliette" en mai 2026.

## Étape 1 : Saisie dans Google Sheets

L'administrateur vérifie que Marie Dupont existe dans "Salaries" et que "Roméo et Juliette" existe dans "Spectacles" (code : "RJ").

Il crée une nouvelle ligne dans "Contrats" :

- Type : A10 / Salarié : Marie Dupont / Spectacle : RJ / Emploi : Comédienne
- 3 répétitions : 1, 2 et 17 mai (4h chacune, 80€/répétition)
- 1 représentation : 4 mai, Gazette Café de Montpellier (240€)
- Défraiements : 18€ / Coût BdP : 13,50€
- Date de la DPAE: 20 avril 2026

Les formules Google Sheets calculent automatiquement :

- Brut total : 480€ (3×80€ + 240€)
- Date de début : 1 mai 2026 / Date de fin : 17 mai 2026
- Nom du fichier de la DPAE :
- Nom du fichier (contrat Gdocs et PDF) : "CDDU Marie Dupont - Comédienne (RJ) - 1-17 mai 2026"
- fichier_dpae : "DPAE Marie Dupont 1-17 mai 2026 `[contrat_id]`" (l'utilisateur renomme ainsi le fichier PDF de la DPAE)

## Étape 2 : Génération

```bash
node index.js
```

```
Traitement de 1 rangée...
✓ Rangée 2: "CDDU Marie Dupont - Comédienne (RJ) - 1-17 mai 2026"
  - Google Doc créé
  - PDF exporté
  - Brouillon Gmail créé

Traitement terminé.
Succès: 1 rangée
Erreurs: 0 rangée
```

## Étape 3 : Vérification et envoi

L'administrateur ouvre le Google Doc (lien dans `url_doc`), vérifie le contrat, ouvre le brouillon Gmail (lien dans `url_mail`), ajuste si nécessaire, et envoie.

## Étape 4 : Après réception du bulletin de paie

L'administrateur saisit manuellement dans le tableur :

- Masse salariale : 900€ / Net à payer : 450€ / Prélèvement à la source : 3,60€

Les formules `ref_virement` et `message_virement` se mettent à jour :

```
ref_virement     : SALAIRE DUPONT RJ MAI 2026 - NET 450.00 - FRAIS 18 - [contrat_id]
message_virement : SALAIRE COMEDIENNE RJ MAI 2026 - NET 450.00 - FRAIS 18 - CONTRAT [contrat_id]
```

# Critères de succès V1

- ✅ Génère un contrat Google Doc conforme au template légal pour chacun des 3 types (A10, A8, RG)
- ✅ Exporte automatiquement en PDF sans intervention manuelle
- ✅ Crée un brouillon Gmail prêt à envoyer (destinataire, objet, PJ)
- ✅ Traite 10+ contrats d'affilée sans erreur
- ✅ Reprend où il s'est arrêté si relancé (pas de doublons)
- ✅ Affiche clairement les erreurs et avertissements pour faciliter le débogage
- ✅ Temps de génération < 15 secondes par contrat

# Hors scope V1

## Structure de données

- Base de données relationnelle (MongoDB/PostgreSQL)
- Répétitions et journées comme entités indépendantes (en V2, une même répétition — même date/lieu/spectacle — pourra apparaître dans les contrats de plusieurs salariés différents)

## Règles métier avancées sur les emplois

La feuille Emplois sera enrichie des règles suivantes :

- Salaires minimaux par type de prestation (service, cachet, représentation, journée)
- Règles de validation : quantité maximale de prestations par période, durée maximale par journée ou service, etc.
- Ces règles seront appliquées automatiquement lors de la validation d'un contrat
- Configuration par convention collective (`idcc_config.json` ou équivalent)

## Suivi des dossiers d'intermittence

- Validation croisée du champ `insee` avec `date_naissance` et `num_dep_naissance`
- Utilisation de `fin_de_droits` : si un contrat chevauche deux dossiers d'intermittence, l'application affiche un avertissement — ce chevauchement peut être défavorable au salarié
- L'application propose alors de découper automatiquement le contrat en deux (un par dossier), ou laisse l'utilisateur le faire manuellement

## Automatisation et envoi des emails

- Choix par contrat : brouillon Gmail ou envoi direct sans relecture

## Workflow avancé

- Commandes indépendantes (`gdocs`, `pdf`, `email-draft`)
- Suivi d'état détaillé (DPAE, signature salarié, paie effectuée)
- Gestion des avenants et modifications de contrats
- Templates configurables par type de contrat
- Éditeur de contrats à tiroirs (activation/désactivation d'articles, choix entre modèles)

## Prestations

- Tarifs et durées variables par prestation individuelle
- Nature des journées A8 (montage, production, répétition, représentation, etc.)
- Contrats multi-mois
- Patterns de nommage configurables par type de contrat (`ref_virement`, `message_virement`, `nom_bdp`)

## Interface

- Front-end web pour saisie et suivi
- Affichage dynamique des champs selon le type de contrat
- Interface de configuration des formats et templates
- Dashboard de suivi des embauches

## Infrastructure

- Multi-utilisateurs avec permissions
- Historique et archivage automatique avec versioning
- Tests automatisés (unitaires, intégration)
- Logs structurés et monitoring
- CI/CD

---

**Pour les détails techniques, voir :**

- `spec.md` : Spécifications fonctionnelles complètes
- `architecture.md` : Design technique et organisation du code
