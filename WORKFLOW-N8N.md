# Workflow n8n — Réception BL → Google Drive

## Architecture du workflow

```
[Webhook] → [Function: préparer les données] → [Google Drive: créer dossiers] → [Google Drive: upload photos] → [Respond to Webhook]
```

## Étape 1 : Webhook (Trigger)

- **Type** : Webhook
- **Méthode** : POST  
- **Path** : `/reception-bl`
- **Response Mode** : Last Node

Le webhook reçoit un JSON avec cette structure :
```json
{
  "labo": "BIOGARAN",
  "receptionnaire": "Xavier",
  "date": "2026-02-15T10:30:00.000Z",
  "photos": [
    {
      "index": 1,
      "data": "data:image/jpeg;base64,....",
      "filename": "BL_page_1.jpg"
    }
  ],
  "site": "NAST"
}
```

## Étape 2 : Function Node — Préparer les chemins

```javascript
const data = $input.first().json;
const date = new Date(data.date);
const year = date.getFullYear().toString();
const month = String(date.getMonth() + 1).padStart(2, '0');
const day = String(date.getDate()).padStart(2, '0');

// Mois en français pour le nom du dossier
const moisFr = [
  'Janvier', 'Février', 'Mars', 'Avril', 'Mai', 'Juin',
  'Juillet', 'Août', 'Septembre', 'Octobre', 'Novembre', 'Décembre'
];

const labo = data.labo.toUpperCase().trim();
const monthFolder = `${month} - ${moisFr[date.getMonth()]}`;

// Structure : Réceptions / [LABO] / [ANNÉE] / [MOIS] / fichier
const folderPath = `Réceptions/${labo}/${year}/${monthFolder}`;

// Préparer chaque photo pour l'upload
const photos = data.photos.map((photo, i) => {
  // Nom du fichier : BL_LABO_DATE_PAGE.jpg
  const filename = `BL_${labo}_${year}${month}${day}_page${photo.index}.jpg`;
  return {
    ...photo,
    finalFilename: filename,
    base64Data: photo.data.replace(/^data:image\/\w+;base64,/, ''),
    folderPath: folderPath
  };
});

return photos.map(photo => ({
  json: {
    ...data,
    folderPath: photo.folderPath,
    filename: photo.finalFilename,
    base64Data: photo.base64Data,
    labo: labo,
    year: year,
    monthFolder: monthFolder
  }
}));
```

## Étape 3 : Google Drive — Créer les dossiers (si nécessaire)

Utilise le node **Google Drive** en mode "Create Folder" pour créer la hiérarchie :

1. Dossier racine `Réceptions` (à créer une seule fois)
2. Sous-dossier `[LABO]`  
3. Sous-dossier `[ANNÉE]`
4. Sous-dossier `[MOIS]`

**Astuce** : Utilise un node **Function** pour vérifier/créer chaque niveau de dossier via l'API Google Drive, ou utilise le node HTTP Request avec l'API Google Drive v3.

### Alternative plus simple : Node HTTP Request

```javascript
// Dans un Function node, utiliser l'API Google Drive pour :
// 1. Chercher si le dossier existe (files.list avec query)
// 2. Créer s'il n'existe pas (files.create)

// Query pour chercher un dossier :
// name='BIOGARAN' and mimeType='application/vnd.google-apps.folder' and 'PARENT_ID' in parents
```

## Étape 4 : Google Drive — Upload des photos

- **Node** : Google Drive (Upload)
- **Binary Property** : Convertir le base64 en binaire d'abord
- **Filename** : `{{ $json.filename }}`
- **Parent Folder** : L'ID du dossier créé à l'étape 3

### Conversion base64 → binaire

Ajoute un node **Function** avant l'upload :
```javascript
const items = $input.all();
const results = [];

for (const item of items) {
  const binaryData = Buffer.from(item.json.base64Data, 'base64');
  
  results.push({
    json: item.json,
    binary: {
      file: {
        data: binaryData.toString('base64'),
        mimeType: 'image/jpeg',
        fileName: item.json.filename
      }
    }
  });
}

return results;
```

## Étape 5 : Respond to Webhook

Renvoie un JSON de confirmation :
```json
{
  "success": true,
  "message": "BL uploadé",
  "labo": "BIOGARAN",
  "files": 2,
  "path": "Réceptions/BIOGARAN/2026/02 - Février/"
}
```

## Résumé de la structure Google Drive

```
📁 Réceptions/
  📁 ARROW/
    📁 2026/
      📁 01 - Janvier/
      📁 02 - Février/
        📄 BL_ARROW_20260215_page1.jpg
        📄 BL_ARROW_20260215_page2.jpg
  📁 BIOGARAN/
    📁 2026/
      📁 02 - Février/
        📄 BL_BIOGARAN_20260215_page1.jpg
  📁 SANDOZ/
  📁 SANOFI/
  ...
```

## Notes importantes

1. **Credential Google Drive** : Utilise ton OAuth2 existant dans n8n
2. **ID du dossier racine** : Note l'ID du dossier "Réceptions" dans Google Drive et mets-le en variable n8n
3. **Taille des photos** : Les photos en base64 sont ~33% plus grosses. Pour des BL, ça devrait rester sous la limite du webhook n8n (16MB par défaut)
4. **Compression** : Tu peux ajouter un node pour compresser les images avant upload si nécessaire
