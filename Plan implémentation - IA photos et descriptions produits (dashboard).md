# IA photos & descriptions produits — Plan d'implémentation

> ## ✅ RÉALISÉ + DÉPLOYÉ EN PROD — 2026-07-02
>
> **Ce plan est entièrement implémenté.** Les 10 tâches ont été codées (mode subagent-driven : implémenteur + reviewer par tâche + revue finale), mergées sur `main`, et la feature est **déployée en production** sur `kornercash-dashboard.vercel.app` le 2026-07-02. Vérifs de sortie : tests vitest verts + `tsc --noEmit` / `npm run lint` / `npm run build` à 0.
>
> **Écarts notables entre le plan initial et le code réellement livré** (le détail des tâches ci-dessous reste tel qu'écrit à l'origine, pour l'historique — se référer à cet encadré pour la vérité terrain) :
> - **Multi-photos de référence** (jusqu'à 6, constante `MAX_REFS`), pas une seule photo. `buildImageInput(template, produitDataUris[])` prend un tableau ; `PhotoIA` gère une liste `refs[]` avec upload immédiat de chaque photo.
> - **Upload sur le stockage fal** (`uploadImageToFal` dans `fal.ts` / `uploadRefPhoto` dans `ia.ts`) pour contourner la limite ~4,5 Mo du corps de requête Vercel — le plan envoyait le base64 inline (voir « Fallback data URI » en fin de plan, finalement adopté par défaut).
> - **Prompt description remplacé** : le gabarit strict « 80-120 mots / À noter : » a été abandonné au profit d'un **prompt marketing e-commerce** (titre accrocheur, adaptation du ton par catégorie, CTA) — le garde-fou anti-invention est conservé. `DESCRIPTION_SYSTEM_PROMPT` dans `builders.ts` fait foi.
> - **V2 UX (post-plan) déployée** : barre de progression **estimée** (le modèle ne renvoie pas de vrai %, approche asymptotique de 92 % puis 100 %) ; bouton **rotation 90°** (sharp côté serveur dans `validerPhotoIA`) ; **zoom** plein écran ; **réordonnancement** des photos validées (flèches) ; bouton **Régénérer**. *(Le badge « Couverture » a ensuite été retiré — la 1ère image reste la couverture, sans badge.)*
> - **V3 (2026-07-04) — flux image asynchrone** : les images passent par la **file fal** (`queue.submit` → `request_id` + polling client 4 s), **plusieurs générations en parallèle**, jobs en état local du formulaire (pas de table de suivi — avertissement UI : générations non gardées perdues à la fermeture) ; boutons **Ignorer** (annulation fal best-effort) et **Reprendre les photos** (refs renvoyées dans la zone). Actions : `lancerPhotoIA`/`verifierPhotosIA`/`annulerPhotoIA` remplacent `traiterPhotoIA` ; côté `fal.ts`, `soumettrePhotoProduit`/`statutPhotoProduit`/`resultatPhotoProduit`/`annulerPhotoProduit` remplacent `genererPhotoProduit`. Nouvelle action `pivoterPhotoProduit` (rotation d'une image **déjà stockée**, écrasement en place + cache-bust). **Timeout : desc 45 s uniquement** (`withTimeout` privé dans `fal.ts`) — plus de timeout image. Les refs sont gérées par `RefUploader` (composant **partagé avec l'atelier**, `components/atelier/ref-uploader.tsx`, `MAX_REFS = 6`) + **caméra live getUserMedia** (`CameraCapture`) — l'`<input capture>` du plan est un simple fallback.
> - **Erreurs en valeur** : `uploadRefPhoto`/`lancerPhotoIA` renvoient `{ok:true,…}|{ok:false,error}` (un throw est masqué par un digest en prod) + helper `lib/client-errors.ts` (bundle périmé après redeploy → « recharge la page »). `validerPhotoIA`/`pivoterPhotoProduit`/`genererDescriptionProduit` throwent encore.
> - **Params modèle image confirmés en prod** : `fal-ai/nano-banana-2/edit`, **`resolution: "2K"`** (passée de 4K à 2K le 2026-07-04 — la résolution est le seul levier de coût : 1K=$0.08 · 2K=$0.12 · 4K=$0.16 ; 2K = meilleur rapport netteté/prix), `num_images: 1`, `aspect_ratio: "1:1"`, `output_format: "jpeg"`, **pas de `temperature`** (le modèle ne l'expose pas). L'`IMAGE_PROMPT` a été réécrit pour le multi-photos (même angle que la 1ère photo produit, autres refs = compléments de détails). Description : `fal-ai/any-llm/vision` + `google/gemini-2.5-flash`, `temperature: 0.2`.
> - **Piège Vercel** : uploader un Buffer sharp brut via supabase-js corrompt le JPEG (ré-encodage UTF-8) → toujours `new Blob([new Uint8Array(bytes)])` (helper `uploadJpeg` dans `ia.ts`).
> - ~~Dette V2 : fuite blob mineure dans `garder()`~~ — **réglée** par la réécriture V3 (les object URLs sont révoqués dans `retirerJob`/`reprendrePhotos`).

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Automatiser, dans le dashboard KornerCash, la mise sur fond blanc/4K des photos produit (via fal.ai, avec validation avant/après) et la rédaction de la description produit (via fal.ai LLM multimodal, prompt marketing e-commerce éditable).

**Architecture :** Un module serveur isolé `lib/ai/` (logique pure de construction des requêtes + wrapper fal). Des server actions qui l'appellent. Deux composants clients dédiés (`PhotoIA`, `DescriptionIA`) orchestrés par le formulaire produit existant. Aucun appel fal côté navigateur.

**Tech Stack :** Next.js 16.2.9 (server actions), React 19, TypeScript, `@fal-ai/client`, Supabase Storage (bucket `produits`), vitest (tests unitaires des fonctions pures), `@base-ui/react` (UI existante).

Spec source : `Spec - IA photos et descriptions produits (dashboard).md`.

## Global Constraints

- **Next.js 16.2.9 — version particulière.** Lire `node_modules/next/dist/docs/` avant de coder. Server actions = fichiers `"use server"`. `cookies()` async. Middleware = `proxy.ts`.
- **UI = `@base-ui/react`** (base-nova) : prop `render` et non `asChild` ; `Select` = `value`/`onValueChange` + `items` (Record value→label). Réutiliser les composants `components/ui/*` existants.
- **`npm run build`, `npx tsc --noEmit` et `npm run lint` doivent rester à 0 erreur/warning** à la fin de chaque tâche.
- **fal appelé uniquement depuis les server actions** — `FAL_KEY` jamais exposée au client.
- **Image model sans `temperature`** (`nano-banana-2/edit` ne l'expose pas) : la fidélité repose sur le prompt + la validation humaine. La `temperature` (0.2) ne s'applique qu'à la **description**.
- **Description** : ⚠️ *cette contrainte du plan initial (gabarit strict 80–120 mots / défauts en fin) n'a pas été retenue — voir l'encadré de statut en tête.* Le prompt livré est un prompt **marketing e-commerce** (titre accrocheur, ton adapté à la catégorie, CTA) avec garde-fou anti-invention conservé. Aucune invention de spec dans tous les cas.
- **Vocabulaire métier** : jamais « achat » ; « rachat » / « vente ».
- **Grades DB** (verbatim) : `"Parfait état"`, `"Très bon état"`, `"État correct"`.
- **Git** : commits **locaux** dans le repo `dashboard/` à chaque tâche. **Ne pas push** (le repo auto-déploie sur Vercel) tant qu'Anatole n'a pas mis `FAL_KEY`/modèles dans Vercel — sinon la feature déployée planterait au clic. Le push/déploiement est une étape finale hors de ce plan.
- Toutes les commandes s'exécutent depuis `1 PROJETS/kornerCash/dashboard/`.

---

## File Structure

**Créés :**
- `lib/ai/builders.ts` — logique PURE (prompts figés, assemblage des inputs, ordre des défauts). Zéro import externe → testable en isolation.
- `lib/ai/builders.test.ts` — tests vitest des fonctions pures.
- `lib/ai/config.ts` — ids de modèles lus depuis l'environnement (avec défauts).
- `lib/ai/template.ts` — le template carré encodé en constante base64 data URI.
- `lib/ai/fal.ts` — wrapper `server-only` autour de `@fal-ai/client` (`genererPhotoProduit`, `genererDescription`).
- `lib/actions/ia.ts` — server actions (`traiterPhotoIA`, `validerPhotoIA`, `genererDescriptionProduit`).
- `lib/image.ts` — utilitaires client (compression + conversion data URI), extraits de `photo-upload.tsx`.
- `components/produits/photo-ia.tsx` — photos principales (capture → génération → avant/après).
- `components/produits/description-ia.tsx` — description générée/éditable.
- `vitest.config.ts` — config test.

**Modifiés :**
- `package.json` — deps `@fal-ai/client`, `vitest` (dev) + script `test`.
- `.env.example` + `.env.local` — `FAL_KEY`, `FAL_IMAGE_MODEL`, `FAL_TEXT_MODEL`, `FAL_TEXT_LLM`.
- `components/shared/photo-upload.tsx` — importe `compressImage` depuis `lib/image.ts` (au lieu de le définir en interne). Reste utilisé tel quel pour les **photos de défauts**.
- `lib/actions/produits.ts` — `ProduitInput` + `createProduit`/`updateProduit` gèrent `description_fr`.
- `components/produits/produit-form.tsx` — orchestre `PhotoIA` + `DescriptionIA`, état `description`.

---

## Task 1 : Setup (deps, env, vitest, template)

**Files:**
- Modify: `package.json`
- Create: `vitest.config.ts`
- Modify: `.env.example`, `.env.local`
- Create: `lib/ai/template.ts`

**Interfaces:**
- Produces: script `npm run test` fonctionnel ; `TEMPLATE_CARRE_DATA_URI: string` exporté depuis `lib/ai/template.ts` ; variables d'env `FAL_KEY`, `FAL_IMAGE_MODEL`, `FAL_TEXT_MODEL`, `FAL_TEXT_LLM`.

- [x] **Step 1 : Installer les dépendances**

```bash
npm install @fal-ai/client
npm install -D vitest
```

- [x] **Step 2 : Ajouter le script test à `package.json`**

Dans la section `"scripts"`, ajouter la ligne (le flag `--passWithNoTests` évite un exit code 1 quand aucun test n'existe encore — vitest 4.x) :

```json
    "test": "vitest run --passWithNoTests",
```

- [x] **Step 3 : Créer `vitest.config.ts`**

```ts
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: {
    environment: "node",
    include: ["lib/**/*.test.ts"],
  },
});
```

- [x] **Step 4 : Ajouter les variables d'environnement**

Dans `.env.example` ET `.env.local`, ajouter :

```bash
# fal.ai — IA photos & descriptions (dashboard)
FAL_KEY=
# Endpoint image (édition multi-images). Changeable pour tester d'autres modèles.
FAL_IMAGE_MODEL=fal-ai/nano-banana-2/edit
# Endpoint LLM multimodal (description).
FAL_TEXT_MODEL=fal-ai/any-llm/vision
# LLM utilisé par l'endpoint vision (Gemini/Claude/GPT…). Changeable pour tester.
FAL_TEXT_LLM=google/gemini-2.5-flash
```

- [x] **Step 5 : Encoder le vrai template dans `lib/ai/template.ts`**

Le template fourni est `../docs/assets/autres/carré 700 700.png` (dans le vault, à côté du dossier `dashboard/`). Générer le fichier avec le data URI base64 :

```bash
mkdir -p lib/ai
printf '// Template carré de placement (fond blanc + cadre noir), encodé en data URI.\nexport const TEMPLATE_CARRE_DATA_URI =\n  "data:image/png;base64,%s";\n' "$(base64 -w0 '../docs/assets/autres/carré 700 700.png')" > lib/ai/template.ts
```

Vérifier : le fichier commence par le commentaire et contient une longue chaîne base64 (pas vide).

- [x] **Step 6 : Vérifier que build et test tournent**

Run: `npm run build && npm run test`
Expected: build OK ; vitest affiche « No test files found » (normal, aucun test encore) et se termine sans erreur.

- [x] **Step 7 : Commit**

```bash
git add package.json package-lock.json vitest.config.ts .env.example lib/ai/template.ts
git commit -m "chore(ia): setup fal client, vitest, env vars et template placeholder"
```

---

## Task 2 : Logique pure — builders + config (TDD)

**Files:**
- Create: `lib/ai/config.ts`
- Create: `lib/ai/builders.ts`
- Test: `lib/ai/builders.test.ts`

**Interfaces:**
- Consumes: rien.
- Produces :
  - `imageModel(): string`, `textModel(): string`, `textLlm(): string` (depuis `config.ts`)
  - `interface ChampsDescription { designation: string; marque: string | null; categorie: string; sousCategorie: string | null; grade: string | null; defauts: string[] }`
  - `IMAGE_PROMPT: string`
  - `buildImageInput(templateDataUri: string, produitDataUri: string): { prompt: string; image_urls: string[]; aspect_ratio: "1:1"; output_format: "jpeg" }` *(livré : signature `(templateDataUri, produitDataUris: string[])` + `resolution: "4K"` — multi-photos, voir encadré)*
  - `DESCRIPTION_SYSTEM_PROMPT: string`
  - `buildDescriptionUserPrompt(c: ChampsDescription): string`
  - `buildDescriptionInput(c: ChampsDescription, photoUrls: string[]): { prompt: string; system_prompt: string; image_urls: string[]; temperature: number }`

- [x] **Step 1 : Écrire les tests (qui échouent)**

`lib/ai/builders.test.ts` :

```ts
import { describe, it, expect } from "vitest";
import {
  IMAGE_PROMPT,
  buildImageInput,
  DESCRIPTION_SYSTEM_PROMPT,
  buildDescriptionUserPrompt,
  buildDescriptionInput,
  type ChampsDescription,
} from "./builders";

const champs: ChampsDescription = {
  designation: "Sac Louis Vuitton Neverfull",
  marque: "Louis Vuitton",
  categorie: "Mode / Accessoires",
  sousCategorie: "Sacs",
  grade: "Très bon état",
  defauts: ["Rayure au dos", "Légère usure des angles"],
};

describe("buildImageInput", () => {
  it("place le template en 1er et le produit en 2e, format 1:1 jpeg", () => {
    const input = buildImageInput("data:template", "data:produit");
    expect(input.image_urls).toEqual(["data:template", "data:produit"]);
    expect(input.aspect_ratio).toBe("1:1");
    expect(input.output_format).toBe("jpeg");
    expect(input.resolution).toBe("4K");
    expect(input.prompt).toBe(IMAGE_PROMPT);
  });
  it("le prompt interdit de retoucher les défauts", () => {
    expect(IMAGE_PROMPT.toLowerCase()).toContain("defect");
    expect(IMAGE_PROMPT.toLowerCase()).toContain("white");
  });
});

describe("buildDescriptionUserPrompt", () => {
  it("liste les défauts numérotés dans l'ordre de saisie", () => {
    const p = buildDescriptionUserPrompt(champs);
    expect(p).toContain("1. Rayure au dos");
    expect(p).toContain("2. Légère usure des angles");
    expect(p.indexOf("1. Rayure au dos")).toBeLessThan(
      p.indexOf("2. Légère usure des angles"),
    );
  });
  it("gère l'absence de défauts", () => {
    const p = buildDescriptionUserPrompt({ ...champs, defauts: [] });
    expect(p).toContain("Défauts: aucun");
  });
  it("marque non précisée affichée proprement", () => {
    const p = buildDescriptionUserPrompt({ ...champs, marque: null });
    expect(p).toContain("Marque: non précisée");
  });
});

describe("DESCRIPTION_SYSTEM_PROMPT", () => {
  it("impose les garde-fous anti-invention et la longueur", () => {
    expect(DESCRIPTION_SYSTEM_PROMPT).toContain("N'invente");
    expect(DESCRIPTION_SYSTEM_PROMPT).toContain("80 à 120 mots");
    expect(DESCRIPTION_SYSTEM_PROMPT).toContain("À noter");
  });
});

describe("buildDescriptionInput", () => {
  it("passe les URLs photos et une température basse", () => {
    const input = buildDescriptionInput(champs, ["https://a/1.jpg"]);
    expect(input.image_urls).toEqual(["https://a/1.jpg"]);
    expect(input.temperature).toBe(0.2);
    expect(input.system_prompt).toBe(DESCRIPTION_SYSTEM_PROMPT);
  });
});
```

- [x] **Step 2 : Lancer les tests → échec attendu**

Run: `npm run test`
Expected: FAIL — `Cannot find module './builders'`.

- [x] **Step 3 : Créer `lib/ai/config.ts`**

```ts
// Ids de modèles fal, configurables via l'environnement (pour tester d'autres modèles).
const DEFAULT_IMAGE_MODEL = "fal-ai/nano-banana-2/edit";
const DEFAULT_TEXT_MODEL = "fal-ai/any-llm/vision";
const DEFAULT_TEXT_LLM = "google/gemini-2.5-flash";

export function imageModel(): string {
  return process.env.FAL_IMAGE_MODEL || DEFAULT_IMAGE_MODEL;
}
export function textModel(): string {
  return process.env.FAL_TEXT_MODEL || DEFAULT_TEXT_MODEL;
}
export function textLlm(): string {
  return process.env.FAL_TEXT_LLM || DEFAULT_TEXT_LLM;
}
```

- [x] **Step 4 : Créer `lib/ai/builders.ts`**

```ts
// Logique PURE de construction des requêtes IA. Aucun import, aucun appel réseau.
// Isolée ici pour être testable sans mock (vitest).

/** Champs produit fournis pour rédiger la description. */
export interface ChampsDescription {
  designation: string;
  marque: string | null;
  categorie: string;
  sousCategorie: string | null;
  grade: string | null;
  /** Descriptions des défauts, dans l'ordre de saisie. */
  defauts: string[];
}

/** Prompt image figé. image_urls = [template (1er), produit (2e)]. */
export const IMAGE_PROMPT =
  "Create an image with the same placement and same background as the first image (a square template), in 1:1 format. " +
  "The product shown in the second image must match the dimensions and placement of the inner area of the square template, " +
  "completely removing the black outline of the template. Keep the product ABSOLUTELY inside the square. " +
  "The background must be pure white. Maintain 100% of the original details, logos, text, wear, scratches and defects of the product — " +
  "do not retouch, smooth, add or remove any detail or defect. Keep the product entire and faithful to the original photo. " +
  "Only remove reflections, glares and shadows from the original shooting conditions to make the product look clean and professional.";

export interface ImageInput {
  prompt: string;
  image_urls: string[];
  aspect_ratio: "1:1";
  output_format: "jpeg";
  resolution: "4K";
}

/** Assemble l'input du modèle d'édition d'image. Ordre imposé : [template, produit]. */
export function buildImageInput(
  templateDataUri: string,
  produitDataUri: string,
): ImageInput {
  return {
    prompt: IMAGE_PROMPT,
    image_urls: [templateDataUri, produitDataUri],
    aspect_ratio: "1:1",
    output_format: "jpeg",
    resolution: "4K",
  };
}

/** Consignes système : garde-fous anti-invention + gabarit fixe. */
export const DESCRIPTION_SYSTEM_PROMPT = [
  "Tu rédiges des descriptions de produits d'occasion pour une boutique, en français.",
  "Règles STRICTES :",
  "- Ne décris QUE ce qui est visible sur les photos ou fourni dans les champs.",
  "- N'invente JAMAIS de spécification non fournie (capacité, année, matériau, authenticité, dimensions).",
  "- Intègre TOUS les défauts fournis, dans l'ordre, sans en ajouter ni en retirer.",
  "- Ne laisse jamais entendre que l'objet est neuf ou sans défaut.",
  "Structure imposée, TOUJOURS dans cet ordre, sans titres de section :",
  "1) une phrase d'accroche (marque + désignation + type d'objet) ;",
  "2) 2 à 3 phrases de présentation (caractéristiques, sans rien inventer) ;",
  "3) une phrase sur l'état général reprenant le grade ;",
  "4) les défauts en fin, introduits par « À noter : », dans l'ordre fourni, séparés par des points-virgules.",
  "Longueur : 80 à 120 mots. Réponds UNIQUEMENT par la description, sans préambule.",
].join("\n");

/** Sérialise les champs pour le prompt utilisateur. */
export function buildDescriptionUserPrompt(c: ChampsDescription): string {
  return [
    `Désignation: ${c.designation}`,
    `Marque: ${c.marque ?? "non précisée"}`,
    `Catégorie: ${c.categorie}${c.sousCategorie ? ` / ${c.sousCategorie}` : ""}`,
    `État (grade): ${c.grade ?? "non précisé"}`,
    c.defauts.length > 0
      ? `Défauts (dans l'ordre): ${c.defauts
          .map((d, i) => `${i + 1}. ${d}`)
          .join(" ")}`
      : "Défauts: aucun",
  ].join("\n");
}

export interface DescriptionInput {
  prompt: string;
  system_prompt: string;
  image_urls: string[];
  temperature: number;
}

/** Assemble l'input du LLM multimodal (le champ `model` est ajouté par le wrapper fal). */
export function buildDescriptionInput(
  c: ChampsDescription,
  photoUrls: string[],
): DescriptionInput {
  return {
    prompt: buildDescriptionUserPrompt(c),
    system_prompt: DESCRIPTION_SYSTEM_PROMPT,
    image_urls: photoUrls,
    temperature: 0.2,
  };
}
```

- [x] **Step 5 : Lancer les tests → succès attendu**

Run: `npm run test`
Expected: PASS (tous les tests verts).

- [x] **Step 6 : Vérifier types**

Run: `npx tsc --noEmit`
Expected: 0 erreur.

- [x] **Step 7 : Commit**

```bash
git add lib/ai/config.ts lib/ai/builders.ts lib/ai/builders.test.ts
git commit -m "feat(ia): builders purs (prompts image/description) + config modèles, testés"
```

---

## Task 3 : Wrapper fal `lib/ai/fal.ts`

**Files:**
- Create: `lib/ai/fal.ts`

**Interfaces:**
- Consumes: `imageModel`, `textModel`, `textLlm` (config) ; `buildImageInput`, `buildDescriptionInput`, `ChampsDescription` (builders) ; `@fal-ai/client`.
- Produces :
  - `genererPhotoProduit(templateDataUri: string, produitDataUri: string): Promise<string>` (retourne l'URL fal temporaire)
  - `genererDescription(champs: ChampsDescription, photoUrls: string[]): Promise<string>`

- [x] **Step 1 : Créer `lib/ai/fal.ts`**

```ts
import "server-only";
import { fal } from "@fal-ai/client";
import { imageModel, textModel, textLlm } from "./config";
import {
  buildImageInput,
  buildDescriptionInput,
  type ChampsDescription,
} from "./builders";

let configured = false;
function ensureConfigured(): void {
  if (configured) return;
  const key = process.env.FAL_KEY;
  if (!key) throw new Error("FAL_KEY manquante (variable d'environnement).");
  fal.config({ credentials: key });
  configured = true;
}

/**
 * Génère la version fond-blanc/4K d'une photo produit.
 * @returns l'URL (temporaire, hébergée par fal) de l'image générée — pas encore stockée.
 */
export async function genererPhotoProduit(
  templateDataUri: string,
  produitDataUri: string,
): Promise<string> {
  ensureConfigured();
  const result = await fal.subscribe(imageModel(), {
    input: buildImageInput(templateDataUri, produitDataUri),
  });
  const url = (result.data as { images?: { url: string }[] }).images?.[0]?.url;
  if (!url) throw new Error("Aucune image renvoyée par l'IA.");
  return url;
}

/** Rédige la description à partir des champs + des URLs de photos (publiques). */
export async function genererDescription(
  champs: ChampsDescription,
  photoUrls: string[],
): Promise<string> {
  ensureConfigured();
  const result = await fal.subscribe(textModel(), {
    input: { ...buildDescriptionInput(champs, photoUrls), model: textLlm() },
  });
  const texte = (result.data as { output?: string }).output?.trim();
  if (!texte) throw new Error("Aucune description renvoyée par l'IA.");
  return texte;
}
```

- [x] **Step 2 : Vérifier types + build**

Run: `npx tsc --noEmit && npm run build`
Expected: 0 erreur. (Le module est `server-only` ; il n'est pas appelé au build.)

- [x] **Step 3 : Commit**

```bash
git add lib/ai/fal.ts
git commit -m "feat(ia): wrapper fal.ts (genererPhotoProduit, genererDescription)"
```

---

## Task 4 : Server actions `lib/actions/ia.ts`

**Files:**
- Create: `lib/actions/ia.ts`

**Interfaces:**
- Consumes: `genererPhotoProduit`, `genererDescription` (fal.ts) ; `TEMPLATE_CARRE_DATA_URI` (template.ts) ; `supabaseAdmin` (`lib/supabase/server.ts`) ; `ChampsDescription` (builders).
- Produces :
  - `traiterPhotoIA(produitDataUri: string): Promise<{ previewUrl: string }>`
  - `validerPhotoIA(falUrl: string): Promise<{ url: string }>`
  - `genererDescriptionProduit(champs: ChampsDescription, photoUrls: string[]): Promise<{ description: string }>`

- [x] **Step 1 : Créer `lib/actions/ia.ts`**

```ts
"use server";

import { randomUUID } from "crypto";
import { supabaseAdmin } from "@/lib/supabase/server";
import { genererPhotoProduit, genererDescription } from "@/lib/ai/fal";
import { TEMPLATE_CARRE_DATA_URI } from "@/lib/ai/template";
import type { ChampsDescription } from "@/lib/ai/builders";

const BUCKET_PRODUITS = "produits";

/**
 * Étape 1 : traite une photo (data URI) → URL fal temporaire.
 * L'image N'est PAS encore stockée dans Supabase (validation employé requise).
 */
export async function traiterPhotoIA(
  produitDataUri: string,
): Promise<{ previewUrl: string }> {
  const previewUrl = await genererPhotoProduit(
    TEMPLATE_CARRE_DATA_URI,
    produitDataUri,
  );
  return { previewUrl };
}

/**
 * Étape 2 (« Garder ») : télécharge l'image fal validée → upload bucket `produits` → URL publique.
 */
export async function validerPhotoIA(
  falUrl: string,
): Promise<{ url: string }> {
  const resp = await fetch(falUrl);
  if (!resp.ok) throw new Error("Téléchargement de l'image IA échoué.");
  const arrayBuffer = await resp.arrayBuffer();

  const path = `${randomUUID()}.jpg`;
  const db = supabaseAdmin();
  const { error } = await db.storage
    .from(BUCKET_PRODUITS)
    .upload(path, Buffer.from(arrayBuffer), {
      contentType: "image/jpeg",
      upsert: false,
    });
  if (error) throw error;

  const { data } = db.storage.from(BUCKET_PRODUITS).getPublicUrl(path);
  return { url: data.publicUrl };
}

/** Génère (ou régénère) la description produit. */
export async function genererDescriptionProduit(
  champs: ChampsDescription,
  photoUrls: string[],
): Promise<{ description: string }> {
  const description = await genererDescription(champs, photoUrls);
  return { description };
}
```

- [x] **Step 2 : Vérifier types + lint + build**

Run: `npx tsc --noEmit && npm run lint && npm run build`
Expected: 0 erreur.

- [x] **Step 3 : Commit**

```bash
git add lib/actions/ia.ts
git commit -m "feat(ia): server actions traiterPhotoIA / validerPhotoIA / genererDescriptionProduit"
```

---

## Task 5 : Utilitaire image partagé `lib/image.ts`

**Files:**
- Create: `lib/image.ts`
- Modify: `components/shared/photo-upload.tsx`

**Interfaces:**
- Produces :
  - `compressImage(file: File): Promise<Blob>` (déplacé depuis photo-upload)
  - `blobToDataUri(blob: Blob): Promise<string>`

- [x] **Step 1 : Créer `lib/image.ts`**

Contenu = la fonction `compressImage` actuellement dans `photo-upload.tsx` (lignes 11-41), + un helper data URI :

```ts
// Utilitaires image côté navigateur (canvas / FileReader). N'appeler que côté client.

/** Redimensionne (max 1400px) et compresse en JPEG → Blob léger accepté par le bucket. */
export function compressImage(file: File): Promise<Blob> {
  return new Promise((resolve, reject) => {
    const reader = new FileReader();
    reader.onload = () => {
      const img = new window.Image();
      img.onload = () => {
        const max = 1400;
        let w = img.width;
        let h = img.height;
        if (w > max || h > max) {
          const s = max / Math.max(w, h);
          w = Math.round(w * s);
          h = Math.round(h * s);
        }
        const canvas = document.createElement("canvas");
        canvas.width = w;
        canvas.height = h;
        canvas.getContext("2d")?.drawImage(img, 0, 0, w, h);
        canvas.toBlob(
          (b) => (b ? resolve(b) : reject(new Error("compression"))),
          "image/jpeg",
          0.8,
        );
      };
      img.onerror = () => reject(new Error("image illisible"));
      img.src = reader.result as string;
    };
    reader.onerror = () => reject(new Error("lecture impossible"));
    reader.readAsDataURL(file);
  });
}

/** Convertit un Blob en data URI base64 (pour l'envoyer à une server action). */
export function blobToDataUri(blob: Blob): Promise<string> {
  return new Promise((resolve, reject) => {
    const reader = new FileReader();
    reader.onload = () => resolve(reader.result as string);
    reader.onerror = () => reject(new Error("lecture impossible"));
    reader.readAsDataURL(blob);
  });
}
```

- [x] **Step 2 : Mettre à jour `components/shared/photo-upload.tsx`**

Supprimer la définition locale de `compressImage` (lignes 10-41) et l'importer :

Remplacer la ligne d'import du haut (après les imports lucide) par l'ajout de :
```ts
import { compressImage } from "@/lib/image";
```
Puis supprimer entièrement le bloc `function compressImage(...) { ... }` local. Le reste du fichier est inchangé (il appelle toujours `compressImage`).

- [x] **Step 3 : Vérifier types + lint + build**

Run: `npx tsc --noEmit && npm run lint && npm run build`
Expected: 0 erreur (photo-upload compile toujours, `compressImage` vient maintenant de `lib/image.ts`).

- [x] **Step 4 : Commit**

```bash
git add lib/image.ts components/shared/photo-upload.tsx
git commit -m "refactor(ia): extrait compressImage + blobToDataUri dans lib/image.ts"
```

---

## Task 6 : Composant `PhotoIA`

**Files:**
- Create: `components/produits/photo-ia.tsx`

**Interfaces:**
- Consumes: `compressImage`, `blobToDataUri` (lib/image) ; `traiterPhotoIA`, `validerPhotoIA` (lib/actions/ia) ; `Button` (components/ui).
- Produces: `<PhotoIA value={string[]} onChange={(urls: string[]) => void} />` — gère la liste des photos principales validées.

> **Note capture** : on n'utilise PAS `CameraCapture` existant (il uploade **directement** dans le bucket via `uploadProduitPhoto` et renvoie une URL publique → il court-circuite l'IA). `PhotoIA` a besoin du **fichier brut** pour le passer par l'IA. On utilise donc deux `<input type="file">` : un avec `capture="environment"` (déclenche l'appareil photo sur mobile) pour « Photo », un sans pour « Importer ». La webcam live desktop est hors périmètre V1.

- [x] **Step 1 : Créer `components/produits/photo-ia.tsx`**

```tsx
"use client";

import Image from "next/image";
import { useRef, useState } from "react";
import { toast } from "sonner";
import { Camera, Upload, X, Loader2 } from "lucide-react";
import { Button } from "@/components/ui/button";
import { compressImage, blobToDataUri } from "@/lib/image";
import { traiterPhotoIA, validerPhotoIA } from "@/lib/actions/ia";

type Comparaison = { originalUrl: string; falUrl: string };

export function PhotoIA({
  value,
  onChange,
}: {
  value: string[];
  onChange: (urls: string[]) => void;
}) {
  const cameraRef = useRef<HTMLInputElement>(null);
  const importRef = useRef<HTMLInputElement>(null);
  const [loading, setLoading] = useState(false);
  const [comparaison, setComparaison] = useState<Comparaison | null>(null);

  // Traite un fichier (import ou capture) : compression → data URI → IA.
  async function traiter(file: File) {
    setLoading(true);
    try {
      const blob = await compressImage(file);
      const dataUri = await blobToDataUri(blob);
      const originalUrl = URL.createObjectURL(blob);
      const { previewUrl } = await traiterPhotoIA(dataUri);
      setComparaison({ originalUrl, falUrl: previewUrl });
    } catch {
      toast.error("Échec du traitement IA. Réessaie.");
    } finally {
      setLoading(false);
    }
  }

  async function garder() {
    if (!comparaison) return;
    setLoading(true);
    try {
      const { url } = await validerPhotoIA(comparaison.falUrl);
      onChange([...value, url]);
      URL.revokeObjectURL(comparaison.originalUrl);
      setComparaison(null);
    } catch {
      toast.error("Impossible d'enregistrer la photo. Réessaie.");
    } finally {
      setLoading(false);
    }
  }

  function reprendre() {
    if (comparaison) URL.revokeObjectURL(comparaison.originalUrl);
    setComparaison(null);
  }

  return (
    <div className="flex flex-col gap-3">
      {/* Photos déjà validées */}
      <div className="flex flex-wrap gap-3">
        {value.map((url) => (
          <div
            key={url}
            className="relative size-20 overflow-hidden rounded-md border"
          >
            <Image src={url} alt="" fill sizes="80px" className="object-cover" />
            <button
              type="button"
              onClick={() => onChange(value.filter((u) => u !== url))}
              className="absolute right-0 top-0 bg-black/60 p-0.5 text-white"
              aria-label="Retirer"
            >
              <X className="size-3" />
            </button>
          </div>
        ))}
        {!comparaison && !loading ? (
          <>
            <button
              type="button"
              onClick={() => cameraRef.current?.click()}
              className="flex size-20 flex-col items-center justify-center gap-1 rounded-md border border-dashed text-xs text-muted-foreground hover:bg-accent"
            >
              <Camera className="size-4" />
              Photo
            </button>
            <button
              type="button"
              onClick={() => importRef.current?.click()}
              className="flex size-20 flex-col items-center justify-center gap-1 rounded-md border border-dashed text-xs text-muted-foreground hover:bg-accent"
            >
              <Upload className="size-4" />
              Importer
            </button>
          </>
        ) : null}
        {loading ? (
          <div className="flex size-20 flex-col items-center justify-center gap-1 rounded-md border text-xs text-muted-foreground">
            <Loader2 className="size-4 animate-spin" />
            Traitement IA…
          </div>
        ) : null}
      </div>

      {/* Écran avant / après */}
      {comparaison ? (
        <div className="rounded-lg border p-3">
          <div className="grid grid-cols-2 gap-3">
            <figure className="flex flex-col gap-1">
              <figcaption className="text-xs text-muted-foreground">
                Avant
              </figcaption>
              {/* eslint-disable-next-line @next/next/no-img-element */}
              <img
                src={comparaison.originalUrl}
                alt="Original"
                className="aspect-square w-full rounded-md border object-contain"
              />
            </figure>
            <figure className="flex flex-col gap-1">
              <figcaption className="text-xs text-muted-foreground">
                Après (IA)
              </figcaption>
              {/* eslint-disable-next-line @next/next/no-img-element */}
              <img
                src={comparaison.falUrl}
                alt="Version IA"
                className="aspect-square w-full rounded-md border object-contain"
              />
            </figure>
          </div>
          <div className="mt-3 flex justify-end gap-2">
            <Button variant="outline" onClick={reprendre} disabled={loading}>
              Reprendre une meilleure photo
            </Button>
            <Button onClick={garder} disabled={loading}>
              Garder la version IA
            </Button>
          </div>
        </div>
      ) : null}

      <input
        ref={cameraRef}
        type="file"
        accept="image/*"
        capture="environment"
        hidden
        onChange={(e) => {
          const f = e.target.files?.[0];
          if (f) traiter(f);
          e.target.value = "";
        }}
      />
      <input
        ref={importRef}
        type="file"
        accept="image/*"
        hidden
        onChange={(e) => {
          const f = e.target.files?.[0];
          if (f) traiter(f);
          e.target.value = "";
        }}
      />
    </div>
  );
}
```

- [x] **Step 2 : Vérifier types + lint + build**

Run: `npx tsc --noEmit && npm run lint && npm run build`
Expected: 0 erreur.

- [x] **Step 3 : Commit**

```bash
git add components/produits/photo-ia.tsx
git commit -m "feat(ia): composant PhotoIA (capture → génération → validation avant/après)"
```

---

## Task 7 : Composant `DescriptionIA`

**Files:**
- Create: `components/produits/description-ia.tsx`

**Interfaces:**
- Consumes: `genererDescriptionProduit` (lib/actions/ia) ; `ChampsDescription` (builders) ; `Button`, `Textarea` (components/ui).
- Produces: `<DescriptionIA champs={ChampsDescription} photoUrls={string[]} value={string} onChange={(v: string) => void} />`.

- [x] **Step 1 : Créer `components/produits/description-ia.tsx`**

```tsx
"use client";

import { useState } from "react";
import { toast } from "sonner";
import { Sparkles, Loader2 } from "lucide-react";
import { Button } from "@/components/ui/button";
import { Textarea } from "@/components/ui/textarea";
import { genererDescriptionProduit } from "@/lib/actions/ia";
import type { ChampsDescription } from "@/lib/ai/builders";

export function DescriptionIA({
  champs,
  photoUrls,
  value,
  onChange,
}: {
  champs: ChampsDescription;
  photoUrls: string[];
  value: string;
  onChange: (v: string) => void;
}) {
  const [loading, setLoading] = useState(false);

  async function generer() {
    if (!champs.designation.trim()) {
      toast.error("Renseigne au moins la désignation avant de générer.");
      return;
    }
    setLoading(true);
    try {
      const { description } = await genererDescriptionProduit(champs, photoUrls);
      onChange(description);
    } catch {
      toast.error("Échec de la génération. Réessaie.");
    } finally {
      setLoading(false);
    }
  }

  return (
    <div className="flex flex-col gap-2">
      <Textarea
        value={value}
        onChange={(e) => onChange(e.target.value)}
        placeholder="Description du produit (générée par l'IA, éditable)…"
        rows={5}
      />
      <Button
        type="button"
        variant="outline"
        size="sm"
        onClick={generer}
        disabled={loading}
        className="self-start"
      >
        {loading ? (
          <Loader2 className="size-4 animate-spin" />
        ) : (
          <Sparkles className="size-4" />
        )}
        {value.trim() ? "Régénérer la description" : "Générer la description"}
      </Button>
    </div>
  );
}
```

- [x] **Step 2 : Vérifier types + lint + build**

Run: `npx tsc --noEmit && npm run lint && npm run build`
Expected: 0 erreur.

- [x] **Step 3 : Commit**

```bash
git add components/produits/description-ia.tsx
git commit -m "feat(ia): composant DescriptionIA (générer/régénérer + éditable)"
```

---

## Task 8 : Câbler `description_fr` dans l'action produits

**Files:**
- Modify: `lib/actions/produits.ts`

**Interfaces:**
- Produces: `ProduitInput` inclut `description_fr?: string | null` ; `createProduit`/`updateProduit` l'écrivent.

- [x] **Step 1 : Ajouter le champ au type `ProduitInput`**

Dans `lib/actions/produits.ts`, ajouter dans le type `ProduitInput` (après `titre_fr`) :

```ts
  description_fr?: string | null;
```

- [x] **Step 2 : Écrire la valeur dans `createProduit`**

Dans l'objet `row: TablesInsert<"produits">` de `createProduit`, ajouter :

```ts
    description_fr: input.description_fr || null,
```

- [x] **Step 3 : Écrire la valeur dans `updateProduit`**

Dans l'objet `.update({ ... })` de `updateProduit`, ajouter :

```ts
      description_fr: input.description_fr || null,
```

- [x] **Step 4 : Vérifier types + build**

Run: `npx tsc --noEmit && npm run build`
Expected: 0 erreur (`description_fr` existe déjà dans `TablesInsert<"produits">` / `TablesUpdate`).

- [x] **Step 5 : Commit**

```bash
git add lib/actions/produits.ts
git commit -m "feat(ia): persiste description_fr (create/update produit)"
```

---

## Task 9 : Orchestration dans `produit-form.tsx`

**Files:**
- Modify: `components/produits/produit-form.tsx`

**Interfaces:**
- Consumes: `PhotoIA`, `DescriptionIA`, `ChampsDescription`.
- Produces: formulaire complet (photos principales via IA, description IA, `description_fr` envoyé).

- [x] **Step 1 : Imports**

Ajouter en haut de `produit-form.tsx` :

```ts
import { PhotoIA } from "@/components/produits/photo-ia";
import { DescriptionIA } from "@/components/produits/description-ia";
import type { ChampsDescription } from "@/lib/ai/builders";
```

- [x] **Step 2 : État description**

Après la ligne `const [photos, setPhotos] = useState<string[]>(editing?.photos ?? []);`, ajouter :

```ts
  const [description, setDescription] = useState(editing?.description_fr ?? "");
```

- [x] **Step 3 : Construire `ChampsDescription` pour l'IA**

Juste avant le `return (`, après `const sousCats = ...`, ajouter :

```ts
  const champsIA: ChampsDescription = {
    designation: titre.trim(),
    marque: marque.trim() || null,
    categorie: categories.find((c) => c.id === categorieId)?.nom_fr ?? "",
    sousCategorie:
      sousCats.find((sc) => sc.id === sousCatId)?.nom_fr ?? null,
    grade: grade || null,
    defauts: defauts.map((d) => d.description.trim()).filter(Boolean),
  };
```

- [x] **Step 4 : Remplacer le `PhotoUpload` des photos principales par `PhotoIA`**

Dans le `<Field label="Photos" required error={errors.photos}>`, remplacer le `<PhotoUpload value={photos} onChange={...} />` par :

```tsx
            <PhotoIA
              value={photos}
              onChange={(urls) => {
                setPhotos(urls);
                clearErr("photos");
              }}
            />
```

(Le `PhotoUpload` des **défauts** plus bas reste inchangé.)

- [x] **Step 5 : Ajouter le champ Description après le bloc Photos**

Juste après la `</Field>` du bloc Photos (avant le bloc « Défauts — repliable »), insérer :

```tsx
          <Field label="Description">
            <DescriptionIA
              champs={champsIA}
              photoUrls={photos}
              value={description}
              onChange={setDescription}
            />
          </Field>
```

- [x] **Step 6 : Envoyer `description_fr` dans `input`**

Dans l'objet `const input: ProduitInput = { ... }` de la fonction `submit`, ajouter après `titre_fr` :

```ts
      description_fr: description.trim() || null,
```

- [x] **Step 7 : Vérifier types + lint + build**

Run: `npx tsc --noEmit && npm run lint && npm run build`
Expected: 0 erreur.

- [x] **Step 8 : Commit**

```bash
git add components/produits/produit-form.tsx
git commit -m "feat(ia): branche PhotoIA + DescriptionIA dans le formulaire produit"
```

---

## Task 10 : Vérification manuelle bout-en-bout (nécessite la vraie clé + template)

**Pré-requis (Anatole) : ✅ fournis** — template `../docs/assets/autres/carré 700 700.png` (encodé en Task 1), `FAL_KEY` + `FAL_IMAGE_MODEL=fal-ai/nano-banana-2/edit` + `FAL_TEXT_MODEL` + `FAL_TEXT_LLM` déjà dans `.env.local`.

- [x] **Step 1 : Vérifier les pré-requis (déjà en place)**

`lib/ai/template.ts` contient le vrai template (Task 1) et les variables fal sont dans `.env.local`. Rien à faire — passer à l'étape suivante.

- [x] **Step 2 : Lancer le dashboard en local**

Run: `npm run dev`
Aller sur `/produits`, ouvrir « Nouveau produit ».

- [x] **Step 3 : Tester le flux photo**

Importer une photo produit → vérifier : indicateur « Traitement IA… », écran avant/après, « Garder la version IA » ajoute une vignette fond blanc, « Reprendre une meilleure photo » réinitialise. Vérifier dans Supabase que seule la version validée est dans le bucket `produits`.

- [x] **Step 4 : Tester le flux description**

Remplir désignation/marque/catégorie/état + 2 défauts → « Générer la description » → vérifier : gabarit respecté (accroche → présentation → état → « À noter : » défauts dans l'ordre), 80-120 mots, éditable, « Régénérer » produit une nouvelle version. Enregistrer → vérifier `produits.description_fr` en DB et l'affichage sur le site.

- [x] **Step 5 : Noter le rendu**

Si le modèle image déforme l'objet / gomme un défaut : tester un autre `FAL_IMAGE_MODEL`. Si la plume ne convient pas : tester un autre `FAL_TEXT_LLM` (ex. `anthropic/claude-3.5-sonnet`). Aucun changement de code — que des variables d'env.

---

## Notes de fin

- **Déploiement** : ✅ **fait le 2026-07-02** — `FAL_KEY` + modèles dans Vercel, push → auto-deploy sur `kornercash-dashboard.vercel.app`.
- **Coûts** : 1 appel image par génération (et par « Régénérer »), 1 appel LLM par génération/régénération de description. Le client paie via son compte fal.
- **Fallback data URI trop lourd** : ✅ **adopté par défaut** (limite ~4,5 Mo Vercel) — les photos de référence sont uploadées sur le stockage fal (`uploadImageToFal` / `uploadRefPhoto`) avant génération, plus d'envoi base64 inline.
- **Doc projet** : à la fin, mettre à jour `Road map kornerCash.md` (Phase 3 dashboard : bloc IA) et le MEMORY.
