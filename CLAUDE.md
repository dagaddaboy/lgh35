# LGH 35 – 3D-lägenhet

Interaktiv 3D-modell av Henriks lägenhet i Göteborg (79,4 m², 3 rok, balkong ≈15 m²). Möbleringen delas med sambon via Firebase.
Sajt: https://dagaddaboy.github.io/lgh35/ (GitHub Pages från `main`, roten). Firebase-projekt: `apartment-plan-8abec`.

Svara och skriv UI-texter på **svenska**. Henrik vill ha korta, sakliga svar med siffror.

## Arbetsregler

- **Ändra aldrig lägenhetens geometri av misstag.** Väggar, öppningar, kök och fast inredning är uppmätta. Kör röktestet och jämför fingeravtrycket (se nedan) efter varje ändring.
- Bygg vidare på befintlig kod och skriv inte om från grunden. Behåll alla funktioner och element-id:n som UI och test använder.
- Jobba stegvis, testa och committa per steg. Pusha till `main` först när Henrik har sagt ok. Testa större förändringar som `test.html` innan.
- Inga byggsteg. Allt ligger i `index.html` (HTML + CSS + JS), och biblioteken laddas från jsDelivr med exakta versioner.

## Filer

| Fil | Innehåll |
|---|---|
| `index.html` | Hela appen |
| `firebase-config.js` | `window.FIREBASE_CONFIG = {...}`. Apinyckeln är inte hemlig, det är reglerna som skyddar datan |
| `firestore.rules` | Regler (mallversion; den publicerade har Henriks och sambons e-post) |
| `seed.json` | Startmöblering. Knappen *Lägg in startmöbleringen* visas bara när databasen är tom |
| `tools/qa/` | Röktest med Playwright och låtsas-Firebase |
| `demo-r170.html`, `index.html.html` | Gamla rester som kan tas bort |

## Arkitektur i `index.html`

- **Laddning:** ett `<script type="module">` importerar three **r170** (importmap mot `three@0.170.0`) med OrbitControls, BufferGeometryUtils, RoomEnvironment, EffectComposer/RenderPass/GTAOPass/OutputPass. Det sätter `window.THREE` (shim: `THREE.BufferGeometryUtils.mergeBufferGeometries = mergeGeometries`) och `window.MODERN`, och kör sedan appen.
- **Appen** ligger i `<script type="text/plain" id="appsrc">` och körs som **klassiskt skript (inte strict mode)**, inslagen i en IIFE. Gör den inte till en modul utan att granska block-scopade funktioner.
- Path tracern laddas vid behov: `window.loadPT()` → `three-gpu-pathtracer@0.0.23` + `three-mesh-bvh@0.8.3`. Nyare pathtracer-versioner kräver three ≥ 0.180/0.185.
- Firebase compat SDK 10.12.2 (app, auth, firestore) med Google-inloggning.
- Sektioner i koden (sök på `// ----------`): renderer/scene, performance, walls with openings, floors, lamps, furniture builders (`CAT`), labels, room list, camera, Firestore, phase 2 (katalog/fästning/mått/mätverktyg), walk mode, doors, sun, evening, phase 1 UI, loop, rendering.

### Koordinater

- Meter. Modellens origo ligger ungefär i nordvästra hörnet. `root.position = (-CX, 0, -CZ)` med `CX=4.6`, `CZ=4.9`, så **world = model − (CX, 0, CZ)**.
- Takhöjd `H = 2.7`. Balkongen ligger på −x-sidan (räcke vid x = −1.35, smal del x −1.35…−0.3, bred del x −1.35…1.26, z 5.46…9.06). Balkongen vetter mot 240°, och −x i modellen är 240°.
- Kompass: N = (0.5, 0, −0.866), E = (0.866, 0, 0.5).
- Golv-AABB: x −0.3…10.13, z −0.25…10.27.

### Scen och grupper

