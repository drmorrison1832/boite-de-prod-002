# Objectif

Automatiser les tâches répétitives au sein d'une boîte de production artistique française :

- Édition et envoi des contrats de travail
- Préparation des données de paie
- Envoi des bulletins de paie

**Contexte spécifique** : contrats d'artistes et de techniciens sous régime de l'intermittence + contrats de formateurs et intervenants au régime général. Les contrats ne chevauchent pas entre deux mois; des missions dans des mois différents donnent lieu à des contrats séparés.

# Utilisateur cible

Administrateur / producteur de compagnie artistique (1-3 personnes).

# MVP = définition minimale

`node index.js [commande]

- Un seul type de contrat : artiste intermittent
- Pas d'envoi email automatique : génération de brouillons Gmail
- Pas de parsing automatique des bulletins PDF (saisie manuelle des infos)
- Interface = Google Sheets directement (pas de front-end)
- Base de données = Google Sheets
- Génération déclenchée manuellement en ligne de commande : `node index.js [commande] [lignes]`

# Flux principal (V1)

_L'éxécution se déroule par étapes. L'éxécution d'une étape enclenche celle des étapes précédentes: Création du Google Doc -> Création du PDF -> Création du brouillon Gmail -> Envoie automatique du mail (V2+)_

_Cas général: l'utilisateur ne spécifie pas de ligne : `node index.js [commande]`_

1. L'utilisateur remplit la feuille "Salariés", "Spectacles" et "Contrats" dans Google Sheets
2. L'utilisateur lance l'application
3. L'app récupère les données du contrat via Google Sheets API
4. L'app récupère les données du salarié et du spectacle
5. L'app génère un Google Doc via template pré-défini et l'archive dans un dossier prédéfini
6. L'app écrit l'url du Google Doc dans Google Sheets (colonne url_doc)

[Si la commande était `gdoc` ou équivalente, l'exécution s'achèce ici. Si c'était `pdf` ou `mail-draft`, elle continue]

7. L'app exporte le Google Doc au format PDF et l'archive dans un dossier prédéfini.

[Si la commande était `pdf`, l'exécution s'achèce ici. Si c'était `mail-draft`, elle continue.]

8. L'app crée un brouillon Gmail avec le PDF en pièce jointe et écrit date_brouillon dans Google Sheets
9. L'utilisateur relit les brouillons dans Gmail et les envoie manuellement

Si l'exécution précisait le numéro de la rangée, l'

# Structure des données (Google Sheets)

**Un seul fichier Google Sheet : "BD Embauches"**

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
- code : titre abrégé, 2-4 lettres (ex: "RJ" pour Roméo et Juliette) - utilisé pour message_virement, pour ref_virement et pour nomer les fichiers du contrat (GDocs et PDF)
- numero_objet : numéro administratif interne généré par France Travail (texte)

### Contrats

**Saisies par l'utilisateur**

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

_Note : dans V1, toutes les répétitions d'un même contrat ont :_
_- le même type (service ou cachet)_
_- la même durée (dureeRepetitions)_
_- le même salaire brut (brut_par_repetition)_
\_En V2, chaque répétition pourra avoir sa propre durée et son propre salaire.\_

**Générés automatiquement pendant la saisie (par des formules directement dans Google Sheets)**

- nom_fichier : concaténation "CDDU [prenom] [nom] - [emploi] ([spectacle_id]) - [periode allant de date_debut_contrat à date_fin_contrat, format texte]"
- brut_total
- date_debut_contrat : date de la première répétition/représentation
- date_fin_contrat : date de la dernière répétition/représentation
- message_virement : généré par formule après saisie de 'net' ; "SALAIRE EMPLOI [spectacle_id] [MOIS ANNEE] - NET [à compléter] - FRAIS [FRAIS]" (se met à jour automatiquement quand 'net' est saisi) ; ex. : "SALAIRE CHARGEE DE PROD RJ MAI 2026 - NET 314,12 - FRAIS 50"
- ref_virement : "SALAIRE [PRENOM] [NOM] - [EMPLOI] [SPECTACLE_ID] - [MOIS ANNEE]"; ex. : "SALAIRE MARIE DUPONT - CHARGEE DE PROD RJ - MAI 2026" (texte normalisée selon normes bancaires: tout en majuscules, pas d'accents, seulement certains symboles sont authorisés)
- fichier_bdp : nom que l'utilisateur donnera au bulletin de paie, à définir, ex. "PAIE [PRENOM] [NOM] - [EMPLOI] [spectacle_id] - [MOIS ANNEE]"

**Saisies dans V1, parsés depuis le bdp à partir de V2+**

- masse : masse salariale renseignée sur le bdp
- net : salaire net renseigné sur le bdp
- pas : prélèvement à la source, renseigné sur le bdp
- virement : montant qui doit être viré au salarié

_Note : En V1, ces champs sont remplis manuellement par l'utilisateur après réception du BDP. En V2, ils seront parsés automatiquement depuis le PDF._

**Générés automatiquement lors de l'exécution**

- url_doc : généré lors de la création Google Doc
- url_pdf :
  - Généré lors de `node index.js pdf` ou `node index.js pdf <ligne>` (stockage permanent sur Drive)
  - Reste vide si on passe directement à `email-draft` (PDF temporaire seulement)
  - Effacé après `email-draft` si existant (pour indiquer que le PDF archivé n'est plus à jour)
- date_signature_employeur : date de création du contrat (la signature est intégrée au template, le contrat est signé dès son édition)
- date_brouillon : date de creation du brouillon de l'email

_Note : Google Sheets sert de base de données pour le MVP._
_Migration vers MongoDB/PostgreSQL envisageable en V2._

# Cas d'usage type (exemple)

## Contexte

La compagnie "Les Bateleurs" embauche Marie Dupont (comédienne) pour 3 répétitions et 1 représentation du spectacle "Roméo et Juliette" en mai 2026.

## Étapes

1. L'admin vérifie que Marie est dans la feuille "Salariés" (id: Marou)
2. L'admin crée une nouvelle ligne dans "Contrats" :
   - salarie_id: "Marou"
   - spectacle_id: "RJ"
   - emploi: Comédienne
   - objet: "répétitions et représentations"
   - statut: "artiste"
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
3. L'admin lance : `node index.js gdocs`
4. Un Google Doc "CDDU Marie Dupont - Comédienne (RJ) - 1-17 mai 2026" est créé
5. L'admin vérifie le document généré
6. L'admin lance : `node index.js email-draft`
7. Un brouillon Gmail apparaît avec le PDF en PJ, destinataire = email de Marie
8. L'admin relit, ajuste si besoin, et envoie

# Technologies

- Node.js
- Google Sheets API (base de données)
- Google Docs API (génération contrats)
- Gmail API (création brouillons)
- Google Drive API (stockage documents)

# Hors scope V1

- Base de données relationnelle (MongoDB/PostgreSQL = V2)
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
- Envoi automatique des emails (sans relecture)
- Historique et archivage automatique
- Notifications et rappels
