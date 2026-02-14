# Objectif

Automatiser les tâches répétitives au sein d'une boîte de production artistique française liées à l'embauche :

- Édition et envoi des contrats de travail
- Préparation des données de paie
- Envoi des bulletins de paie
- Registre des montants des paies (masse, brut, net, etc.)
- Suivi des étapes de l'embauche, de la DPAE jusqu'à la paie

**Contexte spécifique :** Contrats à durée déterminée d'usage (CDDU) pour trois types de salariés :

- **Intermittents artistes** (régime des intermittents, prestations artistiques)
- **Intermittents techniciens** (régime des intermittents, prestations techniques)
- **Régime général** (formateurs, intervenants, etc.)

**Note :** Les intermittents artistes et techniciens relèvent respectivement des annexes 10 et 8 de la convention UNEDIC. Ces dénominations (annexe 8, annexe 10) pourront être utilisées dans les versions ultérieures pour référencer les textes réglementaires applicables.

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

4. **Pas d'envoi automatique des contrats**
   - En V1 : un brouillon Gmail est créé pour relecture et ajustement. L'utilisateur envoie manuellement après validation
   - En V2 : l'utilisateur choisit, pour chaque contrat, si l'application doit créer un brouillon de mail ou si elle doit envoyer le mail automatiquement

## Types de contrats en V1

### A10 — Intermittent artiste

- Associé à un spectacle
- `statut` = "artiste"
- Prestations :
  - **Répétitions** : N dates, même durée (heures), même brut unitaire pour toutes
  - **Représentations** : N dates, chacune avec son propre lieu et son propre brut
- Contrainte : au moins 1 répétition OU 1 représentation requise

### A8 — Intermittent technicien

- Associé à un spectacle
- `statut` = "cadre" ou "non-cadre"
- Prestations :
  - **Journées** : N dates, chacune avec son propre lieu, même durée (heures) et même brut unitaire pour toutes
- Contrainte : au moins 1 journée requise
- Note V2 : il sera possible de préciser la nature de chaque journée (montage, production, répétition, représentation, etc.)

### RG — Régime général

- Pas de spectacle associé
- `statut` = "cadre" ou "non-cadre"
- Prestation unique et permanente (pas de changement prévu en V2) :
  - date de début, date de fin, nature, lieu, brut total

## Champs ignorés selon le type de contrat

Certains champs de la feuille "Contrats" ne sont pertinents que pour certains types. En V1, si un champ non applicable à un type est renseigné, l'application affiche un avertissement dans la console mais continue le traitement normalement.

En V2 (avec front-end), les champs seront affichés ou masqués dynamiquement selon le type de contrat.

| Champ                                                                                           | A10    | A8     | RG     |
| ----------------------------------------------------------------------------------------------- | ------ | ------ | ------ |
| spectacle_id                                                                                    | ✓      | ✓      | ignoré |
| dateRepetition\*, dureeRepetitions, brut_par_repetition                                         | ✓      | ignoré | ignoré |
| dateRepresentation*, lieuRepresentation*, brutRepresentation\*                                  | ✓      | ignoré | ignoré |
| dateJournee*, lieuJournee*, dureeJournees, brut_par_journee                                     | ignoré | ✓      | ignoré |
| prestation_date_debut, prestation_date_fin, prestation_nature, prestation_lieu, prestation_brut | ignoré | ignoré | ✓      |

## Périmètre V1

**Fonctionnalités :**

- Génération de contrats Google Doc depuis un template (un par type : A10, A8, RG)
- Export automatique en PDF
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
node index.js [rangées]
```

**Exemples :**

- `node index.js` → Traite tous les contrats pas encore traités
- `node index.js 3` → Traite uniquement la rangée 3
- `node index.js 3-5 7` → Traite les rangées 3, 4, 5 et 7

## Gestion des erreurs

Si une erreur survient sur un contrat :

- L'erreur est affichée dans la console
- Le traitement continue avec les contrats suivants
- Un résumé final indique le nombre de succès et d'erreurs

# Exemple d'utilisation

## Contexte

La compagnie "Les Bateleurs" embauche Marie Dupont (comédienne, A10) pour 3 répétitions et 1 représentation du spectacle "Roméo et Juliette" en mai 2026.

## Étape 1 : Saisie dans Google Sheets

L'administrateur vérifie que Marie Dupont existe dans "Salariés" et que "Roméo et Juliette" existe dans "Spectacles" (code : "RJ").

Il crée une nouvelle ligne dans "Contrats" :

- Type : A10 / Salarié : Marou / Spectacle : RJ / Emploi : Comédienne
- 3 répétitions : 1, 2 et 17 mai (4h chacune, 80€/répétition)
- 1 représentation : 4 mai, Gazette Café de Montpellier (240€)
- Défraiements : 18€ / Coût BDP : 13,50€

Les formules Google Sheets calculent automatiquement :

- Brut total : 480€ (3×80 + 240)
- Date de début : 1 mai 2026 / Date de fin : 17 mai 2026
- Nom du fichier : "CDDU Marie Dupont - Comédienne (RJ) - 1-17 mai 2026"

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

L'administrateur ouvre le Google Doc (lien dans `url_doc`), vérifie le contrat, ouvre le brouillon Gmail, ajuste si nécessaire, et envoie.

## Étape 4 : Après réception du bulletin de paie

L'administrateur saisit manuellement dans le tableur :

- Masse salariale : 900€ / Net à payer : 450€ / Prélèvement à la source : 3,60€

La formule `message_virement` se met à jour :

```
SALAIRE COMEDIENNE RJ MAI 2026 - NET 450,00 - FRAIS 18
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

## Automatisation

- Envoi automatique d'emails sans relecture
- Parsing automatique des bulletins de paie PDF
- Notifications et rappels automatiques

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
