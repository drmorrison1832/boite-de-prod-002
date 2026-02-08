# Objectif

Automatiser les tâches répétitives au sein d'une boîte de production artistique française :

- Édition et envoi des contrats de travail
- Préparation des données de paie
- Envoi des bulletins de paie

**Contexte spécifique :** contrats d'artistes et de techniciens sous régime de l'intermittence + contrats de formateurs et intervenants au régime général. Les contrats ne chevauchent pas entre deux mois; des missions dans des mois différents donnent lieu à des contrats séparés.

# Utilisateur cible

Administrateur / producteur de compagnie artistique (1-3 personnes).

# V1 = Première version fonctionnelle

- Un seul type de contrat : artiste intermittent
- Pas d'envoi email automatique : génération de brouillons Gmail
- Pas de parsing automatique des bulletins PDF (saisie manuelle des infos)
- Interface = Google Sheets directement (pas de front-end)
- Base de données = Google Sheets
- Génération déclenchée manuellement en ligne de commande : `node index.js [rangées]`.
  Exemples:
  - `node index.js` : génère les brouillons d'email de tous les rangées par encore traitées ;
  - `node index.js 3` : traite la rangée 3 ;
  - `node index.js 3-5 7` : traite les rangées 3 à 5 et la rangée 7 ;
  - `node index.js 2-4 6 9-10` : traite les rangées 2 à 4, la rangée 6, et les rangées 9 à 10.
  - `node index.js 1` : ne fait rien : impossible de traiter la rangée des headers ;
  - `node index.js 1-4` : traite les rangées 2 à 4.

# Flux principal (V1)

1. L'utilisateur remplit la feuille "Salariés", "Spectacles" et "Contrats" dans Google Sheets
2. L'utilisateur lance l'application
3. L'app récupère les données de "Contrats" via Google Sheets API et identifie toutes les rangées pas encore traitées. Une rangée est considérée comme "traitée" si la colonne `date_brouillon` n’est pas `null`. Aucun autre critère n’est utilisé en V1.
   L'application ignore les rangées déjà traitées.

Si au moment de lancer l'exécution l'utilisateur à précisé les numéros de rangées, l'application processe uniquement les rangées demandées.

Pour chacune des rangées non traités :

4. L'app récupère les données du salarié et du spectacle et valide tous les champs avant de continuer
5. L'app crée le contrat au format Google Doc d'après un template prédéfini, l'archive dans un dossier prédéfini, et met l'URL du document dans la colonne `url_doc`
6. L'app exporte le Google Doc au format PDF, l'archive dans un dossier prédéfini, et met l'URL du document dans la colonne `url_pdf`
7. L'app crée un brouillon Gmail avec le PDF en pièce jointe et met à jour les colonnes `date_brouillon` et `url_brouillon`
8. L'utilisateur relit les brouillons dans Gmail et les envoie manuellement

**Gestion des erreurs :**
En cas d'erreur sur une rangée, l'app :

- Log l'erreur dans la console
- Continue avec les rangées suivantes
- À la fin, affiche un résumé : X rangées traitées, Y erreurs

# Structure des données (Google Sheets)

Un seul fichier Google Sheet : "BD Embauches", avec plusieurs feuilles: Salaires, Spectacles, Contrats.

## Feuilles et colonnes

### Salariés