- `walls`, `fixed` (kök, badrum m.m.), `ctx` (omgivning) och `furn` (möbler) ligger under `root`.
- `freezeStatic()` beräknar `STATIC_BOXES` (världs-AABB `[x0,x1,z0,z1]` för kollision och fästning) och slår sedan ihop statiska meshar per material. Undantag: `noMerge`, dörrar, lampor och transparenta material.
- Delad `UNIT_BOX` som skalas per mesh, och `CYL_CACHE`. Möbler slås ihop per objekt i `buildItem`.
- **Rendering på begäran:** `tick` ritar varje bildruta bara medan något händer och 1,2 s efteråt (`poke()`, `wakeUntil`), annars ungefär 1 gång per sekund. `markShadow()`, kamerarörelser, inmatning och storleksändring anropar `poke()`. Kod som ändrar scenen på annat sätt måste anropa `poke()` eller `markShadow()`.
- **Adaptiv upplösning** (`adapt`, `adp`): om bildrutorna tar över 42 ms i snitt under interaktion sänks pixelkvoten i steg om 0,25 ner till 1, och därefter stängs AO av för sessionen.
- Skuggor: `renderer.shadowMap.autoUpdate=false`. Anropa **`markShadow()`** efter allt som flyttar geometri.
- Klippning för vägghöjd: `C=[clip]` på alla material. Det osynliga taket (`colorWrite:false`) kastar solskugga.

### Material och grafik (Fas 3)

- `NeutralToneMapping`, sRGB, PMREM av `RoomEnvironment` som `scene.environment`. `scene.environmentIntensity` styrs av solen.
- Ljuset använder fysiska enheter. Gamla intensiteter är multiplicerade med π: sol `s*2.6π`, hemi `(.06+.3*day)π`, lampor `int*.7*π*2.8`.
- Ek (`mat.oak`) samplas i **världskoordinater** via `worldMapped()` (`onBeforeCompile`). Plankorna är 10 cm breda och ligger längs modellens x-axel: `uWS = (1/.60, 1/4.0)`, ovansidan använder `vWP.zx`.
- Kvalitet `QUAL = {hog, normal, lag}`. Hög ger GTAO via EffectComposer, där GTAO:s normalpass döljer `colorWrite:false`-, transparenta och Basic-material och respekterar klippningen. Valet sparas i `lgh35-quality`.

### Möbler och data

- Firestore: `lgh35_layouts/{variant}` = `{name, order}` och `lgh35_layouts/{variant}/items/{id}` = `{type, x, z, rot, w, d, y?, color?}` i modellkoordinater (m, grader).
- `CAT[type] = {name, w, d, m, b(g,w,d,m)}`. Byggaren har lokal ram med bredd längs x, djup längs z och framsidan mot +z. Kategorierna för katalogen finns i `CATS`.
- Import/export-JSON: `{"items": {id: {...} | null}}`, där `null` tar bort. Exporten lägger även till `variant`.
- Ångra/gör om: `hist` med operationerna `patch`, `add` och `del`, per variant och bara lokalt.
- **Tavlor och speglar** (`ART_CAT`, gruppen *Tavlor & speglar*): bilderna är beskurna och perspektivkorrigerade från Henriks foton och ligger inbäddade som JPEG i `ART_TEX`. För dessa typer är `d` = **höjd** (inspektören visar *Höjd*), `y` = underkant över golv (standard 1,3 m), tjocklek 2,5 cm, framsidan mot +z (rot 90 = mot +x, −90 = mot −x, 180 = mot −z). Placera mitten 1,3 cm från väggytan. De fäster mot väggar även högt upp. Startplaceringen finns i `tools/qa/fixtures/tavlor.json`.
- Fästning (`snapTo`, 8 cm) och avståndsmått (`updateAids`) räknar på AABB, så snett vridna möbler blir något större.

### Övrigt

- **Gå runt:** `walk`, där `collectBoxes()` är dörrbladens sampel + `STATIC_BOXES` + möblernas AABB, radie 0.2. Ögonhöjd = längd × 0.93 (standard 179 cm).
- **Dörrar:** `doors[]`. Klick växlar, och läget sparas i `lgh35-door-*`. Balkongdörrarna öppnas 88°.
- **Lampor:** `lamps` (Map) och `setLamp(L,on)`, som bara växlar intensitet (konstant antal ljus = ingen omkompilering av shaders). Läget sparas i `lgh35-lamp-*`.
- **Sol:** förenklad NOAA-beräkning för Göteborg med svensk sommartid. Standard är kl. 13:00. Variablerna `DARK`, `EVE` och `SUNDAY`. *Tvinga kväll* sparas inte.
- **Rendering** (`R`, `startRender`/`stopRender`): döljer tak och poché, lägger till ett vitt tak om kameran är under H, sätter världs-UV på ekmeshar, gradient-himmel som env och bakgrund, och osynliga `RectAreaLight`-"portaler" i fönster och balkongdörrar (`PORTALS`). Exponering via `renderExp`.
- **localStorage-nycklar:** `lgh35-layout`, `-tab`, `-quality`, `-snap`, `-kitchen`, `-height`, `-door-*`, `-lamp-*`.
- **Rum (`data-id`):** `vard`, `kok`, `smyg`, `sov1`, `sov2` (Rosas rum), `bad`, `wc`, `hall`, `klk`, `balk`.

