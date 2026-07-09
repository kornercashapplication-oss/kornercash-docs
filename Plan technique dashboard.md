# PLAN TECHNIQUE — Dashboard KornerCash

Plan d'écriture et d'ordonnancement du TypeScript pour le dashboard interne.
Stack : **Next.js 16 (App Router) + React 19 + TypeScript + Tailwind 4 + Base UI (`@base-ui/react`, installé via shadcn) + Supabase**.
Référence visuelle/fonctionnelle initiale : prototype `KornerCash-Interne/`. Source de vérité données : Supabase (9 tables).

> ⚠️ Next.js 16 = ruptures d'API. `cookies()` est asynchrone (`await cookies()`), et le middleware est renommé **`proxy.ts`** (fonction `proxy`, à la racine). Avant de coder, lire les guides dans `node_modules/next/dist/docs/` (rappel `dashboard/AGENTS.md`).

**Statut (2026-07-04) : dashboard TERMINÉ et EN LIGNE → https://kornercash-dashboard.vercel.app** (Vercel, auto-deploy à chaque push). 7 écrans opérationnels (Tableau de bord, Ventes, Rachats, Produits, **Atelier**, Clients, Module Or) + pages imprimables (tickets vente/rachat, étiquette). **Feature IA photos & descriptions produits déployée en prod** (fal.ai `nano-banana-2/edit`, résolution **2K** — voir Étape 7). **Atelier de saisie en lot déployé** : génération d'images IA **asynchrone** (file fal `queue`) pour ajouter ~1000 produits sans attente bloquante (voir section dédiée en fin de doc). Dossier code : `dashboard/`.

**Reste à faire** : compléter `lib/domain/entreprise.ts` (coordonnées légales réelles : raison sociale, adresse, IDE/TVA), faire valider les tickets par la fiduciaire.

**Scan douchette code-barres (2026-07-04, déployé)** : composant partagé `components/shared/scanner-dialog.tsx` (dialog + input auto-focus `initialFocus` Base UI, Entrée/Tab, auto-validation 250 ms sur pattern `^[A-Z]{3}-\d{4,}$`, refocus permanent, historique des scans). Branché sur la liste Produits (scan → `openEdit` de la fiche) et sur Nouvelle vente (mode caisse enchaîné via `enchaine`, re-scan = quantité +1 ; `addProduit` plafonne désormais au stock disponible — fin de la dette survente ; bonus : code exact + Entrée dans le champ recherche = ajout direct). Zéro dépendance (la douchette émule un clavier). `trouverParCode()` résout par `code_barres` OU `code_article` (insensible à la casse).

→ Schéma DB : `database architecture.md` · Cadrage : `Road map kornerCash.md` (décisions Bloc 0)
→ Schéma exact des colonnes (source de vérité) : `dashboard/lib/supabase/types.ts` (complet, 9 tables — celui du site est un sous-ensemble volontaire à 8 tables)

---

## Principes d'architecture

1. **Server Components par défaut.** Mutations via **Server Actions**. Le navigateur ne parle jamais à Supabase directement.
2. **Service key côté serveur uniquement** (`lib/supabase/server.ts`). Jamais dans un Client Component, jamais exposée. Le dashboard ignore le RLS (clé maître) — c'est voulu.
3. **Tout typé** depuis les types générés Supabase (`Database`). Aucun `any`.
4. **Une couche données par domaine** : `queries` (lecture) + `actions` (écriture), jamais de SQL dans les composants.
5. **Découpage par domaine métier** : produits / clients / rachats / ventes / or / dashboard.
6. **Auth simple** : 2 mots de passe (équipe/patron) en variables d'env → cookie de session **httpOnly signé HMAC-SHA256** (pas de stockage serveur, rôle + expiration signés avec `SESSION_SECRET`, durée 12 h) → `proxy.ts` (garde-fou rapide « pas de cookie → /login », la vérif complète signature+rôle se fait côté serveur). Le rôle `patron` débloque l'affichage des chiffres **et** l'accès aux pages Tableau de bord + Or (`patronOnly` dans la nav).

---

