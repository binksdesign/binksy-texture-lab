# TEXTURE LAB — MODE D’EMPLOI POUR AGENTS IA

Ce document s’adresse aux agents IA (Claude Code, Codex, ou tout agent qui pilote un navigateur via Playwright / CDP). Il explique comment utiliser **Texture Lab** (`app.html`) : importer une image ou une vidéo, empiler et régler des effets (dither, halftone, VHS, glitch, gradient maps…), animer des réglages et exporter en PNG, JPEG ou vidéo.

Il existe en deux endroits, toujours identiques : `AGENTS.md` (Markdown, français puis anglais) et la page `agent/index.html`. Résumé court : `llms.txt`.

Points d’entrée découvrables sans JavaScript : `index.html` et `docs.html` déclarent `<link rel="alternate" type="text/markdown" href="AGENTS.md">`, `<link rel="help" href="agent/">` et `<link rel="alternate" type="text/plain" href="llms.txt">`, et contiennent un commentaire HTML et un `<noscript>` qui renvoient ici. `robots.txt` autorise tout et cite `llms.txt` et `AGENTS.md`.

---

## 0. API pour agents : `window.BinksyAPI`, WebMCP et page MCP

L’app expose une **API JavaScript publique et versionnée** (`apiVersion: "1"`) : `window.BinksyAPI`, présente dans `app.html` dès le chargement. C’est la façon la plus fiable de piloter l’app : chaque méthode appelle la même logique que l’interface (l’écran se met à jour, l’enregistrement automatique suit), toutes sont asynchrones, renvoient du JSON (pas de nœuds DOM ni de données binaires lourdes, sauf les fichiers exportés en base64) et rejettent avec `{ code, message }` en cas d’erreur. Les **mêmes outils** sont enregistrés en **WebMCP** quand le navigateur le permet (`document.modelContext.registerTool()`, ou `navigator.modelContext` sur les anciennes préversions) ; sans WebMCP, rien ne change. Page dédiée : [`mcp/`](mcp/) (agent dans le navigateur, Claude Code / Codex / Hermes / OpenCode via Playwright MCP, prompt prêt à copier).