## Röktest

Se `tools/qa/README.md`. Kör `tools/qa/run.sh`, eller `GPU=1 tools/qa/run.sh` för ditt riktiga grafikkort. Referensfingeravtryck för `fp` efter import av alla fixtures:

```
walls  tri 2256   sx -2657.61  sy 9676.75  sz -3352.39
fixed  tri 33440  sx -208004.97 sy 117596.83 sz -205962.19
ctx    tri 264    sx -750.51   sy 337.32   sz -522.18
furn   tri 27966  sx 34296.72  sy 61657.95 sz 125267.49
```

Även `errors: []`, `lamps: 21`, `doors: 8`, `boxCount: 248`, och att `programsAfterLampToggle` är lika med `nightPrograms`.

## Status och plan

- Köket, planerat (`KITCHEN.plan`): IKEA HAVSTORP beige luckor (`0xb8ad9e`) och LOCKEBO ljusbeige bänkskiva/sockel (`0xd6cab7` + prickig textur `stoneTex.speck`, världsmappad via `worldMapped`). `applyKitchen` byter färg och textur (idag: `stoneTex.plain`).
- Köket: bänkskivan är delad runt spisen (x 2,66–3,26) och diskbänkens urtag (`SU`, x 0,52–1,82). Diskbänken (foto 2026-11-21) är en rostfri enhet: räfflad torkyta till vänster över diskmaskinen, stor rundad ho och liten ho till höger (`BOWLS`, ExtrudeGeometry via `rrPath`/`slab`). `SINK.now` = ilagd ram 6 mm över skivan, `SINK.plan` = samma enhet plan med skivan. Spisen är modellerad efter er SMEG (60 cm, induktion, 3+3 rattar, display, sockellist).
- Köket: handtagen ligger i `KH.now` (stålstänger) och `KH.plan` (Linfalk 1353 nickel, cc 128, `nickelHandle()`), undergrupper som slås ihop var för sig, växlas i `applyKitchen` och har `userData.noCollide`.
- Köket: överskåpen finns i två versioner (`UPPERS.now` 90 cm luckor, överkant 2,35; `UPPERS.plan` 100 cm, överkant 2,45), egna undergrupper i `fixed` som inte slås ihop och växlas i `applyKitchen`.
- Klädkammaren: skalet (hatthyllor + övre stänger) ligger i `fixed`. Innehållet är två katalogtyper, `klkIdag` (foto 2026-11-16) och `klkForslag` (förslaget från `klk-forslag.html`), placerade i x 6.66, z 3.815, så att en möbleringsvariant kan visa det ena eller det andra. `seed.json` har `klk-idag`. Deras kollisionsytor för gå runt-läget ligger i `g.userData.cb`.
- Klart: Fas 0 (prestanda), Fas 1 (gränssnitt med flikar, inspektör, solreglage och mobilpanel), Fas 2 (möbleringsverktyg), Fas 3 (grafik r170), rendering i webbläsaren och 10 cm parkett.
- Renderingen är bara provad med mjukvarugrafik (≈8 s per omgång). Den behöver verifieras och kalibreras på riktigt grafikkort: exponering, portalernas styrka `2.2*d*k` och env-styrkan `.1+1.1*d`.
- **Fas 4 – nya funktioner** (förslag): parkettriktning per rum, flera val samtidigt och gruppflytt, egna möbler och mått, bättre möbelmodeller, delningslänk eller skrivskyddad vy.
- **Fas 5 – teknik:** dela upp koden i moduler (granska strict mode först), rendering på begäran i stället för varje bildruta, bygg- och testpipeline.