## Arborescence cible

Les écrans CRUD se pilotent par **modales client** (formulaires en `Dialog`) dans une seule page par domaine — pas de sous-routes `nouveau/` ni `[id]/` (seule exception : la fiche client `clients/[id]`). Chaque page serveur charge les données puis délègue à un composant client (`*-client.tsx`).

```
dashboard/
├── app/
│   ├── layout.tsx                      # root (fonts, providers, Toaster)
│   ├── globals.css
│   ├── (auth)/login/page.tsx           # page login (2 mots de passe)
│   ├── (app)/
│   │   ├── layout.tsx                  # shell : sidebar + topbar + mobile-nav
│   │   ├── page.tsx                    # Tableau de bord (KPI + graphes, patron)
│   │   ├── ventes/page.tsx             # ventes : liste + création (modale)
│   │   ├── rachats/page.tsx            # rachats : liste + création (pièce id + signature)
│   │   ├── produits/page.tsx           # produits : liste filtrable + CRUD (modale)
│   │   ├── atelier/page.tsx            # atelier : saisie en lot + file de jobs IA (maxDuration 60)
│   │   ├── clients/page.tsx            # clients : liste + filtre blacklist
│   │   │   └── [id]/page.tsx           # fiche client + historique + toggle blacklist
│   │   └── or/page.tsx                 # module Or : KPI + rachat/vente or (patron)
│   ├── etiquette/[id]/page.tsx         # étiquette produit imprimable (code-barres)
│   └── ticket/
│       ├── vente/[id]/page.tsx         # ticket de vente imprimable
│       └── rachat/[id]/page.tsx        # ticket de rachat imprimable
├── components/
│   ├── ui/                             # Base UI (installés via shadcn) : button, input, dialog, table, select, switch…
│   ├── layout/                         # sidebar, topbar, mobile-nav, money-gate, nav-items.ts
│   ├── shared/                         # data-table, kpi-card, photo-upload, camera-capture (caméra live getUserMedia), scanner-dialog (douchette), signature-pad, money-value, confirm-dialog, empty-state
│   ├── tickets/                        # ticket-print, ticket-vente, ticket-rachat, ticket-parts
│   └── (domaine)/                      # produits/ clients/ ventes/ rachats/ or/ dashboard/ auth/ atelier/
│       │                               #   produits/ : photo-ia.tsx & description-ia.tsx
│       │                               #   ventes/ : ajout-client-rapide.tsx & ajout-produit-rapide.tsx
│       │                               #   atelier/ : atelier-client, atelier-form, file-jobs, job-validation, ref-uploader
├── lib/
│   ├── supabase/
│   │   ├── types.ts                    # types générés (Database) — 9 tables
│   │   └── server.ts                   # supabaseAdmin() service-role (server-only, cache module)
│   ├── auth/                           # constants.ts (Role, cookie), session.ts (HMAC), passwords.ts, actions.ts (login/logout)
│   ├── ai/                             # config.ts (ids modèles fal), fal.ts (file queue + subscribe desc), builders.ts (prompts, + builders.test.ts), template.ts
│   ├── domain/
│   │   ├── constants.ts                # CARATS, GRADES, STATUTS, modes paiement, mapping catégories
│   │   └── entreprise.ts               # identité légale + TVA (À COMPLÉTER : réelles)
│   ├── queries/                        # produits clients rachats ventes or dashboard categories atelier (lecture)
│   ├── actions/                        # produits clients rachats ventes or uploads ia atelier (server actions)
│   ├── barcode.ts · charts.ts · format.ts · image.ts · utils.ts · client-errors.ts
├── proxy.ts                            # ex-middleware Next 16 : pas de cookie → /login
├── next.config.ts                      # serverActions bodySizeLimit 8mb + remotePatterns Supabase
├── .env.local / .env.example           # voir « Variables d'environnement » ci-dessous
├── components.json · tsconfig.json · package.json · vitest.config.ts
```