| Outil WebMCP | Méthode `BinksyAPI` | Rôle |
| --- | --- | --- |
| `texturelab_describe` | `describe()` | Décrit Texture Lab (pile d'effets de texture sur une image ou une vidéo, export PNG/JPEG/vidéo) et liste ses outils avec leurs schémas JSON. |
| `texturelab_get_state` | `getState()` | État courant : média chargé (nom, type, taille ; le média n'est jamais enregistré), pile d'effets (du bas vers le haut, avec uid et réglages), sortie et taille exacte de l'export. |
| `texturelab_importer` | `importer()` | Importe l'image (PNG, JPEG, WebP, GIF, AVIF, SVG…) ou la vidéo (MP4, WebM, MOV) de l'utilisateur, ou un projet texture-lab-projet.json, comme un glisser-déposer. |
| `texturelab_exporter` | `exporter()` | Exporte en pleine résolution sans téléchargement forcé : png, jpg, video (MP4 WebCodecs, sinon WebM) ou project (texture-lab-projet.json). |
| `texturelab_lister_effets` | `listerEffets()` | Liste les 80 effets (id, nom, famille) ; avec details: true, leurs réglages (clé, libellé, type, min, max, défaut, options). |
| `texturelab_lister_presets` | `listerPresets()` | Liste les 20 préréglages intégrés et ceux enregistrés par l’utilisateur, avec leurs effets. |
| `texturelab_appliquer_preset` | `appliquerPreset()` | Applique un préréglage par son nom (ex. « VHS 1994 », « Game Boy ») : il REMPLACE la pile. |
| `texturelab_ajouter_effet` | `ajouterEffet()` | Ajoute un effet en haut de la pile (appliqué en dernier), avec des réglages optionnels. |
| `texturelab_regler_effet` | `reglerEffet()` | Règle une couche de la pile : réglages, visibilité, opacité (0–1), mode de fusion (Normal, Produit, Écran, Incrustation, Lumière tamisée, Addition, Différence, Couleur, Luminosité), ou réinitialisation. |
| `texturelab_retirer_effet` | `retirerEffet()` | Retire une couche de la pile. |
| `texturelab_deplacer_effet` | `deplacerEffet()` | Déplace une couche dans la pile : position 0 = en bas (appliquée en premier). |
| `texturelab_animer_effet` | `animerEffet()` | Anime un réglage numérique d'une couche en boucle parfaite (ou l'opacité avec param: "_mix"), ou coupe son animation. |
| `texturelab_regler_sortie` | `reglerSortie()` | Règle la sortie : cadrage (orig, 1:1, 4:5, 9:16, 16:9), résolution (orig, 2160, 1440, 1080, 720 = petit côté), format (png, jpg, video), qualité JPEG 0,5–1, durée vidéo (s), images/s (24, 25, 30, 60), source vidéo (inT, outT, loopSrc, audio), aperçu (fast, full). |
| `texturelab_annuler` | `annuler()` | Annule la dernière modification de la pile ou de la sortie (redo: true pour rétablir). |
| `texturelab_reset` | `reset()` | Vide la pile d'effets (comme « Tout retirer »). |

Fichiers en entrée : `importer({ files: [{ name, type, dataBase64 | text | url (data: seulement) }] })`, comme un glisser-déposer. En sortie : `exporter({ format, … })` → `{ files: [{ name, type, dataBase64 }] }`, **sans téléchargement** sauf `download: true`. Les actions destructives (`reset`) exigent `confirm: true`, après accord de l’utilisateur.

```js
// Découvrir l’app et la liste des effets
await BinksyAPI.ready();
const info = await BinksyAPI.describe();
const { effects } = await BinksyAPI.listerEffets({ cat: 'trame', details: true });   // id, nom, réglages
const { presets } = await BinksyAPI.listerPresets();   // VHS 1994, Game Boy, Riso zine…
```

```js
// Importer une image en base64 et construire la pile
await BinksyAPI.importer({ files: [{ name: 'photo.jpg', type: 'image/jpeg', dataBase64: b64 }] });
await BinksyAPI.appliquerPreset({ name: 'VHS 1994' });            // remplace la pile
const { layer } = await BinksyAPI.ajouterEffet({ id: 'grain', params: { amount: 0.3 } });
await BinksyAPI.reglerEffet({ uid: layer.uid, opacity: 0.8, blend: 'Écran' });
```

```js
// Exporter (rien n’est téléchargé sauf download: true)
await BinksyAPI.reglerSortie({ aspect: '4:5', res: '1080' });
const { files } = await BinksyAPI.exporter({ format: 'jpg' });   // png, jpg, video ou project
files[0].name;   // photo-texturelab.jpg  (+ width, height, dataBase64)
```

Importe toujours le média de l’utilisateur **avant** d’ajouter des effets : sans média, l’API refuse (`NO_SOURCE`) au lieu de charger l’image d’exemple. Le média n’est pas enregistré : après rechargement, la pile revient, réimporte le fichier. Avec Playwright : `page.evaluate("BinksyAPI.describe()")`, ou `browser_evaluate` dans Playwright MCP (`npx @playwright/mcp@latest`). Les sélecteurs de l’interface (plus bas) restent valables.

---

## 1. Règle fondamentale

Texture Lab applique des effets au fichier que l’utilisateur fournit. Ce n’est **pas** un générateur d’images.

* **Travaille uniquement sur les fichiers que l’utilisateur te donne.** L’image d’exemple intégrée (bouton « Image d’exemple ») sert à tester ; ne la livre jamais à la place du fichier de l’utilisateur.
* **N’ajoute aucun texte ni visuel de marque** sans accord. L’effet « Incrustation VHS » affiche du texte (par défaut `PLAY ▶` et `OCT. 23 1994`) : ne le remplace pas par un nom, un slogan ou une date de ton invention.
* Ne remplace pas un fichier manquant ou illisible par un fichier « approchant ». Dis ce qui manque.
* **Aucune clé API, aucun service en ligne.** Texture Lab n’embarque aucune IA et n’en appelle aucune. Ne demande jamais de clé à l’utilisateur.
* **Tout est local** : lecture, calcul (WebGL2) et encodage se font dans le navigateur. Rien n’est envoyé à un serveur ; seules les polices de l’interface sont chargées en ligne (Google Fonts, Fontshare).
* **API pour agents : `window.BinksyAPI`** (et outils WebMCP, § 0). C’est la voie conseillée : mêmes actions que l’interface, résultats en JSON, aucun clic à simuler. L’interface reste pilotable par ses sélecteurs (§ 4). L’objet `window.TL` reste interne (§ 6).

Après chaque action, vérifie l’état (texte affiché, `#stack`, fichier téléchargé). **Un clic n’est pas une preuve de réussite.**

---

## 2. Démarrer

### Servir l’app

Le dépôt est un site statique : `index.html` (accueil), `app.html` (l’application, un seul fichier autonome), `docs.html` (documentation), `agent/index.html` (cette page), `AGENTS.md`, `llms.txt`, `robots.txt`, `assets/` (styles, polices et images des pages).

```bash
python3 -m http.server 8000
# puis ouvrir http://127.0.0.1:8000/app.html
```

* `app.html?demo` charge directement l’image d’exemple.
* `file://` marche aussi (l’app n’a besoin d’aucune requête) ; seul le manifeste « écran d’accueil » est omis.
* En ligne : `https://texture.binksdesign.com/app.html`.

### Navigateur

* **WebGL2 est obligatoire.** Sans lui, l’app remplace la page par un message (« Ton navigateur ne gère pas WebGL2… »).
* En Chromium sans GPU (headless), lance avec `--use-angle=swiftshader --enable-unsafe-swiftshader --ignore-gpu-blocklist`.
* Le rendu logiciel est lent : la boucle d’aperçu peut empêcher Playwright de juger un élément « stable ». Mets la lecture en pause (bouton `#playBtn` ou touche Espace) avant d’enchaîner les clics, ou déclenche le clic par `element.click()` dans la page (`locator.dispatch_event('click')`).
* Utilise un profil dédié aux tests : la session (pile d’effets et sortie) est mémorisée pour l’origine (§ 7).
* Viewport ≥ 900 px de large pour la mise en page bureau décrite ici. En dessous, l’app passe en mise en page mobile (§ 9).

---

## 3. Parcours type

1. Ouvrir `app.html`.
2. Importer : `set_input_files('#file', chemin)`. L’import est **asynchrone** : attends que `#srcName` affiche le nom du fichier (ou que `window.TL.ready` soit `true`).
3. Construire la pile : cliquer un préréglage `#presets .preset` et/ou des effets `#lib .tile[data-id="…"]`.
4. Régler l’effet sélectionné dans `#fxPanel` (onglet `.tab[data-tab="fx"]`).
5. Choisir la sortie dans `#outPanel` (onglet `.tab[data-tab="out"]`).
6. Exporter : cliquer `#exportBtn` et attendre l’événement de téléchargement.
7. Vérifier le fichier (§ 10).

```python
page.goto("http://127.0.0.1:8000/app.html")
page.set_input_files("#file", "photo.jpg")
page.wait_for_selector("#srcName:has-text('photo.jpg')")
page.click("#playBtn")                                   # pause : clics plus fiables
page.click('#presets .preset:has-text("VHS 1994")')
page.click('.tab[data-tab="out"]')
page.click('#outPanel .seg button:has-text("JPEG")')
with page.expect_download(timeout=120_000) as dl:
    page.click("#exportBtn")
dl.value.save_as("photo-texturelab.jpg")
```

---

## 4. Repères de l’interface (sélecteurs réels)

### Fixes

| Sélecteur | Rôle |
| --- | --- |
| `#file` | champ fichier caché (image/\*, video/\*) — à remplir avec `set_input_files` |
| `#drop` | zone d’import (clic ou glisser-déposer) ; `#srcName` = nom du fichier chargé |
| `#demoBtn`, `#emptyDemo` | charger l’image d’exemple (`#emptyDemo` applique aussi le préréglage Binks si la pile est vide) |
| `#emptyImport` | ouvrir le sélecteur de fichier (écran vide) |
| `#presets` | préréglages ; `.more-presets` déplie la liste complète |
| `#savePreset` | enregistrer la pile comme préréglage (ouvre un `prompt()` : gère la boîte de dialogue) |
| `#stack` | pile d’effets ; `#addFx` ouvre la bibliothèque |
| `#libSearch`, `#cats`, `#lib` | recherche, filtres de famille, bibliothèque |
| `.tab[data-tab="lib"\|"fx"\|"out"]` | onglets Effets / Réglages / Sortie |
| `#fxPanel` | réglages de l’effet sélectionné |
| `#outPanel` | réglages de sortie |
| `#compareBtn`, `#surpriseBtn`, `#clearBtn` | Comparer, Surprends-moi, Tout retirer |
| `#undoBtn`, `#redoBtn` | annuler, rétablir |
| `#exportBtn` | exporter (bureau : export direct ; mobile : ouvre un récapitulatif) |
| `#modal`, `#mText`, `#mCancel` | fenêtre de progression de l’export, son texte, Annuler |
| `#toast` | message bref (succès ou erreur) |
| `#saveProj`, `#openProj`, `#projFile` | enregistrer / ouvrir un projet `.json` (`#projFile` = champ fichier caché) |
| `#playBtn`, `#tRange`, `#muteBtn` | lecture / pause, position, son (vidéo) |
| `#view` | le canvas d’aperçu |

### Générés

| Sélecteur | Rôle |
| --- | --- |
| `#presets .preset` | un préréglage ; texte = son nom. Préréglage perso : contient `.x` (supprimer) |
| `#lib .tile[data-id="<id>"]` | un effet de la bibliothèque (ajout au clic) ; classe `added` s’il est déjà dans la pile |
| `#cats .chip[data-cat="<famille>"]` | filtre (`all`, `couleur`, `dither`, `trame`, `pixel`, `shader`, `grain`, `analog`, `glitch`, `distort`, `style`) |
| `#stack li.layer[data-uid]` | une couche (la première de la liste = le haut de la pile, appliqué en dernier) ; `.l-name`, `.l-eye` (masquer / afficher), `.l-del` (retirer) ; classes `sel`, `off` |
| `#fxPanel .fx-head h2` | nom de l’effet sélectionné |
| `#fxPanel [data-a="anim"\|"rand"\|"reset"\|"dup"]` | animer l’effet, au hasard, réinitialiser, dupliquer |
| `#fxPanel .ctl` | un réglage : libellé dans `.ctl-l span`, puis `input.rg` (curseur), `select`, `input[type=color]`, `.switch[role=switch]` ou `input.txt` |
| `.anim-btn` | bouton d’animation d’un réglage ; ouvre `.anim-box` (valeur d’arrivée, mouvement, vitesse) |
| `#outPanel .seg button` | choix segmentés : `Original`, `1:1`, `4:5`, `9:16`, `16:9` ; `PNG`, `JPEG`, `Vidéo` ; `24`, `25`, `30`, `60` ; `Fluide`, `Net` |
| `#outPanel .help` | la première indique la taille exacte : « Sortie en L × H px » |

### Changer une valeur

Les curseurs réagissent aux événements `input` puis `change` (le `change` enregistre l’étape d’historique et la session) :

```js
const ctl = [...document.querySelectorAll('#fxPanel .ctl')]
  .find(c => c.querySelector('.ctl-l span')?.textContent === 'Taille de la trame');
const r = ctl.querySelector('input.rg');
r.value = 14;
r.dispatchEvent(new Event('input'));
r.dispatchEvent(new Event('change'));
```

Listes : `select.value = index; select.dispatchEvent(new Event('change'))` (la valeur est l’**index** de l’option). Couleurs : `input[type=color]` + `input` puis `change`. Double-clic sur un curseur = valeur par défaut.

---

## 5. Effets, préréglages, sortie

* **80 effets** en 10 familles : `couleur` (11), `dither` (5), `trame` (13), `pixel` (8), `shader` (6), `grain` (9), `analog` (6), `glitch` (9), `distort` (7), `style` (6). Identifiants, noms et réglages : `window.TL.DEFS` (§ 6) ou la [documentation](docs.html#bibliotheque).
* Effets calculés par le processeur (plus lents) : `errdiff`, `pixelsort`, `jpeg`. Effets à rétroaction : `datamosh`, `echo`.
* **20 préréglages** : VHS 1994, Game Boy, Riso zine, Journal, Glitch club, Cyber CRT, Film 35 mm, Gravure, Verre cannelé, Terminal, Thermique, 1-bit Atkinson, Pixel sort, Photocopie, CMJN offset, Datamosh, PICO-8, Hologramme, Vitrail, Binks. Un préréglage **remplace** la pile.
* Ajouter un effet ou un préréglage sans média chargé charge l’image d’exemple : importe le fichier de l’utilisateur **avant**.
* **Fusion** par couche : opacité 0–100 % et 9 modes (Normal, Produit, Écran, Incrustation, Lumière tamisée, Addition, Différence, Couleur, Luminosité).
* **Animation** : chaque réglage numérique et l’opacité ; valeur d’arrivée, mouvement (Va-et-vient, Montée en boucle, Pulsation, Dérive organique, Saccadé), vitesse en cycles par boucle (arrondie pour boucler sur la durée).
* **Sortie** : cadrage `Original` / `1:1` / `4:5` / `9:16` / `16:9` (recadrage centré) ; résolution Originale, 4K, 1440p, 1080p, 720p (petit côté ; 8192 px max ou limite GPU) ; PNG, JPEG (qualité 50–100 %, 92 % par défaut) ou Vidéo (0,5–60 s pour une image ; 24 / 25 / 30 / 60 images/s ; pour une vidéo source : début, fin, boucler la source, garder le son). Aperçu `Fluide` (1500 px) ou `Net` (2600 px) ; **l’export est toujours en pleine résolution**.

### Fichiers produits

| Cas | Nom du fichier |
| --- | --- |
| PNG / JPEG | `<nom de la source sans extension>-texturelab.png` / `.jpg` |
| Vidéo (WebCodecs) | `<nom>-texturelab.mp4` — H.264, sinon VP9 ou AV1 ; son AAC ou Opus si « Garder le son » |
| Vidéo (repli temps réel MediaRecorder) | `<nom>-texturelab.mp4` ou `.webm`, 1080p max, sans son |
| Projet | `texture-lab-projet.json` |

Avec l’image d’exemple, le nom de la source est « Image d’exemple ». Les fichiers sont remis par un lien de téléchargement : écoute l’événement *download* de ton navigateur piloté (dossier de téléchargements). Avec une vidéo en entrée, PNG / JPEG exportent l’image affichée à l’instant courant.

---

## 6. L’objet interne `window.TL`

`app.html` expose `window.TL` pour ses propres tests. **Ce n’est pas l’API publique** (c’est `window.BinksyAPI`, § 0) : rien n’est versionné ni garanti. Utilise-le surtout en lecture.

```js
TL.ready            // true quand une image ou une vidéo est chargée
TL.st.stack         // [{uid, id, on, mix, blend, v:{…}, anim:{…}}] — du bas vers le haut
TL.st.out           // {aspect, res, fmt, dur, fps, inT, outT, loopSrc, audio, quality, jpgQ}
TL.DEFS             // définitions des 80 effets : {id: {name, cat, params:[{k, label, type, min, max, def, options}]}}
TL.outDims(true)    // {w, h, crop} : taille exacte de l’export
TL.addFx('halftone')   // ajoute un effet en haut de la pile (historique et session compris)
TL.loadDemo()          // charge l’image d’exemple
TL.setPlaying(false)   // pause
TL.runExport()         // lance l’export avec la sortie courante (comme #exportBtn au bureau)
```

Ne modifie pas `TL.st` directement : l’historique et la session ne seraient pas mis à jour. Pour imposer des valeurs exactes, passe par l’interface (§ 4) ou par un fichier projet (§ 7).

---

## 7. État persistant et projets

| Où | Clé / fichier | Contenu |
| --- | --- | --- |
| `localStorage` | `texturelab.session` | `{stack, out}` — réécrit à chaque modification, relu au chargement (une sortie `video` repasse en `png`) |
| `localStorage` | `texturelab.presets` | `[{name, stack}]` — préréglages perso |
| fichier | `texture-lab-projet.json` | `{app: "texturelab", v: 1, stack, out}` |

* **Le média n’est jamais enregistré** : après un rechargement, la pile revient mais `TL.ready` est `false` ; réimporte le fichier.
* L’historique (annuler / rétablir, 120 étapes) n’est pas conservé.
* `#projFile` (ou un `.json` glissé sur la page) ouvre un projet : effets inconnus ignorés, image d’exemple chargée si aucun média. Chaque couche du projet doit contenir **tous** ses réglages dans `v` (les valeurs manquantes ne sont pas complétées) : pars d’un projet enregistré par l’app, puis modifie ses valeurs.
* Pour repartir de zéro : `#clearBtn`, ou `localStorage.removeItem('texturelab.session')` puis rechargement.

---

## 8. Clavier

`Ctrl`/`⌘` + `Z` annuler ; `Ctrl`/`⌘` + `Maj` + `Z` ou `Ctrl`/`⌘` + `Y` rétablir ; `Espace` lecture / pause ; `C` maintenu = original ; `Suppr` / `Retour arrière` = retirer la couche sélectionnée ; `Échap` ferme panneau ou dialogue (mobile). Les touches simples sont ignorées pendant la saisie dans un champ.

---

## 9. Mise en page mobile

Sous 900 px de large (ou téléphone en paysage), les panneaux sont déplacés dans des feuilles : barre `#mTabs` avec `button[data-sheet="lib"|"stack"|"fx"|"out"|"media"]`, feuille `#sheet`. Les mêmes éléments (`#lib`, `#stack`, `#fxPanel`, `#outPanel`, `#drop`…) y sont déplacés, sans doublon. `#exportBtn` ouvre un récapitulatif dont le bouton `#xGo` lance l’export ; si le navigateur sait partager des fichiers, une boîte « C’est prêt » propose « Télécharger » (`#dlBtn`) ou « Enregistrer ou partager » (`#shBtn`). Pour un agent, la mise en page bureau est plus simple.

---

## 10. Vérifications avant de rendre le travail

* Le fichier source est bien celui de l’utilisateur (`#srcName`) et non l’image d’exemple.
* La pile contient ce qui était demandé (`#stack li.layer .l-name`, de haut en bas), couches masquées comprises (`.off`).
* Le fichier téléchargé existe, son extension correspond au format, ses dimensions correspondent à « Sortie en L × H px ».
* Vidéo : durée ≈ durée réglée ; piste son présente seulement si demandée et disponible.
* Regarde le résultat (capture de `#view` ou ouverture du fichier) : effet visible, pas d’image noire ou vide.
* Aucun toast d’erreur, aucune erreur dans la console.

---

## 11. Limites et erreurs

| Message (`#toast` ou page) | Cause |
| --- | --- |
| « Ton navigateur ne gère pas WebGL2… » | pas de WebGL2 : changer de navigateur ou de drapeaux |
| « Impossible d’ouvrir cette image. » | image illisible par le navigateur |
| « Cette vidéo ne peut pas être lue par le navigateur. » | codec non pris en charge : convertir (MP4 H.264) |
| « Format non reconnu. » | ni image, ni vidéo, ni `.json` |
| « Importe d’abord une image ou une vidéo. » | export sans média |
| « Ton navigateur ne sait pas encoder de vidéo : exporte une image. » | ni WebCodecs ni MediaRecorder |
| « L’export a échoué : … » / « L’export vidéo a échoué : … » | erreur d’export (souvent mémoire ou taille) : réduire la résolution ou la durée |
| « Ce fichier n’est pas un projet Texture Lab. » | JSON sans `stack` |
| « Ajoute d’abord des effets. » | enregistrer un préréglage avec une pile vide |

* Images importées réduites au-delà de 8192 px (3072 px en mobile / tablette).
* Chromium headless n’a souvent pas H.264 : la vidéo sort alors en VP9 dans un MP4. Le rendu logiciel rend l’export vidéo lent : commence par une durée courte.
* Le MP4 WebCodecs est assemblé en mémoire : longues vidéos en 4K = lourd.

---

## 12. Logique à suivre pour chaque tâche

1. Clarifier : quel fichier, quel rendu (préréglage, effets, réglages), quel format, quelle taille, quelle durée.
2. Importer le fichier de l’utilisateur et attendre `#srcName`.
3. Construire la pile, régler, animer si demandé ; mettre en pause pour vérifier l’aperçu.
4. Régler la sortie, lire « Sortie en L × H px ».
5. Exporter, récupérer le téléchargement, vérifier (§ 10).
6. Rendre le fichier, et proposer `texture-lab-projet.json` pour refaire ou ajuster plus tard.

---
---

# TEXTURE LAB — GUIDE FOR AI AGENTS

This document is for AI agents (Claude Code, Codex, or any agent driving a browser through Playwright / CDP). It explains how to use **Texture Lab** (`app.html`): import an image or a video, stack and tune effects (dither, halftone, VHS, glitch, gradient maps…), animate settings and export to PNG, JPEG or video.

It lives in two places, always identical: `AGENTS.md` (Markdown, French then English) and the page `agent/index.html`. Short summary: `llms.txt`.

Entry points discoverable without JavaScript: `index.html` and `docs.html` declare `<link rel="alternate" type="text/markdown" href="AGENTS.md">`, `<link rel="help" href="agent/">` and `<link rel="alternate" type="text/plain" href="llms.txt">`, and contain an HTML comment and a `<noscript>` pointing here. `robots.txt` allows everything and lists `llms.txt` and `AGENTS.md`.

---

## 0. Agent API: `window.BinksyAPI`, WebMCP and MCP page

The app exposes a **public, versioned JavaScript API** (`apiVersion: "1"`): `window.BinksyAPI`, present in `app.html` from load. It is the most reliable way to drive the app: every method runs the same logic as the UI (screen updates, auto-save follows), is async, returns JSON and rejects with `{ code, message }`. The **same tools** are registered as **WebMCP** tools when the browser supports it (`document.modelContext.registerTool()`, or `navigator.modelContext` in older previews). Dedicated page: [`mcp/`](mcp/).

Tools: `texturelab_describe` (`describe()`), `texturelab_get_state` (`getState()`), `texturelab_importer` (`importer()`), `texturelab_exporter` (`exporter()`), `texturelab_lister_effets` (`listerEffets()`), `texturelab_lister_presets` (`listerPresets()`), `texturelab_appliquer_preset` (`appliquerPreset()`), `texturelab_ajouter_effet` (`ajouterEffet()`), `texturelab_regler_effet` (`reglerEffet()`), `texturelab_retirer_effet` (`retirerEffet()`), `texturelab_deplacer_effet` (`deplacerEffet()`), `texturelab_animer_effet` (`animerEffet()`), `texturelab_regler_sortie` (`reglerSortie()`), `texturelab_annuler` (`annuler()`), `texturelab_reset` (`reset()`).

Files in: `importer({ files: [{ name, type, dataBase64 | text | url (data: only) }] })`. Files out: `exporter({ format, … })` → `{ files: [{ name, type, dataBase64 }] }`, **no download** unless `download: true`. Destructive actions (`reset`) need `confirm: true`, after the user agrees.

```js
await BinksyAPI.ready();
const { tools } = await BinksyAPI.describe();   // JSON schemas of every tool
```

---

## 1. Golden rule

Texture Lab applies effects to the file the user provides. It is **not** an image generator.

* **Only work on the files the user gives you.** The built-in example image (“Image d’exemple” button) is for testing; never deliver it instead of the user’s file.
* **Do not add brand text or visuals** without approval. The “Incrustation VHS” effect draws text (default `PLAY ▶` and `OCT. 23 1994`): do not replace it with a name, slogan or date you made up.
* Never replace a missing or unreadable file with a “close enough” one. Say what is missing.
* **No API key, no online service.** Texture Lab ships and calls no AI. Never ask the user for a key.
* **Everything is local**: reading, rendering (WebGL2) and encoding happen in the browser. Nothing is uploaded; only the interface fonts load from the web (Google Fonts, Fontshare).
* **Agent API: `window.BinksyAPI`** (and WebMCP tools, § 0). This is the recommended way: same actions as the UI, JSON results, no clicks to simulate. The UI can still be driven through its selectors (§ 4). The `window.TL` object stays internal (§ 6).

After every action, check the state (displayed text, `#stack`, downloaded file). **A click is not proof of success.**

---

## 2. Getting started

### Serving the app

The repository is a static site: `index.html` (home), `app.html` (the app, one self-contained file), `docs.html` (docs), `agent/index.html` (this page), `AGENTS.md`, `llms.txt`, `robots.txt`, `assets/` (styles, fonts and images of the pages).

```bash
python3 -m http.server 8000
# then open http://127.0.0.1:8000/app.html
```

* `app.html?demo` loads the example image right away.
* `file://` works too (the app needs no request); only the home-screen manifest is skipped.
* Online: `https://texture.binksdesign.com/app.html`.

### Browser

* **WebGL2 is required.** Without it the app replaces the page with a message (“Ton navigateur ne gère pas WebGL2…”).
* In GPU-less (headless) Chromium, launch with `--use-angle=swiftshader --enable-unsafe-swiftshader --ignore-gpu-blocklist`.
* Software rendering is slow: the preview loop can stop Playwright from seeing an element as “stable”. Pause playback (`#playBtn` or the Space key) before chaining clicks, or fire the click inside the page (`element.click()`, `locator.dispatch_event('click')`).
* Use a dedicated test profile: the session (effect stack and output) is stored per origin (§ 7).
* Use a viewport ≥ 900 px wide for the desktop layout described here. Below that the app switches to its mobile layout (§ 9).

---

## 3. Typical flow

1. Open `app.html`.
2. Import: `set_input_files('#file', path)`. Import is **asynchronous**: wait until `#srcName` shows the file name (or `window.TL.ready` is `true`).
3. Build the stack: click a preset `#presets .preset` and/or effects `#lib .tile[data-id="…"]`.
4. Tune the selected effect in `#fxPanel` (tab `.tab[data-tab="fx"]`).
5. Pick the output in `#outPanel` (tab `.tab[data-tab="out"]`).
6. Export: click `#exportBtn` and wait for the download event.
7. Check the file (§ 10).

```python
page.goto("http://127.0.0.1:8000/app.html")
page.set_input_files("#file", "photo.jpg")
page.wait_for_selector("#srcName:has-text('photo.jpg')")
page.click("#playBtn")                                   # pause: steadier clicks
page.click('#presets .preset:has-text("VHS 1994")')
page.click('.tab[data-tab="out"]')
page.click('#outPanel .seg button:has-text("JPEG")')
with page.expect_download(timeout=120_000) as dl:
    page.click("#exportBtn")
dl.value.save_as("photo-texturelab.jpg")
```

---

## 4. Interface hooks (real selectors)

### Static

| Selector | Role |
| --- | --- |
| `#file` | hidden file input (image/\*, video/\*) — fill it with `set_input_files` |
| `#drop` | import area (click or drag and drop); `#srcName` = name of the loaded file |
| `#demoBtn`, `#emptyDemo` | load the example image (`#emptyDemo` also applies the Binks preset when the stack is empty) |
| `#emptyImport` | open the file picker (empty screen) |
| `#presets` | presets; `.more-presets` expands the full list |
| `#savePreset` | save the stack as a preset (opens a `prompt()`: handle the dialog) |
| `#stack` | effect stack; `#addFx` opens the library |
| `#libSearch`, `#cats`, `#lib` | search, family filters, library |
| `.tab[data-tab="lib"\|"fx"\|"out"]` | Effects / Settings / Output tabs |
| `#fxPanel` | settings of the selected effect |
| `#outPanel` | output settings |
| `#compareBtn`, `#surpriseBtn`, `#clearBtn` | Compare, Surprise me, Remove all |
| `#undoBtn`, `#redoBtn` | undo, redo |
| `#exportBtn` | export (desktop: exports directly; mobile: opens a summary) |
| `#modal`, `#mText`, `#mCancel` | export progress dialog, its text, Cancel |
| `#toast` | short message (success or error) |
| `#saveProj`, `#openProj`, `#projFile` | save / open a `.json` project (`#projFile` = hidden file input) |
| `#playBtn`, `#tRange`, `#muteBtn` | play / pause, position, sound (video) |
| `#view` | the preview canvas |

### Generated

| Selector | Role |
| --- | --- |
| `#presets .preset` | a preset; text = its name. Custom preset: contains `.x` (delete) |
| `#lib .tile[data-id="<id>"]` | a library effect (added on click); class `added` when already in the stack |
| `#cats .chip[data-cat="<family>"]` | filter (`all`, `couleur`, `dither`, `trame`, `pixel`, `shader`, `grain`, `analog`, `glitch`, `distort`, `style`) |
| `#stack li.layer[data-uid]` | a layer (first in the list = top of the stack, applied last); `.l-name`, `.l-eye` (hide / show), `.l-del` (remove); classes `sel`, `off` |
| `#fxPanel .fx-head h2` | name of the selected effect |
| `#fxPanel [data-a="anim"\|"rand"\|"reset"\|"dup"]` | animate the effect, randomize, reset, duplicate |
| `#fxPanel .ctl` | one setting: label in `.ctl-l span`, then `input.rg` (slider), `select`, `input[type=color]`, `.switch[role=switch]` or `input.txt` |
| `.anim-btn` | animate one setting; opens `.anim-box` (target value, motion, speed) |
| `#outPanel .seg button` | segmented choices: `Original`, `1:1`, `4:5`, `9:16`, `16:9`; `PNG`, `JPEG`, `Vidéo`; `24`, `25`, `30`, `60`; `Fluide`, `Net` |
| `#outPanel .help` | the first one gives the exact size: “Sortie en W × H px” |

### Changing a value

Sliders listen to `input` then `change` (the `change` records the history step and the session):

```js
const ctl = [...document.querySelectorAll('#fxPanel .ctl')]
  .find(c => c.querySelector('.ctl-l span')?.textContent === 'Taille de la trame');
const r = ctl.querySelector('input.rg');
r.value = 14;
r.dispatchEvent(new Event('input'));
r.dispatchEvent(new Event('change'));
```

Selects: `select.value = index; select.dispatchEvent(new Event('change'))` (the value is the option **index**). Colours: `input[type=color]` + `input` then `change`. Double-click on a slider = default value.

---

## 5. Effects, presets, output

* **80 effects** in 10 families: `couleur` (11), `dither` (5), `trame` (13), `pixel` (8), `shader` (6), `grain` (9), `analog` (6), `glitch` (9), `distort` (7), `style` (6). Ids, names and settings: `window.TL.DEFS` (§ 6) or the [documentation](docs.html#bibliotheque).
* CPU effects (slower): `errdiff`, `pixelsort`, `jpeg`. Feedback effects: `datamosh`, `echo`.
* **20 presets**: VHS 1994, Game Boy, Riso zine, Journal, Glitch club, Cyber CRT, Film 35 mm, Gravure, Verre cannelé, Terminal, Thermique, 1-bit Atkinson, Pixel sort, Photocopie, CMJN offset, Datamosh, PICO-8, Hologramme, Vitrail, Binks. A preset **replaces** the stack.
* Adding an effect or a preset with no media loaded loads the example image: import the user’s file **first**.
* **Blending** per layer: opacity 0–100 % and 9 modes (Normal, Produit, Écran, Incrustation, Lumière tamisée, Addition, Différence, Couleur, Luminosité).
* **Animation**: every numeric setting and the opacity; target value, motion (Va-et-vient, Montée en boucle, Pulsation, Dérive organique, Saccadé), speed in cycles per loop (rounded so it loops on the duration).
* **Output**: framing `Original` / `1:1` / `4:5` / `9:16` / `16:9` (centred crop); resolution Originale, 4K, 1440p, 1080p, 720p (short side; 8192 px max or the GPU limit); PNG, JPEG (quality 50–100 %, 92 % by default) or Video (0.5–60 s for an image; 24 / 25 / 30 / 60 fps; for a video source: start, end, loop the source, keep the sound). Preview `Fluide` (1500 px) or `Net` (2600 px); **export is always full resolution**.

### Output files

| Case | File name |
| --- | --- |
| PNG / JPEG | `<source name without extension>-texturelab.png` / `.jpg` |
| Video (WebCodecs) | `<name>-texturelab.mp4` — H.264, else VP9 or AV1; AAC or Opus sound if “Garder le son” |
| Video (real-time MediaRecorder fallback) | `<name>-texturelab.mp4` or `.webm`, 1080p max, no sound |
| Project | `texture-lab-projet.json` |

With the example image, the source name is “Image d’exemple”. Files are delivered through a download link: listen for your driven browser’s *download* event (downloads folder). With a video source, PNG / JPEG export the frame shown at the current time.

---

## 6. The internal `window.TL` object

`app.html` exposes `window.TL` for its own tests. **It is not the public API** (that is `window.BinksyAPI`, § 0): nothing is versioned or guaranteed. Use it mostly to read.

```js
TL.ready            // true once an image or a video is loaded
TL.st.stack         // [{uid, id, on, mix, blend, v:{…}, anim:{…}}] — bottom to top
TL.st.out           // {aspect, res, fmt, dur, fps, inT, outT, loopSrc, audio, quality, jpgQ}
TL.DEFS             // the 80 effect definitions: {id: {name, cat, params:[{k, label, type, min, max, def, options}]}}
TL.outDims(true)    // {w, h, crop}: exact export size
TL.addFx('halftone')   // adds an effect on top of the stack (history and session included)
TL.loadDemo()          // loads the example image
TL.setPlaying(false)   // pause
TL.runExport()         // runs the export with the current output (like #exportBtn on desktop)
```

Do not modify `TL.st` directly: history and session would not be updated. To force exact values, go through the interface (§ 4) or a project file (§ 7).

---

## 7. Persistent state and projects

| Where | Key / file | Content |
| --- | --- | --- |
| `localStorage` | `texturelab.session` | `{stack, out}` — rewritten on every change, read on load (a `video` output falls back to `png`) |
| `localStorage` | `texturelab.presets` | `[{name, stack}]` — custom presets |
| file | `texture-lab-projet.json` | `{app: "texturelab", v: 1, stack, out}` |

* **The media is never stored**: after a reload the stack comes back but `TL.ready` is `false`; import the file again.
* History (undo / redo, 120 steps) is not kept.
* `#projFile` (or a `.json` dropped on the page) opens a project: unknown effects are skipped, the example image is loaded if no media is. Each layer must hold **all** its settings in `v` (missing values are not filled in): start from a project saved by the app, then edit its values.
* To start fresh: `#clearBtn`, or `localStorage.removeItem('texturelab.session')` then reload.

---

## 8. Keyboard

`Ctrl`/`⌘` + `Z` undo; `Ctrl`/`⌘` + `Shift` + `Z` or `Ctrl`/`⌘` + `Y` redo; `Space` play / pause; hold `C` = original; `Delete` / `Backspace` = remove the selected layer; `Escape` closes a sheet or dialog (mobile). Single keys are ignored while typing in a field.

---

## 9. Mobile layout

Below 900 px wide (or a phone in landscape), panels move into sheets: tab bar `#mTabs` with `button[data-sheet="lib"|"stack"|"fx"|"out"|"media"]`, sheet `#sheet`. The same elements (`#lib`, `#stack`, `#fxPanel`, `#outPanel`, `#drop`…) are moved there, never duplicated. `#exportBtn` opens a summary whose `#xGo` button starts the export; when the browser can share files, a “C’est prêt” dialog offers “Télécharger” (`#dlBtn`) or “Enregistrer ou partager” (`#shBtn`). For an agent, the desktop layout is simpler.

---

## 10. Checks before handing back

* The source file is the user’s (`#srcName`), not the example image.
* The stack holds what was asked (`#stack li.layer .l-name`, top to bottom), hidden layers included (`.off`).
* The downloaded file exists, its extension matches the format, its size matches “Sortie en W × H px”.
* Video: duration ≈ the set duration; sound track only if requested and available.
* Look at the result (screenshot of `#view` or open the file): the effect is visible, no black or empty image.
* No error toast, no error in the console.

---

## 11. Limits and errors

| Message (`#toast` or page) | Cause |
| --- | --- |
| “Ton navigateur ne gère pas WebGL2…” | no WebGL2: change browser or flags |
| “Impossible d’ouvrir cette image.” | image the browser cannot decode |
| “Cette vidéo ne peut pas être lue par le navigateur.” | unsupported codec: convert (MP4 H.264) |
| “Format non reconnu.” | not an image, a video or a `.json` |
| “Importe d’abord une image ou une vidéo.” | export with no media |
| “Ton navigateur ne sait pas encoder de vidéo : exporte une image.” | neither WebCodecs nor MediaRecorder |
| “L’export a échoué : …” / “L’export vidéo a échoué : …” | export error (often memory or size): lower the resolution or duration |
| “Ce fichier n’est pas un projet Texture Lab.” | JSON without `stack` |
| “Ajoute d’abord des effets.” | saving a preset with an empty stack |

* Imported images are scaled down beyond 8192 px (3072 px on mobile / tablet).
* Headless Chromium often lacks H.264: video then comes out as VP9 in an MP4. Software rendering makes video export slow: start with a short duration.
* The WebCodecs MP4 is assembled in memory: long 4K videos are heavy.

---

## 12. Logic for every task

1. Clarify: which file, which look (preset, effects, settings), which format, which size, which duration.
2. Import the user’s file and wait for `#srcName`.
3. Build the stack, tune, animate if asked; pause to check the preview.
4. Set the output, read “Sortie en W × H px”.
5. Export, grab the download, check (§ 10).
6. Hand back the file, and offer `texture-lab-projet.json` to redo or adjust later.
