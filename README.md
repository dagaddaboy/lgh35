# Lägenhet 35 – 3D

Interaktiv 3D-modell av lägenheten med delad, flyttbar möblering.
Statisk sida på GitHub Pages och data i Firebase Firestore, med inloggning via Google.

| Fil | Innehåll |
|---|---|
| `index.html` | Hela appen (three.js + Firebase compat SDK från CDN) |
| `firebase-config.js` | Din Firebase-webbkonfiguration – **fyll i** |
| `firestore.rules` | Säkerhetsregler – **fyll i era e-postadresser** |
| `seed.json` | Nuvarande möblering (29 möbler), läggs in med en knapp i appen |

## 1. Firebase (ca 10 min)

Du kan använda samma projekt som budgetappen eller skapa ett nytt. Datan ligger i en egen samling, `lgh35_layouts`, så den krockar inte med något annat.

1. **Webbapp:** ⚙️ **Project settings** → **General** → **Your apps** → **Add app** → **Web** (`</>`). Under **SDK setup and configuration** → **Config**: kopiera värdena i `firebaseConfig` till `firebase-config.js`.
2. **Inloggning:** **Build** → **Authentication** → **Sign-in method** → **Add new provider** → **Google** → **Enable** → **Save**.
3. **Tillåten domän:** **Authentication** → **Settings** → **Authorized domains** → **Add domain** → `DITT-GITHUB-NAMN.github.io`.
4. **Databas:** **Build** → **Firestore Database** → **Create database**, om den inte redan finns. Välj location `eur3 (europe)` eller `europe-north1`.
5. **Regler:** **Firestore Database** → **Rules**. Klistra in `firestore.rules` och byt e-postadresserna mot era Google-konton, med gemener. Om projektet redan har regler för budgetappen klistrar du bara in `isHome()`-funktionen och `lgh35_layouts`-blocket bredvid de befintliga reglerna. Ersätt inte hela filen. Klicka **Publish**.

API-nyckeln i `firebase-config.js` är inte hemlig. Det är reglerna som skyddar datan.

## 2. GitHub Pages (ca 5 min)

1. **New repository** → namn `lgh35`. Det kan vara privat om du har GitHub Pro, annars **Public**. Datan är ändå skyddad av inloggningen.
2. **Add file** → **Upload files** → dra in de fem filerna → **Commit changes**.
3. **Settings** → **Pages** → **Source: Deploy from a branch** → **Branch: main / (root)** → **Save**.
4. Efter någon minut finns sidan på `https://DITT-GITHUB-NAMN.github.io/lgh35/`.

## Token för framtida uppdateringar

Profilbild → **Settings** → **Developer settings** → **Personal access tokens** → **Fine-grained tokens** → **Generate new token**:

- **Repository access:** **Only select repositories** → `lgh35`
- **Permissions** → **Repository permissions** → **Contents:** **Read and write**
- **Expiration:** t.ex. 30 days

## 3. Första gången

1. Öppna sidan och klicka **Logga in med Google**.
2. Klicka **Lägg in startmöbleringen**. Knappen visas bara när databasen är tom.
3. Dela länken med din sambo. Hon loggar in med kontot som står i reglerna.

## Felsökning

| Symptom | Åtgärd |
|---|---|
| "Firebase är inte konfigurerat ännu" | `firebase-config.js` har kvar platshållarna eller saknas i repot |
| "Kontot … har inte behörighet" | E-postadressen saknas i `firestore.rules`, eller reglerna är inte publicerade |
| Inloggningsfönstret stängs direkt / `auth/unauthorized-domain` | Lägg till `DITT-GITHUB-NAMN.github.io` under **Authentication → Settings → Authorized domains** |
| Ändringar syns inte hos den andra | Ladda om sidan. Kontrollera att båda är inne på samma variant |