**Variables d'environnement** (`.env.example`) : `NEXT_PUBLIC_SUPABASE_URL`, `SUPABASE_SERVICE_KEY`, `DASHBOARD_PW_EQUIPE`, `DASHBOARD_PW_PATRON`, `SESSION_SECRET`, `FAL_KEY`, `FAL_IMAGE_MODEL`, `FAL_TEXT_MODEL`, `FAL_TEXT_LLM`.

> Pas de `tailwind.config.ts` : Tailwind 4 se configure dans `globals.css` (`@import "tailwindcss"`) + `postcss.config.mjs`.

---

## Ordre d'écriture du TypeScript

L'ordre a suivi une logique **bottom-up** : fondations typées avant l'UI. **Toutes les étapes 1 à 8 sont réalisées.** Statut par étape ci-dessous.

### Étape 1 — Fondations (aucune UI visible) ✅ FAIT
1. Scaffold Next.js 16 + Tailwind 4 + Base UI + `.env.local`
2. `lib/supabase/types.ts` — **générer** les types depuis Supabase (9 tables)
3. `lib/supabase/server.ts` — `supabaseAdmin()` service-role (cache au niveau module)
4. `lib/domain/constants.ts` — `CARATS`, `GRADES` (états), `STATUTS`, modes de paiement, mapping catégories prototype→officielles
5. `lib/format.ts` — `chf()` (arrondi), `chfExact()` (centimes — utilisé partout via `MoneyValue` depuis le 2026-07-04), `chfRappen()`, formats de date ; référence produit auto générée à la création

### Étape 2 — Auth & shell ✅ FAIT
6. `lib/auth/` — `constants.ts` (Role, `SESSION_COOKIE`, durée), `session.ts` (cookie HMAC-SHA256 : `createSession`/`getSession`/`destroySession`), `passwords.ts`, `actions.ts` (login/logout)
7. `proxy.ts` — pas de cookie de session → `/login`
8. `app/(auth)/login/page.tsx` + composant `login-form`
9. `app/(app)/layout.tsx` — sidebar + topbar + mobile-nav (tiroir) ; `MoneyGate` masque les chiffres si rôle ≠ patron ; pages `patronOnly` (Tableau de bord, Or)

