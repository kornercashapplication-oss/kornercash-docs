# SPEC — IA photos & descriptions produits (dashboard KornerCash)

> ✅ **FEATURE EN PROD depuis 2026-07-02, VALIDÉE en conditions réelles** (189 générations bijoux le 2026-07-03, 0 échec) — codée, mergée sur `main`, déployée sur `kornercash-dashboard.vercel.app`.
> Ce document reste la **spec de référence** ; les sections « à coder / à faire » ont été mises à jour en « réalisé » et les détails techniques alignés sur le code réel (`dashboard/lib/ai/`, `dashboard/lib/actions/ia.ts`, `dashboard/components/produits/`).
>
> **Écarts notables vs. design initial (tous en prod) :**
> - Fichiers du module IA : `builders.ts` (logique pure testable) + `config.ts` (ids modèles) + `fal.ts` (server-only) + `template.ts` (asset embarqué) — **pas** de `prompts.ts`.
> - Server actions dans un fichier dédié `lib/actions/ia.ts` (pas dans `produits.ts`).
> - **Multi-photos de référence** : jusqu'à **6** photos du même produit → **1** image générée (le design initial parlait d'« une photo à la fois »). Gérées par `RefUploader` (composant **partagé avec l'atelier**, `components/atelier/ref-uploader.tsx`, `MAX_REFS = 6`).
> - Photo générée : **rotation 90°** (sharp, côté serveur — avant validation via `validerPhotoIA`, et après via `pivoterPhotoProduit`), **zoom** plein écran, **réordonnancement** des images validées (le badge « Couverture » a été retiré — la 1ère image reste la couverture, sans badge).
> - Pas de `temperature` sur l'image (le modèle `nano-banana-2/edit` ne l'expose pas) ; température **0.2** sur la description.
>
> **MISE À JOUR V3 (2026-07-04) — écarts supplémentaires vs. ce qui suit :**
> - **Résolution `2K`** (plus 4K) + **`num_images: 1`**, figés en dur dans `builders.ts` : la résolution est le seul levier de coût du modèle (fal : 0.5K=0.75× · 1K=$0.08 · 2K=$0.12/1.5× · 4K=$0.16/2× — le « Quantity 2.00 » du 4K est un multiplicateur de prix, pas 2 images). 2K = meilleur rapport netteté/prix pour l'e-commerce (÷1.33 vs 4K).
> - **Flux image asynchrone** : file fal (`queue.submit` → `request_id`, polling client 4 s) — **plusieurs générations en parallèle**, la zone de photos est libérée dès le lancement. `fal.subscribe` (timeout 45 s) ne sert plus qu'à la description. Les images n'ont **pas de timeout**.
> - **Actions renommées** : `lancerPhotoIA` / `verifierPhotosIA` / `annulerPhotoIA` remplacent `traiterPhotoIA` ; côté `fal.ts`, `soumettrePhotoProduit`/`statutPhotoProduit`/`resultatPhotoProduit`/`annulerPhotoProduit` remplacent `genererPhotoProduit`. `validerPhotoIA` et `genererDescriptionProduit` inchangées ; nouvelle action `pivoterPhotoProduit` (rotation d'une image déjà stockée, écrasement en place + cache-bust).
> - **Écran résultat = 4 choix** : ✅ Garder · 🔄 Régénérer · 📷 Reprendre les photos (refs renvoyées dans la zone) · ✖ Ignorer (annule le job fal en file, best-effort). Jobs en **état local du formulaire** (pas de table de suivi, contrairement à l'atelier) → avertissement UI : générations non gardées perdues à la fermeture (crédit fal consommé quand même).
> - **Caméra live** : `CameraCapture` (getUserMedia, `components/shared/camera-capture.tsx`) via `RefUploader` — l'`<input capture>` seul est ignoré sur desktop (fallback conservé).
> - **Erreurs en valeur** : `uploadRefPhoto` et `lancerPhotoIA` renvoient `{ok:true,…}|{ok:false,error}` (un throw est masqué par un digest en prod) ; `verifierPhotosIA` renvoie `statut:"echec"+erreur` par job. `validerPhotoIA`/`pivoterPhotoProduit`/`genererDescriptionProduit` throwent encore. Helper `lib/client-errors.ts` (`messageErreur`, `estActionPerimee` → « recharge la page » si bundle périmé après redeploy).
> - **Piège Vercel** : un Buffer sharp passé brut à supabase-js est ré-encodé UTF-8 → JPEG corrompu. Toujours uploader via `new Blob([new Uint8Array(bytes)])` (helper `uploadJpeg`, `lib/actions/ia.ts`).

> Design validé le 2026-07-01 (protocole brainstorming). Feature du **dashboard** (app Next.js interne), pas du site.
> Contexte projet → `contexte kornerCash.md` · Road map → `Road map kornerCash.md` · Code → `dashboard/`

---

## 1. Objectif

Quand un employé crée (ou édite) un produit dans le dashboard, deux tâches sont automatisées par IA :

1. **Photos principales** — chaque photo prise/importée passe par une IA qui la met sur **fond blanc**, en **haute qualité (2K — voir MISE À JOUR V3)**, en restant **strictement fidèle à l'objet réel** (défauts, usures et rayures conservés). But : un catalogue 100 % uniforme, sans reprise manuelle.
2. **Description** — une IA rédige la **description du produit** à partir des champs saisis + des photos, selon un **gabarit fixe** toujours identique, incluant tous les défauts dans un ordre déterminé.

Les **photos de défauts** restent saisies **à la main** (1 ligne + 1 photo par défaut), comme aujourd'hui — elles ne passent **pas** par l'IA.

---

## 2. Décisions validées

| # | Sujet | Décision |
|---|---|---|
| 1 | Nature transfo photo | Fidèle à l'objet (défauts conservés) **+** amélioration de la *prise de vue* : fond blanc, lumière, netteté, résolution 2K (initialement 4K, abaissée pour le coût — V3). **On corrige la photo, pas l'objet.** |
| 2 | Validation photo | Écran de validation par job, 4 choix depuis la V3 : ✅ Garder · 🔄 Régénérer · 📷 Reprendre les photos · ✖ Ignorer. Pas d'option « garder l'original brut ». |
| 3a | Multi-photos | **Réalisé (élargi)** : jusqu'à **6** photos du **même produit** (angles / détails) sont envoyées ensemble → l'IA en synthétise **1** seule image propre (constante `MAX_REFS = 6`). Le prompt impose de garder l'angle de la 1ère photo produit et d'utiliser les autres uniquement pour compléter les détails. |
| 3b | Original brut | **Non conservé.** Seule la version IA validée est uploadée dans le bucket `produits`. |
| 4 | Données description | **Multimodale** : l'IA lit les champs **et** regarde les photos IA validées (`image_urls`). Bridée par le prompt pour ne rien inventer. |
| 5 | Gabarit description | **Réalisé (prompt marketing e-commerce)** : titre accrocheur → intro bénéfice → corps (caractéristiques, ton adapté à la catégorie : gaming / beauté / luxe) → section **Défauts** honnête → CTA subtil. Puces autorisées, SEO naturel. Longueur **moyenne**. **Pas** de mention « Contrôlé et garanti ». *(Voir prompt système réel dans `builders.ts` → `DESCRIPTION_SYSTEM_PROMPT`.)* |
| 6 | Validation description | Champ **éditable** (textarea) + bouton **Générer / Régénérer**. |
| 7 | Outil | **fal.ai** (une clé). Modèles **configurables** via l'env. Modèle image retenu : **`fal-ai/nano-banana-2/edit`**. |
| 8 | Provider description | **fal.ai** aussi — LLM multimodal **`fal-ai/any-llm/vision`** avec le modèle **`google/gemini-2.5-flash`**. Une seule clé (`FAL_KEY`) pour toute la feature. |
| 9 | Prompt image | Prompt figé (voir §7) avec **template carré** en 1ère image de référence. ⚠️ **`fal-ai/nano-banana-2/edit` n'expose PAS de paramètre `temperature`** (la « 0.1 » ne venait que de l'UI Gemini) → non appliqué à l'image. La fidélité est assurée par le **prompt** (« maintain 100% of details… do not retouch defects ») + la **validation**. Résolution : **`resolution: "2K"`** + `num_images: 1` (V3 — le 4K initial coûtait 1.33× plus cher ; avec `aspect_ratio: "1:1"`, `output_format: "jpeg"`). La température **0.2** s'applique uniquement à la **description** (`any-llm/vision`). |

---

## 3. Architecture (approche « composants dédiés + module IA ») — RÉALISÉE

**Module serveur** `lib/ai/` — seul endroit qui parle à fal.ai :
- `lib/ai/fal.ts` — `import "server-only"`. Expose la **file fal image** : `soumettrePhotoProduit(...)` (`queue.submit` → `request_id`), `statutPhotoProduit(...)`, `resultatPhotoProduit(...)`, `annulerPhotoProduit(...)` (best-effort) + `genererDescription(...)` (`fal.subscribe`, timeout **45 s** — les images n'ont pas de timeout) et `uploadImageToFal(...)`. Gère la config de la clé. *(L'ancien `genererPhotoProduit` synchrone a été supprimé — V3.)*
- `lib/ai/builders.ts` — **logique PURE** (aucun import, aucun réseau → testable sans mock) : prompt image figé (`IMAGE_PROMPT`), prompt système description (`DESCRIPTION_SYSTEM_PROMPT`), assemblage des inputs (`buildImageInput`, `buildDescriptionInput`, `buildDescriptionUserPrompt`) + type `ChampsDescription`. La température 0.2 (description) y est fixée en dur. *(remplace le `prompts.ts` prévu au design.)*
- `lib/ai/config.ts` — ids des modèles lus depuis l'environnement (`imageModel()`, `textModel()`, `textLlm()`) avec valeurs par défaut.
- `lib/ai/template.ts` — asset **template carré** embarqué en data URI (`TEMPLATE_CARRE_DATA_URI`), envoyé à fal en 1ère image de référence. *(remplace le fichier `.png` externe prévu au design.)*
- `lib/ai/builders.test.ts` — tests unitaires (vitest) de la logique pure.

**Server actions** `lib/actions/ia.ts` (fichier dédié, `"use server"`) :
- `uploadRefPhoto(dataUri)` — upload d'UNE photo de référence sur le stockage fal (contourne la limite ~4,5 Mo du corps de requête Vercel). Renvoie `{ok,url}|{ok:false,error}`.
- `lancerPhotoIA(produitUrls[])` — soumet la génération à la **file fal** → `{ok, requestId}` (V3, remplace `traiterPhotoIA`).
- `verifierPhotosIA(requestIds[])` — polling batch : par job, `en_cours` / `pret`+`previewUrl` / `echec`+`erreur`.
- `annulerPhotoIA(requestId)` — annulation best-effort d'un job en file (bouton Ignorer).
- `validerPhotoIA(falUrl, rotation)` — télécharge l'image validée, applique la rotation (multiple de 90°, via **sharp** côté serveur), upload dans le bucket `produits` → URL publique.
- `pivoterPhotoProduit(url, degres)` — rotation d'une image **déjà validée/stockée** (écrasement en place, upsert + cache-bust `?v=`).
- `genererDescriptionProduit(champs, photoUrls[])` — génère / régénère la description.

**Composants clients :**
- `components/produits/photo-ia.tsx` — photos **principales** : ajout de 1..6 réf. via `RefUploader` (caméra live / import) → « Générer » = job en fond + zone libérée (générations **en parallèle**) → polling 4 s → **barre de progression estimée** par job → écran résultat (rotation, zoom, sources) → Garder / Régénérer / Reprendre les photos / Ignorer. Gère aussi le **réordonnancement** des images validées (flèches — la 1ère = couverture, sans badge) et la rotation post-validation.
- `components/produits/description-ia.tsx` — textarea éditable + bouton Générer / Régénérer.

**Fichiers modifiés :**
- `components/produits/produit-form.tsx` — orchestre les deux composants (state `description`, objet `champsIA: ChampsDescription`, câblage `PhotoIA` + `DescriptionIA`).
- `lib/actions/produits.ts` + type `ProduitInput` — champ `description_fr` ajouté et écrit dans `createProduit` **et** `updateProduit`.
- `app/(app)/produits/page.tsx` — `export const maxDuration = 60;` (plafond Vercel Hobby).

**Inchangé :** `PhotoUpload` (conservé tel quel pour les **photos de défauts**), la compression navigateur (`lib/image`), `lib/actions/uploads.ts`.

**Asset :** template de placement (carré 700×700) embarqué **en data URI** dans `lib/ai/template.ts` — pas de fichier binaire externe à charger.

---

## 4. Flux photo principale — RÉALISÉ

```
1. Employé prend / importe 1 à 6 photos du MÊME produit (caméra LIVE getUserMedia ou fichier)
2. Compression navigateur (existant) + preview locale
3. Chaque photo est immédiatement uploadée sur le stockage fal (uploadRefPhoto),
   une par requête (limite ~4,5 Mo du corps Vercel)
4. Clic « Générer » → lancerPhotoIA(urls) → file fal (queue.submit → request_id) :
     [template carré (data URI)] + [1..6 photos produit] + prompt figé
     (aspect_ratio 1:1, output_format jpeg, resolution 2K, num_images 1 ; PAS de temperature)
   → la zone de photos est LIBÉRÉE immédiatement : on enchaîne les prises de vue
     et on lance PLUSIEURS générations en parallèle
5. Polling client (verifierPhotosIA, ~4 s) + barre de progression ESTIMÉE par job ;
   le résultat (URL temporaire fal) arrive au fil de l'eau — PAS encore stocké dans Supabase
6. Écran RÉSULTAT par job :
     [image générée : zoom plein écran + rotation 90°]  + vignettes des sources
     ✅ Garder l'image   🔄 Régénérer   📷 Reprendre les photos   ✖ Ignorer
7a. « Garder »    → validerPhotoIA : téléchargement de l'image fal → rotation sharp
                    éventuelle → upload bucket `produits` → ajoutée à la fiche
7b. « Régénérer » → nouveau job fal sur les mêmes photos de référence
7c. « Reprendre » → les références retournent dans la zone de prise de vue
7d. « Ignorer »   → job abandonné (annulation fal best-effort s'il est encore en file)
```

**Règles :**
- On **n'uploade dans le bucket `produits` que l'image validée** → aucun fichier orphelin, et l'original brut n'est jamais stocké (décision 3b). *(Les photos de référence transitent par le stockage fal, pas par Supabase.)*
- **Asynchrone (V3)** : file fal + polling — pas de timeout sur l'image. Les jobs vivent dans l'**état local du formulaire** (pas de table de suivi, contrairement à l'atelier) : fermer le formulaire perd les générations non gardées (avertissement affiché ; crédit fal consommé quand même). Pour du volume avec reprise → l'atelier.
- **Une génération = un appel payant.** Chaque « Régénérer » ou nouveau lot de photos = un nouvel appel. Assumé.
- La **1ère image validée** = photo de couverture sur le site ; réordonnancement possible (flèches).
- Échec fal (réseau / quota) → erreur lisible sur le job (renvoyée en valeur, pas en throw). La photo n'est pas ajoutée.

---

## 5. Flux description — RÉALISÉ

```
1. Employé remplit : désignation, marque, catégorie/sous-cat, état (grade), défauts
2. Il valide ses photos principales (§4)
3. Il clique « Générer la description » (bloqué si la désignation est vide)
4. Server action genererDescriptionProduit → fal (any-llm/vision, gemini-2.5-flash) :
     - system_prompt = DESCRIPTION_SYSTEM_PROMPT (rédaction e-commerce, décision 5)
     - prompt utilisateur = champs sérialisés (défauts numérotés dans l'ordre)
     - image_urls = photos IA validées
     - temperature = 0.2
5. Le texte s'affiche dans une textarea ÉDITABLE
6. L'employé peut retoucher à la main OU cliquer « Régénérer » (nouvel appel)
7. À l'enregistrement → sauvé dans `produits.description_fr`
```

**Garde-fous réels du prompt description** (`DESCRIPTION_SYSTEM_PROMPT`, règle « FIDÉLITÉ » prioritaire) :
- Ne décrire **que** ce qui est fourni dans les champs ou clairement visible sur les photos.
- **Jamais** inventer de spécification non fournie (capacité, année, matériau, authenticité, dimensions, modèle exact). Dans le doute, ne pas la mentionner.
- Traiter les défauts **honnêtement** dans une section dédiée, sans langage dépréciatif ; si « aucun », ne rien mentionner.
- Ne jamais laisser entendre que l'objet est neuf / sans défaut ; « mieux vaut une description courte que des spécifications inventées ».

**Édition d'un produit existant :** le bouton « Générer » reste disponible (utile pour les ~1000 produits actuels sans description). Rien n'est écrasé sans action de l'employé.

---

## 6. Champ `description_fr` — CÂBLÉ

La colonne existait déjà en DB (`produits.description_fr`, lue par le site) mais n'était jamais remplie. Câblage réalisé :
- `ProduitInput` : `description_fr?: string | null` ✅
- `createProduit` **et** `updateProduit` : écrivent la valeur (`input.description_fr || null`) ✅
- `produit-form.tsx` : state `description` (initialisé depuis `editing?.description_fr`) alimenté par `DescriptionIA` ✅

---

## 7. Configuration, secrets & passation client

**Variables d'environnement** (`.env.local` en dev + Vercel en prod) — avec valeurs par défaut dans `lib/ai/config.ts` :

| Variable | Rôle | Défaut |
|---|---|---|
| `FAL_KEY` | clé du compte fal.ai (**créé au nom du client**) | — (requise) |
| `FAL_IMAGE_MODEL` | id du modèle image | `fal-ai/nano-banana-2/edit` |
| `FAL_TEXT_MODEL` | id de l'endpoint LLM multimodal | `fal-ai/any-llm/vision` |
| `FAL_TEXT_LLM` | modèle LLM passé à l'endpoint | `google/gemini-2.5-flash` |

- **Prompt image**, **prompt système description** et **température** = constantes dans `lib/ai/builders.ts` (versionnées).
- **Clé serveur uniquement** : `fal.ts` est `server-only` ; tous les appels fal partent des **server actions**, jamais du navigateur.
- **Passation** : documenter dans le `HANDOFF` la procédure « créer/obtenir la clé fal → la coller dans Vercel ». Le **client paie son usage** (son volume de produits).

**Prompt image réel en prod** (`IMAGE_PROMPT` dans `builders.ts`) — le template est envoyé en **1ère** image, suivi des 1..6 photos produit :

> The first image is a square template. All the following images show the SAME product, photographed from different angles or showing different details. Create ONE single clean product photo of that product, on a background that must be PURE WHITE (hex #FFFFFF), in 1:1 format, matching the dimensions and placement of the inner area of the square template and completely removing its black outline. Keep the product ABSOLUTELY inside the square. The final image MUST show the product from the SAME angle, orientation and viewpoint as the first product photo (the first image right after the square template). Use the other reference images only to complete details that are not clearly visible in that first photo, WITHOUT changing the viewpoint or the orientation. Maintain 100% of the real details, logos, text, wear, scratches and defects visible in the reference photos — do not retouch, smooth, add, remove or invent any detail or defect. Keep the product entire and faithful to the reference photos. Only remove reflections, glares and shadows from the shooting conditions to make the product look clean and professional. The result must show one single product, and the background must be PURE WHITE, hex #FFFFFF, with no other color or shade.
>
> Paramètres image : `aspect_ratio: "1:1"`, `output_format: "jpeg"`, `resolution: "2K"`, `num_images: 1`. **Pas de `temperature`** (non exposé par le modèle). — La température **0.2** ne concerne que la description. *(Résolution et num_images sont figés en dur dans `builders.ts`, pas pilotés par env.)*

---

## 8. Erreurs, coûts & tests — RÉALISÉ

- **Erreurs** : `uploadRefPhoto` / `lancerPhotoIA` renvoient l'erreur **en valeur** `{ok:false, error}` (un throw est masqué par un digest en prod — V3) ; `verifierPhotosIA` renvoie l'échec par job ; les autres actions throwent → toast clair + « Réessaie ». Helper `lib/client-errors.ts`. L'IA agit **avant** l'enregistrement → jamais de produit à moitié créé.
- **Timeouts** (V3) : texte ~45 s, gardé **sous** `maxDuration = 60` (plafond Vercel Hobby, posé sur `app/(app)/produits/page.tsx` ET `app/(app)/atelier/page.tsx`). Les images passent par la file fal → **pas de timeout**.
- **Barre de progression estimée** (V2) : le modèle ne renvoyant pas de %, la barre approche 92 % de façon asymptotique puis 100 % à la fin — une barre par job.
- **Coûts maîtrisés** : aucune génération automatique en boucle — chaque appel résulte d'une action explicite (générer / régénérer). Rien au chargement de page.
- **Tests** : `lib/ai/builders.test.ts` (vitest) sur la logique pure (assemblage prompt + champs + défauts numérotés), sans brûler de crédits fal. Rendu réel jugé **manuellement** (vraies générations).
- ⚠️ **Build** : ce projet utilise une version **Next.js particulière** (cf. `dashboard/AGENTS.md`) → relire la doc locale (`node_modules/next/dist/docs/`) avant de coder.

---

## 9. Hors périmètre (YAGNI)

- ~~Génération asynchrone / file d'attente~~ → **finalement implémentée** (V3 + atelier) : la file fal produit une preview à valider, l'écran de validation est conservé.
- Traitement IA des **photos de défauts** (restent manuelles).
- Conservation de l'original brut (décidé : non).
- ~~Traitement en masse rétroactif des 1000 produits existants~~ → couvert par l'**Atelier de saisie en lot** (`/atelier`, brouillons + jobs asynchrones — utilisé pour les ~180 bijoux).
- Mention réassurance en fin de description.
- Multimodalité pour choisir un modèle « meilleur » : l'archi le permet (config), mais le choix se fait par test, pas dans le code.

---

## 10. Pré-requis Anatole — FOURNIS

1. Template de placement (carré 700×700) → embarqué en data URI dans `lib/ai/template.ts`. ✅
2. **Compte fal.ai** + **clé API** (`FAL_KEY`, dans Vercel). ✅
3. Modèle image : `fal-ai/nano-banana-2/edit` — modifiable via l'env. ✅

---

## 11. Vérifications techniques — RÉSOLUES

- Endpoint image multi-images : `fal-ai/nano-banana-2/edit`, entrée par **URLs** (`image_urls`, les photos étant pré-uploadées sur le stockage fal via `fal.storage.upload`). ✅
- LLM multimodal (description) : `fal-ai/any-llm/vision` + `google/gemini-2.5-flash`, images passées par **URLs publiques** (`image_urls`). ✅
- Contournement limite Vercel ~4,5 Mo : upload d'**une** photo de référence par requête sur le stockage fal avant génération. ✅
- Coût réel au clic (image) : **connu** — la résolution est le seul levier de prix du modèle (fal : 1K=$0.08 · 2K=$0.12 · 4K=$0.16 par génération), d'où le choix 2K. Reste le coût texte (Gemini Flash, négligeable) à affiner pour la passation. ✅
