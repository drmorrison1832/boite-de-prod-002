# Architecture technique - V1

Ce document décrit la structure technique de l'application, l'organisation du code, et les décisions d'implémentation.

# Stack technique

## Runtime

- **Node.js** : Version 18+ (LTS)
- **Type de modules** : ES Modules (`"type": "module"` dans package.json)

## APIs Google

- **Google Sheets API v4** : Lecture et écriture dans les feuilles
- **Google Docs API v1** : Création de documents depuis template
- **Google Drive API v3** : Export PDF, gestion de dossiers
- **Gmail API v1** : Création de brouillons (pas d'envoi)

## Bibliothèques principales

- **googleapis** : Client officiel pour toutes les APIs Google
- **commander** : Parsing des arguments CLI (optionnel, peut être fait manuellement)

**Autres dépendances :** À définir pendant le développement selon les besoins.

## Authentification Google

### Méthode : OAuth 2.0

**Fichiers requis :**

```
credentials.json    # Client ID et secret (fourni par Google Cloud Console)
token.json          # Access token et refresh token (généré au premier lancement)
```

**Scopes requis :**

```
https://www.googleapis.com/auth/spreadsheets
https://www.googleapis.com/auth/documents
https://www.googleapis.com/auth/drive.file
https://www.googleapis.com/auth/gmail.compose
```

**Flux d'authentification (première fois) :**

1. L'app lit `credentials.json`
2. L'app génère une URL d'autorisation
3. L'utilisateur ouvre l'URL dans son navigateur
4. L'utilisateur autorise l'app
5. L'app reçoit le code d'autorisation
6. L'app échange le code contre des tokens
7. L'app sauvegarde les tokens dans `token.json`

**Flux d'authentification (suivant) :**

1. L'app lit `token.json`
2. Si le token est expiré, l'app le refresh automatiquement
3. L'app utilise le token pour les appels API

### Obtention des credentials

1. Créer un projet sur [Google Cloud Console](https://console.cloud.google.com/)
2. Activer les APIs : Sheets, Docs, Drive, Gmail
3. Créer des credentials OAuth 2.0 (type "Desktop app")
4. Télécharger le fichier JSON → renommer en `credentials.json`
5. Placer `credentials.json` à la racine du projet

**Note :** `credentials.json` et `token.json` ne doivent JAMAIS être versionnés (`.gitignore`).

# Structure du projet

```
embauches-automatisation/
│
├── src/
│   ├── cli/
│   │   └── parser.js              # Parse les arguments CLI
│   │
│   ├── loaders/
│   │   ├── sheets-loader.js       # Charge données depuis Google Sheets
│   │   └── config-loader.js       # Charge config.json
│   │
│   ├── validators/
│   │   └── contract-validator.js  # Valide les données d'un contrat
│   │
│   ├── builders/
│   │   └── contract-builder.js    # Construit la structure pour le template
│   │
│   ├── generators/
│   │   ├── doc-generator.js       # Génère Google Doc depuis template
│   │   ├── pdf-exporter.js        # Exporte Google Doc en PDF
│   │   └── email-composer.js      # Crée brouillon Gmail
│   │
│   ├── writers/
│   │   └── sheets-writer.js       # Écrit dans Google Sheets
│   │
│   ├── utils/
│   │   ├── google-auth.js         # Gestion authentification Google
│   │   └── logger.js              # Utilitaire de logging
│   │
│   └── orchestrator.js            # Coordonne toutes les étapes
│
├── index.js                       # Point d'entrée de l'application
│
├── config.json                    # Configuration (IDs des fichiers Google)
├── credentials.json               # Credentials OAuth Google (non versionné)
├── token.json                     # Tokens OAuth Google (non versionné)
│
├── .gitignore
├── package.json
├── package-lock.json
│
└── README.md
```

# Modules et responsabilités

## 1. CLI Parser (`src/cli/parser.js`)

### Responsabilité

Interpréter les arguments de ligne de commande et retourner la liste des rangées à traiter.

### Interface

```javascript
/**
 * Parse les arguments CLI
 * @param {string[]} argv - process.argv
 * @returns {{rows: number[] | null}} - null = toutes rangées non traitées
 * @throws {Error} - Si arguments invalides
 */
export function parseArgs(argv)
```

### Exemples

```javascript
parseArgs(["node", "index.js"]);
// → {rows: null}

parseArgs(["node", "index.js", "3"]);
// → {rows: [3]}

parseArgs(["node", "index.js", "3-5", "7"]);
// → {rows: [3, 4, 5, 7]}

parseArgs(["node", "index.js", "5-3"]);
// → Throws Error: "Plage invalide '5-3' (début > fin)"
```

### Logique

1. Extraire les arguments (ignorer `argv[0]` et `argv[1]`)
2. Pour chaque argument :
   - Si format `X-Y` : générer `[X, X+1, ..., Y]`
   - Si format `X` : ajouter `X`
   - Sinon : erreur
3. Filtrer la rangée 1 (headers)
4. Dédupliquer
5. Trier par ordre croissant

---

## 2. Sheets Loader (`src/loaders/sheets-loader.js`)

### Responsabilité

Charger les données depuis Google Sheets et les indexer pour accès rapide.

### Interface

```javascript
/**
 * Charge toutes les feuilles nécessaires
 * @param {string} spreadsheetId - ID du Google Sheet
 * @param {GoogleAuth} auth - Objet d'authentification
 * @returns {Promise<{
 *   salaries: Map<string, Object>,
 *   spectacles: Map<string, Object>,
 *   contrats: Array<{row: number, ...}>
 * }>}
 */
export async function loadSheets(spreadsheetId, auth)
```

### Détails d'implémentation

**Chargement des feuilles :**

```javascript
const response = await sheets.spreadsheets.values.batchGet({
  spreadsheetId,
  ranges: ["Salariés!A:Z", "Spectacles!A:Z", "Contrats!A:Z"],
});
```

**Indexation :**

- Salariés : `Map` avec clé = `id`
- Spectacles : `Map` avec clé = `code`
- Contrats : `Array` avec index = numéro de rangée - 2

**Parsing :**

- Dates : Convertir format Google Sheets → `Date` object
- Nombres : `parseFloat()`
- Cellules vides : `null`

**Structure retournée :**

```javascript
{
  salaries: Map {
    'Marou' => {
      id: 'Marou',
      nom: 'Dupont',
      prenom: 'Marie',
      email: 'marie@example.com',
      // ... tous les champs
    }
  },
  spectacles: Map {
    'RJ' => {
      code: 'RJ',
      titre: 'Roméo et Juliette',
      numero_objet: '2024-RJ-001'
    }
  },
  contrats: [
    // Index 0 = rangée 2
    {
      row: 2,
      salarie_id: 'Marou',
      spectacle_id: 'RJ',
      // ... tous les champs
    },
    // Index 1 = rangée 3
    {...}
  ]
}
```

---

## 3. Config Loader (`src/loaders/config-loader.js`)

### Responsabilité

Charger et valider le fichier de configuration.

### Interface

```javascript
/**
 * Charge la configuration
 * @param {string} path - Chemin vers config.json
 * @returns {Promise<Object>} - Configuration validée
 * @throws {Error} - Si fichier manquant ou invalide
 */
export async function loadConfig(path)
```

### Format du fichier `config.json`

```json
{
  "spreadsheetId": "1ABC...XYZ",
  "templateDocId": "1DEF...UVW",
  "folders": {
    "docs": "1GHI...RST",
    "pdfs": "1JKL...MNO"
  }
}
```

**Validation :**

- Tous les champs sont obligatoires
- Les IDs doivent avoir le bon format (alphanumériques, longueur appropriée)

---

## 4. Contract Validator (`src/validators/contract-validator.js`)

### Responsabilité

Valider qu'un contrat contient toutes les données nécessaires et cohérentes.

### Interface

```javascript
/**
 * Valide un contrat
 * @param {number} row - Numéro de rangée (pour messages d'erreur)
 * @param {Object} contract - Données du contrat
 * @param {Map} salaries - Index des salariés
 * @param {Map} spectacles - Index des spectacles
 * @returns {{valid: boolean, errors: string[]}}
 */
export function validateContract(row, contract, salaries, spectacles)
```

### Exemple de retour

```javascript
// Succès
{
  valid: true,
  errors: []
}

// Échec
{
  valid: false,
  errors: [
    "Salarié 'INCONNU' introuvable",
    "typeRepetition obligatoire si répétitions présentes"
  ]
}
```

### Validations effectuées

Voir `spec.md` section "Validation des données" pour la liste complète.

**Ordre des validations :**

1. Références (salarie_id, spectacle_id)
2. Email du salarié
3. Répétitions (si présentes)
4. Représentations (si présentes)
5. Contrainte globale (au moins 1 prestation)
6. Montants (>= 0)

**Fail-fast :** Dès qu'une validation échoue, retourner immédiatement.

---

## 5. Contract Builder (`src/builders/contract-builder.js`)

### Responsabilité

Construire la structure de données complète prête à être injectée dans le template Google Doc.

### Interface

```javascript
/**
 * Construit les données pour le template
 * @param {Object} contract - Données du contrat (de Sheets)
 * @param {Object} salarie - Données du salarié
 * @param {Object} spectacle - Données du spectacle
 * @returns {Object} - Structure complète pour template
 */
export function buildContract(contract, salarie, spectacle)
```

### Structure retournée

```javascript
{
  salarie: {
    nom: "Dupont",
    prenom: "Marie",
    nom_complet: "Marie Dupont",
    adresse: ["10 rue de la Paix", "75001 Paris", "France"],
    email: "marie.dupont@example.com",
    telephone: "06 12 34 56 78",
    date_naissance: "15/05/1990",
    lieu_naissance: "Paris (75)",
    insee: "1 90 05 75 123 456 78",
    cs: "AUDIENS123"
  },

  spectacle: {
    titre: "Roméo et Juliette",
    code: "RJ",
    numero_objet: "2024-RJ-001"
  },

  repetitions: [
    {
      date: "01/05/2026",
      date_texte: "jeudi 1er mai 2026",
      duree: 4,
      brut: 80
    },
    // ...
  ],

  representations: [
    {
      date: "04/05/2026",
      date_texte: "lundi 4 mai 2026",
      lieu: "Gazette Café de Montpellier",
      brut: 240
    }
  ],

  totaux: {
    nb_repetitions: 3,
    nb_representations: 1,
    brut_repetitions: 240,
    brut_representations: 240,
    brut_total: 480,
    defraiements: 18,
    cout_bdp: 13.5
  },

  dates: {
    debut: "01/05/2026",
    debut_texte: "1er mai 2026",
    fin: "17/05/2026",
    fin_texte: "17 mai 2026",
    signature: "09/02/2026",
    signature_texte: "9 février 2026"
  },

  nom_fichier: "CDDU Marie Dupont - Comédienne (RJ) - 1-17 mai 2026"
}
```

### Transformations principales

**Agrégation des répétitions :**

```javascript
const repetitions = [];
for (let i = 1; i <= 20; i++) {
  const date = contract[`dateRepetition${i}`];
  if (date) {
    repetitions.push({
      date: formatDate(date),
      date_texte: formatDateTexte(date),
      duree: contract.dureeRepetitions,
      brut: contract.brut_par_repetition,
    });
  }
}
```

**Agrégation des représentations :**

```javascript
const representations = [];
for (let i = 1; i <= 12; i++) {
  const date = contract[`dateRepresentation${i}`];
  if (date) {
    representations.push({
      date: formatDate(date),
      date_texte: formatDateTexte(date),
      lieu: contract[`lieuRepresentation${i}`],
      brut: contract[`brutRepresentation${i}`],
    });
  }
}
```

**Formatage de l'adresse :**

```javascript
const adresse = [
  salarie.adresse1,
  salarie.adresse2,
  salarie.adresse3,
  salarie.adresse4,
  salarie.adresse5,
].filter((line) => line != null && line.trim() !== "");
```

**Formatage du numéro INSEE :**

```javascript
// "190055012345678" → "1 90 05 75 123 456 78"
const insee = salarie.insee.replace(
  /(\d)(\d{2})(\d{2})(\d{2})(\d{3})(\d{3})(\d{2})/,
  "$1 $2 $3 $4 $5 $6 $7",
);
```

---

## 6. Doc Generator (`src/generators/doc-generator.js`)

### Responsabilité

Créer un Google Doc à partir du template en remplissant toutes les variables.

### Interface

```javascript
/**
 * Génère un Google Doc depuis le template
 * @param {string} templateId - ID du template Google Doc
 * @param {Object} contractData - Données du contrat (de builder)
 * @param {string} targetFolderId - ID du dossier Drive cible
 * @param {GoogleAuth} auth - Authentification
 * @returns {Promise<{docId: string, url: string}>}
 */
export async function generateDoc(templateId, contractData, targetFolderId, auth)
```

### Étapes

1. **Copier le template**

   ```javascript
   const copy = await drive.files.copy({
     fileId: templateId,
     requestBody: {
       name: contractData.nom_fichier,
       parents: [targetFolderId],
     },
   });
   const docId = copy.data.id;
   ```

2. **Remplir les variables simples**

   Utiliser `batchUpdate` avec des requêtes `replaceAllText` :

   ```javascript
   const requests = [
     {
       replaceAllText: {
         containsText: { text: "{{salarie.nom}}" },
         replaceText: contractData.salarie.nom,
       },
     },
     {
       replaceAllText: {
         containsText: { text: "{{salarie.prenom}}" },
         replaceText: contractData.salarie.prenom,
       },
     },
     // ... toutes les variables
   ];

   await docs.documents.batchUpdate({
     documentId: docId,
     requestBody: { requests },
   });
   ```

3. **Remplir les tableaux (répétitions, représentations)**

   **Option A :** Utiliser des tables Google Docs avec des lignes template
   - Trouver la table dans le document
   - Dupliquer la ligne template N fois
   - Remplir chaque ligne avec les données

   **Option B :** Utiliser une syntaxe de template plus simple
   - Trouver le bloc `{{#repetitions}}...{{/repetitions}}`
   - Générer le texte complet
   - Remplacer le bloc entier

   **Recommandation :** Option B pour V1 (plus simple)

4. **Retourner l'URL**
   ```javascript
   return {
     docId,
     url: `https://docs.google.com/document/d/${docId}/edit`,
   };
   ```

### Gestion des erreurs

- Si le template n'existe pas : erreur fatale
- Si le dossier cible n'existe pas : erreur fatale
- Si une variable n'est pas trouvée : warning (continuer quand même)

---

## 7. PDF Exporter (`src/generators/pdf-exporter.js`)

### Responsabilité

Exporter un Google Doc en PDF et le sauvegarder sur Drive.

### Interface

```javascript
/**
 * Exporte un Google Doc en PDF
 * @param {string} docId - ID du Google Doc
 * @param {string} nom_fichier - Nom pour le PDF (sans extension)
 * @param {string} targetFolderId - ID du dossier Drive cible
 * @param {GoogleAuth} auth - Authentification
 * @returns {Promise<{pdfId: string, url: string}>}
 */
export async function exportPDF(docId, nom_fichier, targetFolderId, auth)
```

### Étapes

1. **Exporter le doc en PDF (blob)**

   ```javascript
   const response = await drive.files.export(
     {
       fileId: docId,
       mimeType: "application/pdf",
     },
     { responseType: "arraybuffer" },
   );
   const pdfBuffer = Buffer.from(response.data);
   ```

2. **Uploader le PDF sur Drive**

   ```javascript
   const file = await drive.files.create({
     requestBody: {
       name: `${nom_fichier}.pdf`,
       parents: [targetFolderId],
       mimeType: "application/pdf",
     },
     media: {
       mimeType: "application/pdf",
       body: Readable.from(pdfBuffer),
     },
   });
   const pdfId = file.data.id;
   ```

3. **Retourner l'URL**
   ```javascript
   return {
     pdfId,
     url: `https://drive.google.com/file/d/${pdfId}/view`,
   };
   ```

---

## 8. Email Composer (`src/generators/email-composer.js`)

### Responsabilité

Créer un brouillon Gmail avec le PDF en pièce jointe.

### Interface

```javascript
/**
 * Crée un brouillon Gmail
 * @param {string} to - Email du destinataire
 * @param {string} subject - Objet du mail
 * @param {string} spectacle_titre - Titre du spectacle (pour le corps)
 * @param {string} salarie_prenom - Prénom du salarié (pour le corps)
 * @param {string} pdfId - ID du PDF sur Drive
 * @param {GoogleAuth} auth - Authentification
 * @returns {Promise<{draftId: string, url: string}>}
 */
export async function createDraft(to, subject, spectacle_titre, salarie_prenom, pdfId, auth)
```

### Étapes

1. **Télécharger le PDF depuis Drive**

   ```javascript
   const pdfResponse = await drive.files.get(
     { fileId: pdfId, alt: "media" },
     { responseType: "arraybuffer" },
   );
   const pdfBuffer = Buffer.from(pdfResponse.data);
   ```

2. **Construire le message MIME**

   ```javascript
   const boundary = "boundary123";
   const message = [
     `To: ${to}`,
     `Subject: ${subject}`,
     `MIME-Version: 1.0`,
     `Content-Type: multipart/mixed; boundary="${boundary}"`,
     "",
     `--${boundary}`,
     `Content-Type: text/html; charset=UTF-8`,
     "",
     `<p>Bonjour ${salarie_prenom},</p>`,
     `<p>Vous trouverez ci-joint votre contrat de travail pour le spectacle "${spectacle_titre}".</p>`,
     `<p>Merci de nous le retourner signé dans les meilleurs délais.</p>`,
     `<p>Cordialement,</p>`,
     "",
     `--${boundary}`,
     `Content-Type: application/pdf`,
     `Content-Transfer-Encoding: base64`,
     `Content-Disposition: attachment; filename="contrat.pdf"`,
     "",
     pdfBuffer.toString("base64"),
     `--${boundary}--`,
   ].join("\r\n");
   ```

3. **Créer le brouillon**

   ```javascript
   const draft = await gmail.users.drafts.create({
     userId: "me",
     requestBody: {
       message: {
         raw: Buffer.from(message).toString("base64url"),
       },
     },
   });
   const draftId = draft.data.id;
   ```

4. **Retourner l'URL**
   ```javascript
   return {
     draftId,
     url: `https://mail.google.com/mail/u/0/#drafts/${draftId}`,
   };
   ```

---

## 9. Sheets Writer (`src/writers/sheets-writer.js`)

### Responsabilité

Écrire les colonnes générées automatiquement dans Google Sheets.

### Interface

```javascript
/**
 * Met à jour une rangée dans Contrats
 * @param {string} spreadsheetId - ID du Google Sheet
 * @param {number} row - Numéro de rangée (2+)
 * @param {Object} updates - Colonnes à mettre à jour
 * @param {GoogleAuth} auth - Authentification
 * @returns {Promise<void>}
 */
export async function updateRow(spreadsheetId, row, updates, auth)
```

### Exemple d'utilisation

```javascript
await updateRow(
  spreadsheetId,
  2,
  {
    url_doc: "https://docs.google.com/document/d/...",
    url_pdf: "https://drive.google.com/file/d/...",
    date_signature_employeur: "2026-02-09 14:30:00",
    date_brouillon: "2026-02-09 14:30:15",
    url_brouillon: "https://mail.google.com/mail/u/0/#drafts/...",
  },
  auth,
);
```

### Implémentation

**Stratégie :** Écrire seulement les colonnes spécifiées (ne pas écraser tout)

```javascript
// Mapping des colonnes (à adapter selon votre structure)
const columnMap = {
  url_doc: "AA",
  url_pdf: "AB",
  date_signature_employeur: "AC",
  date_brouillon: "AD",
  url_brouillon: "AE",
};

const requests = Object.entries(updates).map(([field, value]) => ({
  range: `Contrats!${columnMap[field]}${row}`,
  values: [[value]],
}));

await sheets.spreadsheets.values.batchUpdate({
  spreadsheetId,
  requestBody: {
    valueInputOption: "RAW",
    data: requests,
  },
});
```

---

## 10. Orchestrator (`src/orchestrator.js`)

### Responsabilité

Coordonner toutes les étapes du traitement pour chaque rangée.

### Interface

```javascript
/**
 * Exécute le traitement principal
 * @param {number[]|null} rows - Rangées à traiter (null = auto)
 * @param {Object} config - Configuration chargée
 * @returns {Promise<void>}
 */
export async function run(rows, config)
```

### Pseudo-code

```javascript
export async function run(rows, config) {
  // 1. Authentification
  const auth = await authenticate();

  // 2. Charger les données
  const { salaries, spectacles, contrats } = await loadSheets(
    config.spreadsheetId,
    auth,
  );

  // 3. Sélectionner les rangées
  let toProcess;
  if (rows === null) {
    // Mode auto : seulement rangées avec date_brouillon = null
    toProcess = contrats
      .map((c, idx) => ({ ...c, row: idx + 2 }))
      .filter((c) => c.date_brouillon == null)
      .map((c) => c.row);
  } else {
    // Mode explicite : rangées demandées
    toProcess = rows.filter((r) => r !== 1); // Exclure headers
  }

  console.log(`Traitement de ${toProcess.length} rangée(s)...`);

  // 4. Traiter chaque rangée
  const results = { success: [], errors: [] };

  for (const row of toProcess) {
    try {
      const contract = contrats[row - 2]; // -2 car headers + index 0

      // Validation
      const validation = validateContract(row, contract, salaries, spectacles);
      if (!validation.valid) {
        throw new Error(validation.errors.join(", "));
      }

      // Construction des données
      const contractData = buildContract(
        contract,
        salaries.get(contract.salarie_id),
        spectacles.get(contract.spectacle_id),
      );

      // Génération Google Doc
      const { docId, url: url_doc } = await generateDoc(
        config.templateDocId,
        contractData,
        config.folders.docs,
        auth,
      );

      // Export PDF
      const { pdfId, url: url_pdf } = await exportPDF(
        docId,
        contractData.nom_fichier,
        config.folders.pdfs,
        auth,
      );

      // Création brouillon Gmail
      const { url: url_brouillon } = await createDraft(
        contractData.salarie.email,
        `Contrat - ${contractData.spectacle.titre} - ${contractData.dates.debut_texte}`,
        contractData.spectacle.titre,
        contractData.salarie.prenom,
        pdfId,
        auth,
      );

      // Mise à jour Sheets
      await updateRow(
        config.spreadsheetId,
        row,
        {
          url_doc,
          url_pdf,
          date_signature_employeur: new Date()
            .toISOString()
            .replace("T", " ")
            .slice(0, 19),
          date_brouillon: new Date()
            .toISOString()
            .replace("T", " ")
            .slice(0, 19),
          url_brouillon,
        },
        auth,
      );

      results.success.push(row);
      console.log(`✓ Rangée ${row}: "${contractData.nom_fichier}"`);
      console.log(`  - Google Doc créé`);
      console.log(`  - PDF exporté`);
      console.log(`  - Brouillon Gmail créé`);
    } catch (error) {
      results.errors.push({ row, error: error.message });
      console.error(`✗ Rangée ${row}: ${error.message}`);
    }
  }

  // 5. Résumé
  console.log(`\nTraitement terminé.`);
  console.log(`Succès: ${results.success.length} rangée(s)`);
  console.log(`Erreurs: ${results.errors.length} rangée(s)`);

  if (results.errors.length > 0) {
    console.log(
      `\nRangées en erreur: ${results.errors.map((e) => e.row).join(", ")}`,
    );
    console.log(`Consulter les logs ci-dessus pour détails.`);
  }
}
```

---

## 11. Main Entry Point (`index.js`)

### Code complet

```javascript
import { parseArgs } from "./src/cli/parser.js";
import { loadConfig } from "./src/loaders/config-loader.js";
import { run } from "./src/orchestrator.js";

async function main() {
  try {
    // Parser les arguments CLI
    const args = parseArgs(process.argv);

    // Charger la configuration
    const config = await loadConfig("./config.json");

    // Exécuter
    await run(args.rows, config);
  } catch (error) {
    console.error("✗✗✗ ERREUR FATALE:", error.message);
    process.exit(1);
  }
}

main();
```

# Principes de design

## 1. Fonctions pures

Les modules de transformation (`builder`, `validator`) sont des fonctions pures :

```javascript
// ✓ Pur (pas de side effects)
export function buildContract(contract, salarie, spectacle) {
  return {
    salarie: {...},
    // ...
  }
}

// ✗ Impur (lecture fichier, modification globale)
export function buildContract(contractId) {
  const contract = readFromDatabase(contractId) // Side effect
  globalState.lastContract = contract           // Side effect
  return {...}
}
```

**Avantages :**

- Testable facilement
- Prévisible
- Composable

## 2. Responsabilité unique

Chaque module a UNE responsabilité claire :

```javascript
// ✓ Une responsabilité
doc-generator.js  → Génère Google Doc
pdf-exporter.js   → Exporte PDF

// ✗ Responsabilités multiples
doc-pdf-manager.js → Génère doc ET exporte PDF ET envoie email
```

## 3. Fail-fast

Valider tôt, échouer tôt :

```javascript
// Au début du traitement
const validation = validateContract(row, contract, salaries, spectacles);
if (!validation.valid) {
  throw new Error(validation.errors.join(", "));
}

// Ensuite, traitement assume que données sont valides
// Pas besoin de re-vérifier à chaque étape
```

## 4. Isolation des erreurs

Une erreur sur une rangée n'affecte pas les autres :

```javascript
for (const row of toProcess) {
  try {
    // Traiter la rangée
  } catch (error) {
    // Logger et continuer
    console.error(`✗ Rangée ${row}: ${error.message}`);
    continue; // Passer à la suivante
  }
}
```

## 5. Idempotence

Relancer le script produit le même résultat :

```javascript
// Mode auto : ne traite que rangées avec date_brouillon = null
// → Relancer ne retraite pas les rangées déjà faites

// Mode explicite : écrase les anciennes valeurs
// → Relancer avec même rangée produit le même doc (nouveau, mais identique)
```

# Invariants et contraintes

## Invariants de données

1. **Rangée 1 = headers**
   - Jamais traitée
   - Utilisée pour identifier les colonnes

2. **Colonnes générées = read-only pour utilisateur**
   - `url_doc`, `url_pdf`, `date_brouillon`, etc.
   - Ne doivent jamais être saisies manuellement

3. **1 contrat = 1 mois**
   - `date_debut_contrat` et `date_fin_contrat` dans le même mois civil
   - (Non validé automatiquement en V1, mais documenté)

4. **1 contrat = 1 salarié + 1 spectacle**
   - Relation 1:1:1

## Invariants de process

1. **Idempotence**
   - `node index.js` (sans args) ne traite jamais deux fois la même rangée
   - `node index.js 5` peut retraiter la rangée 5

2. **Atomicité par rangée**
   - Une rangée est soit entièrement traitée, soit pas du tout
   - Si une étape échoue, aucune écriture dans Sheets pour cette rangée

3. **Ordre d'écriture**
   - Créer doc → créer PDF → créer brouillon → écrire Sheets
   - Si étape N échoue, étapes N+1 ne sont pas exécutées

## Garanties de sécurité

1. **Pas de suppression**
   - L'app ne supprime jamais de fichiers
   - L'app ne supprime jamais de rangées

2. **Pas de modification des saisies utilisateur**
   - L'app n'écrit QUE dans les colonnes générées
   - Jamais dans `salarie_id`, `emploi`, `dateRepetition1`, etc.

3. **Pas d'envoi automatique**
   - Création de brouillons uniquement
   - L'utilisateur envoie manuellement

# Gestion des erreurs

## Types d'erreurs

### 1. Erreurs de validation (non-fatales)

```javascript
class ValidationError extends Error {
  constructor(row, errors) {
    super(`Rangée ${row}: ${errors.join(", ")}`);
    this.name = "ValidationError";
    this.row = row;
    this.errors = errors;
  }
}
```

**Comportement :**

- Logger
- Passer à la rangée suivante
- Compter dans le résumé

### 2. Erreurs API (retry)

**Stratégie :** Retry avec exponential backoff

```javascript
async function withRetry(fn, maxAttempts = 3) {
  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await fn();
    } catch (error) {
      if (attempt === maxAttempts) throw error;
      const delay = Math.pow(2, attempt - 1) * 1000; // 1s, 2s, 4s
      console.warn(
        `⚠ Échec (${attempt}/${maxAttempts}), nouvelle tentative dans ${delay / 1000}s...`,
      );
      await sleep(delay);
    }
  }
}
```

### 3. Erreurs fatales

**Exemples :**

- Auth échouée
- `config.json` manquant
- Template introuvable
- Dossier Drive introuvable

**Comportement :**

- Logger
- Exit avec code 1
- Ne traiter AUCUNE rangée

## Logging

### Format

```
[TIMESTAMP] [LEVEL] Message
```

### Niveaux

- `INFO` : Progression normale
- `WARN` : Anomalie non-bloquante (retry)
- `ERROR` : Erreur sur une rangée
- `FATAL` : Erreur critique (exit)

### Implémentation simple

```javascript
// src/utils/logger.js
export const logger = {
  info: (msg) => console.log(`[INFO] ${msg}`),
  warn: (msg) => console.warn(`[WARN] ${msg}`),
  error: (msg) => console.error(`[ERROR] ${msg}`),
  fatal: (msg) => {
    console.error(`[FATAL] ${msg}`);
    process.exit(1);
  },
};
```

# Template Google Doc

## Structure du template

Le template est un Google Doc normal créé dans Google Drive avec des placeholders.

### Syntaxe des variables

**Variables simples :**

```
{{salarie.nom}}
{{salarie.prenom}}
{{spectacle.titre}}
{{dates.debut}}
```

**Tableaux (répétitions) :**

```
{{#repetitions}}
- Le {{date_texte}} de 14h à 18h ({{duree}}h) : {{brut}} €
{{/repetitions}}
```

**Tableaux (représentations) :**

```
{{#representations}}
- Le {{date_texte}} à {{lieu}} : {{brut}} €
{{/representations}}
```

**Conditionnels :**

```
{{#if repetitions}}
Le présent contrat comprend {{totaux.nb_repetitions}} répétition(s).
{{/if}}

{{#if representations}}
Le présent contrat comprend {{totaux.nb_representations}} représentation(s).
{{/if}}
```

### Exemple de template

```
CONTRAT DE TRAVAIL À DURÉE DÉTERMINÉE D'USAGE
Artiste du spectacle - Annexe X

Entre les soussignés :

[Nom de la compagnie]
Représentée par [Nom du signataire], [Titre]
Ci-après dénommée "l'Employeur"

Et :

{{salarie.nom_complet}}
Né(e) le {{salarie.date_naissance}} à {{salarie.lieu_naissance}}
Demeurant {{#salarie.adresse}}{{.}}
{{/salarie.adresse}}
N° Sécurité Sociale : {{salarie.insee}}
Ci-après dénommé(e) "le Salarié"

Il a été convenu ce qui suit :

Article 1 - Engagement

L'Employeur engage le Salarié en qualité de {{emploi}} pour le spectacle "{{spectacle.titre}}" (Objet n° {{spectacle.numero_objet}}).

Article 2 - Durée du contrat

Le présent contrat est conclu pour la période du {{dates.debut}} au {{dates.fin}}.

Article 3 - Prestations

{{#if repetitions}}
Le Salarié s'engage à participer aux répétitions suivantes :
{{#repetitions}}
- {{date_texte}} ({{duree}}h) : {{brut}} €
{{/repetitions}}
Total répétitions : {{totaux.brut_repetitions}} €
{{/if}}

{{#if representations}}
Le Salarié s'engage à participer aux représentations suivantes :
{{#representations}}
- {{date_texte}} à {{lieu}} : {{brut}} €
{{/representations}}
Total représentations : {{totaux.brut_representations}} €
{{/if}}

Article 4 - Rémunération

Salaire brut total : {{totaux.brut_total}} €
Défraiements forfaitaires : {{totaux.defraiements}} €

Fait à Paris, le {{dates.signature_texte}}

[Signature scannée de l'employeur]

[Nom du signataire]
[Titre]
```

## Création du template

1. Créer un Google Doc dans Drive
2. Remplir avec le texte et les placeholders
3. Insérer l'image de signature scannée
4. Noter l'ID du document (dans l'URL)
5. Partager le document avec le compte qui exécutera le script

# Configuration

## Fichier `config.json`

```json
{
  "spreadsheetId": "1ABC...XYZ",
  "templateDocId": "1DEF...UVW",
  "folders": {
    "docs": "1GHI...RST",
    "pdfs": "1JKL...MNO"
  }
}
```

## Obtention des IDs

**Google Sheet :**

```
URL: https://docs.google.com/spreadsheets/d/1ABC...XYZ/edit
ID:  1ABC...XYZ
```

**Template Google Doc :**

```
URL: https://docs.google.com/document/d/1DEF...UVW/edit
ID:  1DEF...UVW
```

**Dossiers Google Drive :**

```
URL: https://drive.google.com/drive/folders/1GHI...RST
ID:  1GHI...RST
```

## Permissions

Le compte Google qui exécute le script doit avoir :

- **Google Sheet** : Accès "Éditeur"
- **Template** : Accès "Lecteur" (copie autorisée)
- **Dossiers** : Accès "Éditeur" (pour créer des fichiers)

# Évolutions V2

## Changements prévus

### 1. Base de données MongoDB

**Migration :**

- `loaders/sheets-loader.js` → `loaders/db-loader.js`
- Schema Mongoose pour Salariés, Spectacles, Contrats, Répétitions, Représentations

**Avantages :**

- Relations many-to-many
- Requêtes plus performantes
- Validations au niveau DB

### 2. Tarifs variables par répétition

**Schéma :**

```javascript
{
  contract_id: "...",
  repetitions: [
    {date: "2026-05-01", duree: 3, brut: 80},
    {date: "2026-05-02", duree: 4, brut: 100}
  ]
}
```

**Impact :** Module `builder` doit lire depuis collection Répétitions

### 3. Parsing automatique des BDP

**Nouveau module :**

- `src/parsers/bdp-parser.js`
- Lit PDF du bulletin
- Extrait : masse, net, pas
- Écrit dans DB

**Librairie :** `pdf-parse` ou `pdfjs-dist`

### 4. Commandes séparées

```bash
node index.js gdocs [rows]        # Seulement génération docs
node index.js pdf [rows]          # Seulement export PDF
node index.js email-draft [rows]  # Seulement brouillons
```

**Impact :** Refactoriser `orchestrator.js` pour permettre exécution partielle

## Modules à ajouter (V2)

```
src/
  db/
    connection.js      # Connexion MongoDB
    models/            # Schémas Mongoose
      Salarie.js
      Spectacle.js
      Contract.js
      Repetition.js
      Representation.js

  parsers/
    bdp-parser.js      # Extraction PDF bulletins
```

## Modules à modifier (V2)

- `loaders/` : Charger depuis MongoDB au lieu de Sheets
- `builders/` : Gérer répétitions avec tarifs variables
- `orchestrator.js` : Permettre exécution partielle (gdocs seul, etc.)

## Modules inchangés (V2)

- `generators/` : Logique de génération identique
- `validators/` : Logique de validation similaire
- `cli/parser.js` : Parsing CLI identique

---

**Pour les spécifications fonctionnelles, voir :**

- `spec.md` : Comportement attendu de l'application
