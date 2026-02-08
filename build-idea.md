# Objectif

Automatiser les tâches répétitives au sein d'une boîte de production artistique française :

- Édition et envoi des contrats de travail
- Préparation des données de paie
- Envoi des bulletins de paie

**Contexte spécifique** : contrats d'artistes et de techniciens sous régime de l'intermittence + contrats de formateurs et intervenants au régime général.

# Utilisateur cible

Administrateur / producteur de compagnie artistique (1-3 personnes).

# MVP = définition minimale

- Un seul type de contrat : artiste intermittent
- Pas d'envoi email automatique : génération de brouillons Gmail
- Pas de parsing automatique des bulletins PDF (saisie manuelle des infos)
- Interface = Google Sheets directement (pas de front-end)
- Génération déclenchée manuellement en ligne de commande :
  - `node index.js generate <contrat_id>` : génère le Google Doc
  - `node index.js email-draft <contrat_id>` : crée le brouillon Gmail

# Flux principal (V1)

1. L'utilisateur remplit la feuille "Salariés", "Spectacles, "Représentations" et "Contrats" dans Google Sheets
2. L'utilisateur lance : `node index.js generate <contrat_id>`
3. L'app récupère les données du contrat (Google Sheets API)
4. L'app récupère les données du salarié et celles des répétitions et des représentations associées
5. L'app génère un Google Doc via template pré-défini
6. L'utilisateur lance : `node index.js email-draft <contrat_id>`
7. L'app exporte le Google Doc au format PDF dans un dossier temporaire
8. L'app crée un brouillon Gmail avec le PDF en pièce jointe
9. L'utilisateur relit le brouillon et l'envoie manuellement

# Structure des données (Google Sheets)

**Un seul fichier Google Sheet : "BD Embauches"**

## Feuilles et colonnes

### Salariés

- id (unique)
- nom
- prenom
- nom_de_scene
- adresse
- telephone
- email
- date_de_naissance
- pays de naissance
- ville de naissance
- departement de naissance
- insee (numéro de sécurité sociale)
- cs (caisse de congés spectacles)

### Spectacles

- id : unique
- titre
- code : titre abrégé du spectacle, sert comme suffix
- numero_objet (numéro administratif interne)
- nombre_de_representations

### Représentations

- id : unique
- spectacle_id : référence à Spectacles
- date
- lieu
- numero : les représentations avec le même spectacle_id constituent une série numérotée

## Répétitions

- id : unique
- spectacle_id : référence à Spectacles
- date
- lieu
- type : "service" ou "cachet"
- durée : si type = service

### Contrats

**Saisies par l'utilisateur**

- id (unique)
- salarie_id : référence à Salariés
- emploi : "Artiste musicien", "Chargé de production", etc.
- objet : intitulé du contrat
- statut : cadre / non cadre / artiste
- spectacle_id : chaque contrat d'artiste est lié à un spectacle
- repetitions_ids : référence à Répétitions, format texte : "id1,id2,id3"
- brut_par_repetition
- representations_ids : référence à Représentations, format texte : "id1,id2,id3"
- brut_par_representation
- defraiements : montant des défraiements forfaitaires figurant sur le contrat
- cout_bdp : coût d'émission du bdp
- analytique : code analytique de la paie
- contrat_doc : lien vers le contrat au format Google Docs
- contrat_pdf : lien vers le contrat au format PDF
- fichier_bdp : nom du fichier PDF du bdp

**Saisies dans V1, parsés depuis le bdp à à partir de V2**

- masse : masse salariale renseignée sur le bdp
- net : salaire net renseigné sur le bdp
- pas : prélèvement à la source, renseigné sur le bdp
- virement : montant viré au salarié

**Générés automatiquent pendant la saisie**

- brut_total : calcul à partir de (nb_repetitions × brut_par_repetition) + (nb_representations × brut_par_representation)
- date_debut : date minimale parmi repetitions_ids et representations_ids
- date_fin : date maximale parmi repetitions_ids et representations_ids
- message_virement : formaté selon le pattern défini
- ref_virement : formaté selon le pattern défini

**Note** : Ces champs sont calculés par des formules directement dans Google Sheets

**Générés automatiquent lors de l'exécution**

- lien_doc
- lien_pdf
- lien_email

**Note** : Google Sheets sert de base de données pour le MVP.
Migration vers MongoDB/PostgreSQL envisageable en V2.

# Cas d'usage type (exemple)

## Contexte

La compagnie "Les Bateleurs" embauche Marie Dupont (comédienne) pour 3 répétitions et 1 représentation du spectacle "Roméo et Juliette" en mars 2026.

## Étapes

1. L'admin vérifie que Marie est dans la feuille "Salariés" (id: 42)
2. L'admin vérifie que les 3 répétitions sont dans "Répétitions" (ids: 15, 16, 17)
3. L'admin vérifie que la représentation sont dans "Représentations" (id: 2)
4. L'admin crée une nouvelle ligne dans "Contrats" :
   - id: 123
   - salarie_id: 42
   - emploi: Comédienne
   - objet: "répétitions et représentations"
   - statut: "artiste"
   - repetitions_ids: "15,16,17"
   - salaire_par_répétition: 80
   - representations_ids: "2"
   - salaire_par_représentation: 240
   - defraiements: 45
   - cout_bdp : 13,5
   - analytique : MNJ
5. L'admin lance : `node index.js generate 123`
6. Un Google Doc "Contrat_Marie_Dupont_123.gdoc" est créé
7. L'admin vérifie le document généré
8. L'admin lance : `node index.js email-draft 123`
9. Un brouillon Gmail apparaît avec le doc en PJ, destinataire = email de Marie
10. L'admin relit, ajuste si besoin, et envoie

# Technologies

- Node.js
- Google Sheets API (base de données)
- Google Docs API (génération contrats)
- Gmail API (création brouillons)
- Google Drive API (stockage documents)

# Hors scope V1

- Suivi de l'avancement de l'embauche (DPAE, signature, paie effectuée)
- Multi-types de contrats (annexe 8 et régime général)
- Interface web dédiée pour l'exécution du script (front-end)
- Parsing automatique des PDF de bulletins (màj automatique des montants dans "Contrats")
- Création de templates à tiroirs / interface de génération de templates
- Gestion des avenants et modifications de contrats
- Système de permissions multi-utilisateurs
- Envoi automatique des emails (sans relecture)
- Base de données relationnelle (MongoDB/PostgreSQL = V2)
- Historique et archivage automatique
- Notifications et rappels
- Validation des données en temps réel dans Google Sheets