### Étape 3 — Couche données par domaine (queries + actions) ✅ FAIT
Pour chaque domaine : 1 fichier lecture (`queries/`) + 1 fichier mutations (`actions/`), entièrement typés.
10. `produits` (liste, create, update, upload photo, remboursement / remise en stock)
11. `clients` (liste, fiche + historique, create, update, `toggleBlacklist`)
12. `rachats` (liste, create + upload pièce d'identité + signature)
13. `ventes` (historique + `createVenteMagasin` : lignes reliées à un client)
14. `or` (liste + create rachat/vente or)
15. `dashboard` (agrégats KPI : CA, bénéfice, capital immobilisé, marge, valeur stock)
16. `categories` (arbre catégories → sous-catégories) · `uploads` (actions upload partagées)

### Étape 4 — Composants partagés ✅ FAIT
17. `data-table` générique (tri/recherche), `kpi-card`, `photo-upload` (→ bucket `produits`), `camera-capture` (prise de photo directe), `signature-pad`, `money-value` (respecte le MoneyGate), `confirm-dialog`, `empty-state`

### Étape 5 — Écrans (du plus simple au plus complexe) ✅ FAIT
18. **Tableau de bord** (KPI + graphes, `patronOnly`)
19. **Produits** (liste filtrable par statut/catégorie, CRUD en modale, photos + caméra, gestion des défauts, référence auto)
20. **Clients** (liste + filtre blacklist, fiche `[id]` + historique + toggle blacklist)
21. **Module Or** (KPI or + formulaire rachat/vente, `patronOnly`)
22. **Rachats** (liste + formulaire : pièce d'identité → bucket privé + signature écran)
23. **Ventes** (liste + création : lignes, **remise par ligne**, recherche produit ; reliée à un client)

### Étape 6 — Finitions ✅ FAIT
24. Étiquette produit imprimable (code-barres Code128 maison `lib/barcode.ts`, route `/etiquette/[id]`) · Remboursement de commande (statut + remise en stock) · Responsive tablette/téléphone (nav mobile tiroir + tableaux scroll) · Tickets vente/rachat imprimables (`/ticket/vente/[id]`, `/ticket/rachat/[id]`)

### Étape 7 — IA photos & descriptions produits ✅ FAIT (déployé en prod)
25. `lib/ai/` (config, `fal.ts`, `builders.ts`, `template.ts`) + `lib/actions/ia.ts` + composants `photo-ia` / `description-ia` dans le formulaire produit. **Photo** : upload des refs sur fal → génération via `nano-banana-2/edit` (fond blanc carré, **résolution `2K` + `num_images: 1` figés en dur** dans `builders.ts` — passés de 4K à 2K le 2026-07-04 pour le coût, 1.5× vs 2×) via la **file fal** (voir section « Formulaire produit » en fin de doc) → validation employé (rotation via `sharp`, y compris après validation via `pivoterPhotoProduit`) → bucket `produits`. **Description** : LLM vision (`any-llm/vision`, Gemini par défaut) à partir des champs + photos, `fal.subscribe` avec timeout 45 s. Modèles configurables par env (`FAL_*`) — mais PAS la résolution.

### Étape 8 — Déploiement ✅ FAIT
26. **En ligne : https://kornercash-dashboard.vercel.app** (Vercel, compte transférable client, auto-deploy à chaque push) + variables d'environnement + test bout-en-bout.
> `next.config.ts` : `serverActions.bodySizeLimit` porté à **8 Mo** (photos) ; page produits `maxDuration = 60` (Vercel Hobby) pour les actions IA lentes.
> Bug résolu au déploiement : `NEXT_PUBLIC_SUPABASE_URL` mal collée (`...cohttps`) → re-saisir la valeur exacte + redeploy SANS cache.

---

## Ce qui reste (dette / suites)

- **`lib/domain/entreprise.ts`** : remplacer les placeholders (`raisonSociale`, `adresseRue`, `npaVille`, `ide` = `CHE-XXX.XXX.XXX`) par les coordonnées réelles de KornerCash — utilisées sur étiquettes et tickets. Régime TVA déjà réglé : `normal`, taux **8.1 %** (mention « dont TVA » incluse dans le prix).
- **Validation fiduciaire** : faire valider le format des tickets par la fiduciaire avant usage comptable réel.
- **Suppression prototype** : le prototype `KornerCash-Interne/` peut être supprimé (remplacé par `dashboard/`).

---

## Points de vigilance

- **Catégories** : à la saisie produit, sélection catégorie → sous-catégorie depuis nos tables (pas la liste plate du prototype). Sous-cat « Bijoux » existe sous **Mode** (18 sous-catégories au total depuis la refonte taxonomie #17).
- **Snapshots** : `ventes.titre_produit` + `prix_unitaire` figés à la vente (le produit unique disparaît du stock).
- **Suppression = archivage** : ne jamais supprimer une donnée historique (statut/flag), surtout rachats et transactions or (valeur légale).
- **Montant or** : colonne générée — ne jamais l'envoyer en écriture (Supabase la refuse).
- **Pièces d'identité** : bucket `pieces-identite` (privé) — URL signée, jamais publique.
- **Colonne `clients.adresses` (jsonb)** : ajoutée le 2026-07-01 (migration `add_adresses_jsonb_clients`) pour les adresses multiples du **site**. Non-bloquant pour le dashboard — le dashboard ignore cette colonne.
- **IA (fal.ai)** : la génération photo produit une **preview fal temporaire** — elle n'est stockée dans le bucket `produits` qu'après validation employé (`validerPhotoIA`). Les photos de référence sont uploadées une par une sur le stockage fal (limite ~4,5 Mo du corps Vercel). Modèles pilotés par env (`FAL_IMAGE_MODEL` = `fal-ai/nano-banana-2/edit`, `FAL_TEXT_MODEL` = `fal-ai/any-llm/vision`, `FAL_TEXT_LLM` = `google/gemini-2.5-flash`) ; **résolution `2K` et `num_images: 1` figés en dur** (`builders.ts`, pas pilotés par env). Timeout : description 45 s (`TEXT_TIMEOUT_MS`) ; les images passent par la file fal, **sans timeout**. `maxDuration = 60` posé sur les pages produits ET atelier.
- **Erreurs server actions IA** : `uploadRefPhoto` et `lancerPhotoIA` renvoient l'erreur **en valeur** `{ok:false, error}` (un throw est masqué par un digest en prod) ; `verifierPhotosIA` renvoie `statut:"echec"+erreur` par job. `validerPhotoIA` / `pivoterPhotoProduit` / `genererDescriptionProduit` throwent encore. Helper client `lib/client-errors.ts` (`messageErreur`, `estActionPerimee` — détecte le bundle client périmé après un redeploy → « recharge la page »).
- **Upload d'un Buffer sharp sur Vercel** : toujours passer par `new Blob([new Uint8Array(bytes)])` (helper `uploadJpeg`, `lib/actions/ia.ts`) — un Buffer brut est ré-encodé UTF-8 par supabase-js sur Vercel → JPEG corrompu.
- **Base de données** : 9 tables (categories, sous_categories, produits, clients, commandes, ventes, rachats, transactions_or, **produit_image_jobs**), 23 migrations. Projet Supabase ref `nztqbfxsaduockzilrve` (région Frankfurt).
- **Pas de skill build-website** pour ce dashboard (règle confirmée : le skill n'est pas utilisé ici).

---

## Atelier de saisie en lot — génération d'images IA asynchrone (2026-07-02) ✅ DÉPLOYÉ

Pour ajouter **~1000 produits** sans subir l'attente **bloquante ~2 min** de la génération IA (le flux modale existant utilise `fal.subscribe`). Nouvel écran **`/atelier`** (accès équipe, nav `PackagePlus`) : formulaire produit **complet** + photos de référence shootées en direct → **« Lancer & produit suivant »** (génération en fond) → enchaînement immédiat → validation / relance / rotation des images **au fil de l'eau** dans un **panneau File d'images** (toujours visible, badges en cours / à valider).

- **File fal** (`@fal-ai/client` v1.10.1) : `queue.submit`→`request_id` (stocké en base) au lieu de `subscribe` bloquant ; suivi par **polling client** (`verifierJobs`, ~4 s) → `queue.status` / `queue.result`. **Depuis le 2026-07-04, le formulaire produit utilise AUSSI la file** (générations parallèles, jobs dans l'état du formulaire — voir section suivante) ; `subscribe` ne sert plus qu'à la description. **Reprise auto au rechargement** (pas de cron : les `request_id` sont en base).
- **Modèle de données** : produit créé en **`brouillon`** (invisible du site **et** de la liste Produits) + nouvelle table **`produit_image_jobs`** (1 produit → N jobs : `fal_request_id`, `ref_photos[]`, `preview_url`, `photo_url`, `rotation`, statut en_cours/a_valider/validee/echec/annulee). **Finalisation explicite** (bouton « Finaliser », activé dès ≥1 image validée) → `en_stock`/`vendu`. **Plusieurs images par produit** possibles.
- **DB** : migration `add_statut_brouillon_produits` (DROP/ADD CHECK `produits.statut` + `brouillon`) + `create_produit_image_jobs` (RLS activée **sans policy** = service-role ; trigger `set_updated_at` ; FK `ON DELETE CASCADE`).
- **Code** : `lib/ai/fal.ts` (soumettre/statut/résultat/annuler) · `lib/actions/atelier.ts` (creerBrouillonEtLancer, lancerGenerationPhoto, verifierJobs, validerJobPhoto, regenererJob, **reprendrePhotoJob**) · `lib/actions/produits.ts` (+`createProduitBrouillon`/`finaliserProduit`) · `lib/queries/atelier.ts` · UI `components/atelier/` (atelier-client [polling+reprise], atelier-form, file-jobs, job-validation [comparaison photo prise ↔ image générée + zoom], ref-uploader). Abandonner un brouillon = bouton « Archiver » (`archiveProduit`) ; le statut de job `annulee` existe en DB/constants mais n'est posé par aucun code actuellement. `listProduits` exclut `brouillon`. **Site : 0 fichier modifié** (les 5 requêtes `catalogue.ts` filtrent `en_stock` = liste blanche → brouillon invisible par construction).
- **Validé en prod** : 189 générations bijoux le 2026-07-03 (0 échec), ~180 brouillons finalisés — flux confirmé en conditions réelles.

---

## Formulaire produit — générations d'images IA en parallèle (2026-07-04) ✅ DÉPLOYÉ

Le flux modale Produits (`components/produits/photo-ia.tsx`) passe du `fal.subscribe` bloquant (une génération à la fois, timeout 58 s) à la **file fal + polling ~4 s** — mêmes fonctions queue que l'atelier. « Générer » = job lancé en fond + zone de photos **libérée immédiatement** → on enchaîne les prises de vue et on lance **plusieurs générations en parallèle** ; chaque résultat se valide au fil de l'eau (Garder / Régénérer / **Reprendre les photos** [refs renvoyées dans la zone] / Ignorer, rotation + barre de progression estimée par job).

- **Pas de table de suivi** (contrairement à l'atelier) : les jobs vivent dans l'état du formulaire — le produit n'existe pas encore en DB à la création et le formulaire est éphémère. Avertissement affiché : générations non gardées perdues si on ferme le formulaire (crédit fal quand même consommé). Pour du volume avec reprise → l'atelier.
- **Server actions** (`lib/actions/ia.ts`) : `uploadRefPhoto` (upload d'1 réf sur fal), `lancerPhotoIA` (submit → `request_id`), `verifierPhotosIA` (polling batch, même logique que `verifierJobs`), `annulerPhotoIA` (best-effort à l'Ignorer d'un job en cours), `validerPhotoIA` (rotation sharp + bucket), `pivoterPhotoProduit` (rotation d'une photo **déjà validée**, écrase le fichier en place + cache-bust `?v=`), `genererDescriptionProduit`. `traiterPhotoIA` + `genererPhotoProduit` (subscribe image) **supprimés** (code mort) — plus de limite 58 s sur l'image.
- **Prise de vue** : `RefUploader` (partagé avec l'atelier, `MAX_REFS = 6`) + `CameraCapture` = **caméra live getUserMedia** avec fallback `<input capture>` (l'input seul est ignoré sur desktop) ; chaque réf est compressée puis uploadée immédiatement sur fal.
- Anti-écrasement : deux « Garder » rapprochés passent par un miroir `valueRef`/`commitValue` (la prop capturée après `await` peut être périmée).

---

## Autres évolutions déployées (2026-07-03 / 2026-07-04)

- **Toggle « Site » par produit** : colonne `produits.visible_site` (migration #22) — Switch en 1ère colonne de la liste Produits (état optimiste, `components/ui/switch.tsx`), action `setVisibleSite` (`lib/actions/produits.ts`). Côté site : les 5 requêtes catalogue filtrent `en_stock` **ET** `visible_site=true` + garde checkout. `updateProduit` n'écrase jamais le toggle.
- **Rachats** : champ optionnel « Code de la pièce » (`clients.code_passeport`, migration #23), pré-rempli depuis la fiche client, affiché sur la fiche ; **photo directe de la pièce d'identité** via `CameraCapture` (prop `upload`) → bucket privé `pieces-identite`.
- **Ventes** : sélection client = **barre de recherche** (nom/e-mail/tél, badge Blacklisté) + création rapide client (`ajout-client-rapide.tsx`, e-mail optionnel) et produit oublié (`ajout-produit-rapide.tsx`, sans IA, avec sous-catégorie) ; plafond stock anti-survente dans `addProduit` ; code exact + Entrée dans la recherche = ajout direct.
- **Montants au centime exact partout** : `chfExact()` via `MoneyValue` (composant unique d'affichage des montants) — fin de l'arrondi `chf()` à l'affichage.
- **Produits** : filtre par sous-catégorie (dépendant de la catégorie, reset au changement), sélecteur d'état en 3 boutons.
- **Fiche client** : historique ventes/rachats avec image + nom d'article.