- id : unique (texte, choisi par l'utilisateur)
- nom : nom de famille (texte)
- prenom : prénom (texte)
- nom_de_scene : nom d'artiste (texte, optionnel)
- adresse1...adresse5 : chaque ligne de l'adresse postale (texte)
- telephone : numéro de téléphone (texte)
- email : adresse email (texte)
- date_de_naissance : format YYYY-MM-DD (date)
- pays_naissance : pays de naissance (texte)
- ville_naissance : ville de naissance (texte)
- departement_naissance : département de naissance pour nés en France (nombre entier)
- insee : numéro de sécurité sociale, 15 chiffres (texte)
- cs : code d'inscription à la caisse de congés spectacles (texte)

### Spectacles

- titre : titre complet du spectacle (texte)
- code : titre abrégé, 2-4 lettres (ex: "RJ" pour Roméo et Juliette) - utilisé pour message_virement, pour ref_virement et pour nommer les fichiers du contrat (GDocs et PDF)
- numero_objet : numéro administratif interne généré par France Travail (texte)

### Contrats

#### Saisies par l'utilisateur

- salarie_id : référence à Salariés
- spectacle_id : chaque contrat d'artiste est lié à un spectacle (réference à code)
- emploi : "Artiste musicien", "Chargé de production", etc.
- objet : intitulé du contrat
- statut : cadre / non cadre / artiste
- dateRepetition1...dateRepetition20 : dates des répétitions
- brut_par_repetition
- typeRepetition : "service" ou "cachet"
- dureeRepetitions : durée en heures (nombre, NULL si cachet)
- dateRepresentation1...dateRepresentation12 : dates de chaque représentation (date)
- lieuRepresentation1...lieuRepresentation12 : lieu de chaque représentation (texte)
- brutRepresentation1...brutRepresentation12 : salaire brut de chaque représentation (nombre)
- defraiements : montant des défraiements forfaitaires figurant sur le contrat
- cout_bdp : coût d'émission du bdp
- analytique : code analytique de la paie
- contrat_envoye : NULL par défaut, à remplir manuellement après envoi

**Note :** dans V1, toutes les répétitions d'un même contrat ont :

- le même type (service ou cachet)
- la même durée (dureeRepetitions)
- le même salaire brut (brut_par_repetition)

En V2, chaque répétition pourra avoir sa propre durée et son propre salaire.

#### Générés automatiquement pendant la saisie (par des formules directement dans Google Sheets)

- nom_fichier : "CDDU [prenom] [nom] - [emploi] ([spectacle_id]) - [jour_debut]-[jour_fin] [mois] [annee]"
- brut_total
- date_debut_contrat : date de la première répétition/représentation
- date_fin_contrat : date de la dernière répétition/représentation
- message_virement : généré par formule après saisie de 'net' ; "SALAIRE EMPLOI [spectacle_id] [MOIS ANNEE] - NET [à compléter] - FRAIS [FRAIS]" (se met à jour automatiquement quand 'net' est saisi) ; ex. : "SALAIRE CHARGEE DE PROD RJ MAI 2026 - NET 314,12 - FRAIS 50"
- ref_virement : "SALAIRE [PRENOM] [NOM] - [EMPLOI] [SPECTACLE_ID] - [MOIS ANNEE]"; ex. : "SALAIRE MARIE DUPONT - CHARGEE DE PROD RJ - MAI 2026" (texte normalisée selon normes bancaires: tout en majuscules, pas d'accents, seulement certains symboles sont authorisés)
- fichier_bdp : nom que l'utilisateur donnera au bulletin de paie, à définir, ex. "PAIE [PRENOM] [NOM] - [EMPLOI] [spectacle_id] - [MOIS ANNEE]"

#### Saisies dans V1, parsés depuis le bdp à partir de V2+

- masse : masse salariale renseignée sur le bdp
- net : salaire net renseigné sur le bdp
- pas : prélèvement à la source, renseigné sur le bdp
- virement : montant qui doit être viré au salarié

**Note :** En V1, ces champs sont remplis manuellement par l'utilisateur après réception du BDP. En V2, ils seront parsés automatiquement depuis le PDF.

#### Générés automatiquement lors de l'exécution

- url_doc : généré lors de la création Google Doc
- url_pdf : généré lors de la création du PDF
- date_signature_employeur : date de création du contrat (la signature est intégrée au template, le contrat est considéré signé dès sa création)
- date_brouillon : date de creation du brouillon de l'email
- url_brouillon : URL du brouillon Gmail pour modification, ex : Exemple : "https://mail.google.com/mail/u/0/#drafts/1823abc..."

**Note :** Google Sheets sert de base de données en V1.
Migration vers MongoDB/PostgreSQL envisageable en V2.

# Cas d'usage type (exemple)

## Contexte

La compagnie "Les Bateleurs" embauche Marie Dupont (comédienne) pour 3 répétitions et 1 représentation du spectacle "Roméo et Juliette" en mai 2026.

## Étapes

1. L'admin vérifie que Marie est dans la feuille "Salariés" (id: Marou)
2. L'admin crée une nouvelle rangée dans "Contrats" :
   - salarie_id : "Marou"
   - spectacle_id : "RJ"
   - emploi : Comédienne
   - objet : "répétitions et représentations"
   - statut : "artiste"
   - dateRepetition1 : 2026-05-01
   - dateRepetition2 : 2026-05-02
   - dateRepetition3 : 2026-05-17
   - typeRepetition : "service"
   - dureeRepetitions : 4
   - brut_par_repetition: 80
   - dateRepresentation1 : 2026-05-04
   - lieuRepresentation1 : "Gazette Café de Montpellier"
   - brutRepresentation1 : 240
   - defraiements: 18
   - cout_bdp : 13,5
   - analytique : "MARIE"
3. L'admin lance : `node index.js`
4. Un Google Doc et un PDF appelés `CDDU Marie Dupont - Comédienne (RJ) - 1-17 mai 2026` sont créés et archivés
5. Un brouillon Gmail apparaît avec le PDF en PJ, destinataire = email de Marie
6. L'admin relit, ajuste si besoin, et envoie

# Technologies

- Node.js
- Google Sheets API (base de données)
- Google Docs API (génération contrats)
- Gmail API (création brouillons)
- Google Drive API (stockage documents)

# Hors scope V1

- Base de données relationnelle (MongoDB)
- Remplacement des colonnes répétées (répétitions et représentations) par des collections référenciés.
- Exécution des tâches de manière indépendante (commandes séparées gdocs, pdf, email-draft)
- Envoi automatique des emails (sans relecture)
- Suivi de l'avancement de l'embauche (DPAE, signature, paie effectuée)
- Multi-types de contrats (annexe 8 et régime général)
- Interface web dédiée pour l'exécution du script (front-end)
- Chaque répétition a une durée et un salaire brut indépendant des autres
- Parsing automatique des PDF de bulletins (màj automatique des montants dans "Contrats")
- Interface de paramètrage:
  - dossiers de destination des fichiers
  - nom du fichier du contrat (l'id sera généré automatiquement en base de données),
  - format nom fichier de fiche de paie
  - format référenca bancaire
  - format message virement
- Template gmail (objet et messages)
- Création de templates de contrats à tiroirs / interface de génération de templates
- Gestion des avenants et modifications de contrats
- Système de permissions multi-utilisateurs
- Historique et archivage automatique
- Notifications et rappels

## Découpage logique des responsabilités (V1)

L'application est conçue comme une suite d'étapes indépendantes.

1. Lecture des arguments CLI
   - Interprète les numéros de rangées demandées
   - Exclut la rangée de headers
   - Retourne la liste finale des rangées à traiter

2. Chargement des données
   - Récupère les feuilles Salariés, Spectacles et Contrats
   - Indexe les données par id / code pour accès rapide

3. Sélection des contrats à traiter
   - Identifie les rangées dont date_brouillon est NULL
   - Applique le filtre des rangées demandées via la CLI

4. Validation métier d’une rangée
   - Vérifie la cohérence des références (salarie_id, spectacle_id)
   - Vérifie la présence des champs obligatoires
   - Vérifie la cohérence des dates et montants
   - En cas d’erreur, la rangée est rejetée sans interrompre le batch

5. Construction des données de contrat
   - Agrège les données Salarié, Spectacle et Contrat
   - Calcule les listes de répétitions et représentations
   - Produit une structure prête à être injectée dans un template

6. Génération du contrat Google Docs
   - Crée un Google Doc à partir d’un template
   - Archive le document dans le dossier cible
   - Retourne l’URL du document

7. Export PDF
   - Exporte le Google Doc en PDF
   - Archive le PDF
   - Retourne l’URL du PDF

8. Création du brouillon Gmail
   - Crée un brouillon avec le PDF en pièce jointe
   - Ne déclenche aucun envoi automatique
   - Retourne l’URL du brouillon

9. Mise à jour de la rangée Contrat
   - Met à jour les colonnes générées automatiquement
   - Enregistre les dates et URLs

## Gestion des erreurs

Une erreur sur une étape pour une rangée donnée :

- n’interrompt pas le traitement des autres rangées
- est loggée avec :
  - numéro de rangée
  - étape concernée
  - message d’erreur

Le résumé final indique :

- nombre de rangées traitées avec succès
- nombre de rangées en erreur

# Architecture logique orientée agents (V1)

L’application est structurée en étapes pures et isolées.
Chaque étape :

- a une responsabilité unique
- peut échouer indépendamment
- ne dépend que de ses entrées explicites
- peut être testée ou exécutée séparément

L’exécution de l’application est idempotente : relancer le script ne doit jamais créer de doublons ni modifier une rangée déjà traitée.

Les seules écritures persistantes autorisées sont :

- la création de fichiers Google Docs / PDF
- la création de brouillons Gmail
- la mise à jour des colonnes générées dans la feuille "Contrats"

# Invariants métier (V1)

- Une rangée de la feuille "Contrats" correspond à un contrat unique.
- Un contrat ne chevauche jamais deux mois civils.
- Un contrat est toujours lié à un seul salarié et un seul spectacle.
- La rangée 1 est toujours la rangée de headers et ne peut jamais être traitée.
- Les colonnes générées automatiquement ne doivent jamais être saisies manuellement.

# Principes de non-fonctionnement (V1)

L’application ne :

- modifie jamais les données saisies manuellement par l’utilisateur
- envoie jamais d’email automatiquement
- supprime jamais de fichiers existants
- interprète jamais un bulletin de paie PDF sans que l'utilisateur le demande explicitement
