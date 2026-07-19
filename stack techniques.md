# STACK TECHNIQUE — KornerCash

Ce document décrit toutes les technologies réellement utilisées dans le projet KornerCash et pourquoi. Il est destiné à un LLM qui doit comprendre l'architecture technique sans ambiguïté.

État à jour : fin de session **2026-07-04**. Les deux apps sont **en ligne** sur Vercel : site https://kornercash-site.vercel.app (repo `kornercashapplication-oss/kornercash-site`, branche `master`) et dashboard https://kornercash-dashboard.vercel.app (repo `kornercash-dashboard`, branche `main`) — auto-deploy à chaque push.

Pour le contexte métier (qu'est-ce que KornerCash, le deal, les catégories, etc.) → voir `contexte kornerCash.md`.

---

## Stack réelle du site (versions exactes)

Source de vérité : `site/package.json`.

| Couche | Technologie | Version | Rôle |
|---|---|---|---|
| **Langage** | TypeScript | `^5` | Langage unique pour tout le projet |
| **Framework** | Next.js (App Router) | `16.2.9` | Site + dashboard, même framework |
| **UI** | React | `19.2.4` (+ react-dom) | Construction de l'interface en composants |
| **CSS** | Tailwind CSS 4 | `^4` (`@tailwindcss/postcss`) | Config **inline** dans `app/globals.css` (plus de `tailwind.config.js`), tokens couleur en variables CSS |
| **Composants UI** | @base-ui/react | `^1.6.0` | Primitives UI (base « base-nova » / shadcn côté dashboard) |
| **Markdown** | react-markdown | `^10.1.0` | Rendu du markdown (ex: descriptions produit générées) |
| **Base de données** | Supabase (PostgreSQL managé) | `@supabase/supabase-js` `^2.109.0` | Tables, requêtes SQL, auth, stockage fichiers |
| **Auth SSR** | @supabase/ssr | `^0.5.2` | Gestion de session par cookie côté serveur + refresh |
| **Emails** | Resend | `^6.16.0` | Vérification d'inscription, reset mot de passe, confirmation de commande (+ expédition/remboursement côté dashboard) |
| **Paiement** | Stripe (Checkout hébergé) | `stripe` `^22.3.0` | Encaissement carte / TWINT / Apple & Google Pay — SDK serveur uniquement (voir section Paiement) |
| **Icônes** | lucide-react | `^1.22.0` | Jeu d'icônes |
| **Toasts** | sonner | `^2.0.7` | Notifications UI |
| **Variantes de style** | class-variance-authority + clsx + tailwind-merge | `^0.7.1` / `^2.1.1` / `^3.6.0` | Composition des classes Tailwind (ex: `components/ui/button.tsx`) |
| **Scripts / seed** | tsx + dotenv | `^4.19.2` / `^16.4.7` (dev) | Scripts one-shot TypeScript (ex: `test:supabase`) |
| **Lint** | eslint + eslint-config-next | `^9` / `16.2.9` (dev) | Qualité de code |

Versions exactes → `site/package.json` (source de vérité).

**Particularités Next 16** : fichier `proxy.ts` à la racine (à la place du `middleware.ts`), `cookies()` et `params` désormais asynchrones (`await`).

---

## Architecture de l'app

**Deux apps Next.js séparées, reliées à la même base de données Supabase.**

- **Site e-commerce** : dossier `site/` — **en ligne** sur https://kornercash-site.vercel.app (repo `kornercashapplication-oss/kornercash-site`, branche `master`, auto-deploy). Reste : DNS `kornercash.ch`.
- **Dashboard de gestion** : dossier `dashboard/` — **en ligne** sur https://kornercash-dashboard.vercel.app (Vercel, auto-deploy à chaque push). Next 16 + React 19 + Tailwind 4 + Base UI (paquet `shadcn` `^4.12.0` pour la CLI de composants, `tw-animate-css`, `next-themes` pour le thème). **7 écrans** (Tableau de bord, Ventes, Rachats, Produits, **Atelier** [saisie en lot + jobs IA asynchrones], Clients, Module Or) + **pages imprimables (étiquettes / tickets)**. Auth à 2 mots de passe (équipe / patron) → **cookie signé HMAC**. Client Supabase **service-role** (`@supabase/supabase-js`, bypass RLS — le dashboard est l'outil interne de l'équipe). `sharp` pour le traitement d'images, `@fal-ai/client` pour la génération IA (voir section IA), `stripe` `^22.3.0` pour le **remboursement automatique** des commandes en ligne (refund du `payment_intent` via `stripe_session_id` — env `STRIPE_SECRET_KEY`, 2026-07-19), `resend` pour les **e-mails d'expédition et de remboursement** (`lib/email/`, env `RESEND_API_KEY`, 2026-07-19). Scan douchette code-barres (`scanner-dialog.tsx`, zéro dépendance) + caméra live getUserMedia (`camera-capture.tsx`). Tests via `vitest`. Reste : compléter `lib/domain/entreprise.ts` (coordonnées légales), faire valider les tickets par la fiduciaire.

**Particularité Next 16 (dashboard)** : middleware = `proxy.ts` (renommage du `middleware.ts`), comme le site.

**Lien entre les deux :** les deux apps lisent et écrivent dans la même base Supabase. Quand un employé vend un produit en magasin via le dashboard → le produit disparaît du site automatiquement. Chaque produit a aussi un **toggle « Site »** (`produits.visible_site`) piloté depuis le dashboard : le site filtre `visible_site=true` sur toutes ses requêtes catalogue + au checkout.

---

## Les 3 clients Supabase (site)

Le site instancie plusieurs clients Supabase selon le contexte d'accès, pour respecter les règles RLS.

| Fichier | Clé utilisée | Usage |
|---|---|---|
| `lib/supabase/public.ts` | anon / publishable | Lectures publiques du catalogue en SSR (catégories, sous-catégories, produits) |
| `lib/supabase/server.ts` | @supabase/ssr, session cookie | Espace client authentifié — RLS « own » (le client ne voit que ses propres commandes/données) |
| `lib/supabase/admin.ts` | service-role (**server-only**) | Server Actions : passage de commande, fiche produit, gestion des adresses, génération du lien de confirmation e-mail |
| `lib/supabase/browser.ts` | anon (navigateur) | Auth côté navigateur (connexion / inscription) |
| `proxy.ts` | — | Rafraîchit la session Supabase à chaque requête |

**Projet Supabase** : ref `nztqbfxsaduockzilrve`, région Frankfurt.

**RLS** :
- `categories` / `sous_categories` / `produits` → lecture publique
- `clients` / `commandes` / `ventes` → `authenticated`, politique « own »
- `rachats` / `transactions_or` / `produit_image_jobs` → verrouillés, accès service-key uniquement (jamais exposés au public)

---

## Direction artistique (DA actuelle)

⚠️ L'ancienne DA « luxe éditoriale » (or / blanc chaud / Archivo-Jost / zéro arrondi) **n'est plus valable**. Elle a été remplacée.

### Polices (chargées via `next/font/google` dans `app/layout.tsx`)
- **Montserrat** (`--font-display`) : titres **et** tous les sous-titres, eyebrows, labels, boutons, nav, prix.
- **Open Sans** (`--font-body`) : corps de texte (poids 400).
- Poids : labels de formulaire + boutons = **700** · liens de nav = **700** (items des menus déroulants = 400) · `.eyebrow` = **600**.

### Couleurs (tokens Tailwind v4 dans `app/globals.css`)
| Token | Hex | Rôle |
|---|---|---|
| `--color-paper` | `#fcfbf8` | Fond principal |
| `--color-ivory` | `#f7f5f0` | Fond secondaire |
| `--color-ink` | `#121110` | Texte principal |
| `--color-grey` | `#5c574e` | Texte secondaire |
| `--color-line` | `#d0cabd` | Bordures visibles |
| `--color-gold` | `#3a5a99` | **ACCENT — bleu roi feutré.** Le token s'appelle encore « gold » mais rend du BLEU ; l'or est banni |
| `--color-sky` | `#e6ecfb` | Fonds bleu clair |
| `--color-forest` | `#24503a` | CTA « Acheter » |
| `--color-placeholder` | `#ece9e1` | Placeholders |

**Grades produit** : parfait `#245c3a` / très bon `#4f9165` / correct `#86bd8f` (échelle de verts).
**Sémantiques** : success `#5b7c63` / error `#9b4b3f`.

### Formes
- `--radius: 0.75rem` → **coins arrondis partout**. La règle « zéro arrondi » est **levée** (un seul point de réglage = `--radius`).
- Ombres douces autorisées (menus déroulants, cartes).
- Boutons (`components/ui/button.tsx`) : variantes `solid` / `ghost` / `onDark`, Montserrat 700 uppercase, arrondis.

---

## Structure de l'accueil (`app/page.tsx`)

Dans l'ordre :
1. **Hero** — image bannière `/banniere-ete-upscale.jpg` pleine largeur, sans bouton (plafonnée à 60vh sur ordi).
2. **Réassurance** — 4 icônes (Contrôlé / Garanti 14j / Livraison / Retour) sur fond blanc, sous le hero.
3. **« Meilleures affaires »** — carrousel de produits (`components/product/product-carousel.tsx` : flèches ‹ › + dots, ~4,5 visibles, défilement produit par produit, mobile inclus). Placé **avant** « Trouvez votre marque ». Requête `listMeilleuresAffaires` (bons plans en priorité, sinon prix les plus bas). Bouton plein « Voir tous les bons plans » → `/bons-plans`.
4. **« Trouvez votre marque »** — grille de tuiles sections = sous-catégories (`components/category/section-tile.tsx`, config `lib/domain/sections.ts`), sans eyebrow.
5. **Avis Google** — `components/blocks/avis-google.tsx` : carrousel **manuel** des captures d'écran (`/public/avis/avis-N.png`), note figée **4,8 étoiles**, flèches + dots, fond en dégradé sky (`#e6ecfb`) → blanc. Contenu curé dans `lib/domain/avis.ts` (`AVIS_IMAGES` + `NOTE_GOOGLE`).

**La vente client (rachat / estimation) a été retirée du site public.** Le rachat se fait uniquement en magasin (dashboard).

---

## Génération d'une nouvelle tuile de section (prompt IA)

Les tuiles de l'accueil (« Trouvez votre marque ») sont des images carrées **500×500** : un objet centré sur un **fond bleu** avec une **ombre portée** dessous. Format et style identiques pour toutes les tuiles (config `site/lib/domain/sections.ts`, fichiers dans `site/public/sections/`).

Pour créer une nouvelle tuile, on utilise un modèle IA d'**édition d'images** (type `nano-banana-2/edit`, qui prend plusieurs images en entrée). Il faut lier **2 images** :

1. **Image 1** — la photo de **l'objet** que l'on veut sur la tuile.
2. **Image 2** — un **exemple de tuile déjà créée** (sert de référence : fond bleu + ombre + format 500×500).

**Prompt à utiliser (verbatim) :**

> Sur l'image 1 tu dois garder seulement l'objet et le mettre à la place de l'objet sur l'image 2.
>
> Le résultat final est donc l'objet de l'image 1 sur fond bleu de l'image 2 avec une ombre en dessous.

**Workflow de mise en ligne** (une fois l'image générée) :
1. Déposer l'image dans `assets/sous sections/` (source vault).
2. Copier vers `site/public/sections/<slug>.png` (nom de fichier sans espace/majuscule).
3. Ajouter une entrée dans `site/lib/domain/sections.ts` (`image`, `label`, `categorieSlug`, `sousCategorieSlug` — slugs exacts en DB).
4. Vérifier build (`typecheck`/`lint`/`build`) → commit → push → redeploy Vercel → vérif prod.

---

## Pages du site (les routes)

`/` · `/c/[slug]` (filtres grade / prix / recherche + `?sousCategorie=`) · `/bons-plans` · `/recherche` · `/p/[slug]` (fiche produit remaniée : `product-header.tsx` + `product-buybox.tsx` + `product-gallery.tsx` + `product-sticky-bar.tsx` + `product-defauts.tsx`, bloc « Vous aimerez aussi ») · `/connexion` · `/mot-de-passe-oublie` · `/compte` · `/compte/nouveau-mot-de-passe` · `/commande` · `/commande/merci` (retour paiement Stripe) · `/contact` · `/cgv` · `/auth/confirm` · `/api/stripe/webhook` (webhook paiement) + `sitemap` / `robots`. *(La route `/vendre` n'existe plus.)*

**Header** (`components/layout/header.tsx`) : logo à gauche, nav catégories avec **menus déroulants de sous-catégories** (`listCategoriesAvecSous`), recherche, sélecteur de devise, compte, panier.

**Espace client** (`/compte`) : e-mail en lecture seule, placeholder « ajouter un numéro » sur le téléphone, **adresses multiples** (colonne jsonb `clients.adresses`, menu déroulant + ajout / suppr via `components/compte/adresses-manager.tsx`), **panier** affiché au-dessus de « Mes commandes ». Checkout = sélecteur d'adresse enregistrée.

---

## Vérification e-mail via Resend (auto-gérée)

La confirmation d'e-mail est gérée **par le code, via Resend** — **sans** utiliser le système d'e-mails de Supabase.

1. Inscription → `admin.generateLink({ type: 'signup' })` génère le lien de confirmation.
2. Resend envoie l'e-mail (`lib/email/resend.ts` + `lib/email/templates.ts`).
3. Le compte reste **non confirmé** jusqu'au clic.
4. La route `/auth/confirm` fait `verifyOtp(token_hash)` → ouvre la session (`type=recovery` → page nouveau mot de passe).
5. La connexion est **bloquée** tant que `email_confirmed_at` est `null`.
6. **Repli** : si `RESEND_API_KEY` est absente, auto-confirmation (utile en dev).

**Variables d'env** : `RESEND_API_KEY` (posée en prod le 2026-07-19 — mode test, envois uniquement vers l'adresse du compte Resend) + `EMAIL_FROM` (défaut `onboarding@resend.dev`).
**Mot de passe oublié (2026-07-19)** : `/mot-de-passe-oublie` → `generateLink({type:'recovery'})` → e-mail Resend → `/auth/confirm?type=recovery` → `/compte/nouveau-mot-de-passe` (`auth.updateUser`).
**En prod** : il faudra **vérifier le domaine `kornercash.ch` dans Resend** avant l'envoi réel.

---

## Base de données Supabase

**9 tables** : `categories`, `sous_categories`, `produits`, `clients`, `commandes`, `ventes`, `rachats`, `transactions_or`, `produit_image_jobs`.

**Les migrations** (25 au total). Dernières : `add_visible_site_to_produits` (#22, 2026-07-03 — toggle « Site »), `add_code_passeport_clients` (#23, 2026-07-04), `add_stripe_session_id_commandes` (#24, 2026-07-09 — paiement Stripe), `add_taux_eur_commandes` (#25, 2026-07-19 — taux EUR figé par commande).

Le schéma exact des colonnes est dans **`dashboard/lib/supabase/types.ts`** (source de vérité, complet — celui du site est un sous-ensemble volontaire à 8 tables).

---

## Devise (CHF / EUR)

Prix stockés en **CHF** dans la DB. Conversion EUR via un **taux fixe provisoire** `1 CHF ≈ 1.06 €` (à brancher plus tard sur une API réelle). Logique dans `lib/domain/devise.ts` (`formatPrix`, `convertir`, `versChf`). Sélecteur manuel CHF / EUR dans le header.

---

## Pipeline d'ajout de produit (photos)

Chaque nouveau produit passe par ce pipeline avant d'apparaître sur le site :

1. **Photo brute** — photo du produit + titre.
2. **Traitement image** — fond blanc, retrait de l'étiquette de prix si visible, compression (~300–500 Ko max, 1600px).
3. **Génération** — description, catégorisation, optimisation SEO → insertion en DB.
4. **Upload** — image stockée dans le bucket Supabase Storage `produits`.
5. **Affichage** — le produit apparaît automatiquement sur le site. Next.js Image ré-optimise à l'affichage (WebP, taille adaptée à l'écran).

**Important sur les images :**
- Toujours compresser AVANT l'upload dans Supabase.
- Supabase Storage stocke l'image telle quelle — pas de compression automatique.
- Fiches produit : images en `object-contain` sur fond blanc avec padding (galerie + miniatures).
- L'ajout normal de produits se fait **via le dashboard**, pas par script one-shot. Pour le **volume** : écran Atelier (`/atelier`) — brouillons + générations IA asynchrones en lot.

---

## IA — photos & descriptions produit (dashboard)

L'IA du dashboard passe par **fal.ai** (`@fal-ai/client`). Les ids de modèles sont configurables via l'environnement (`FAL_IMAGE_MODEL` / `FAL_TEXT_MODEL` / `FAL_TEXT_LLM`), avec ces valeurs par défaut (`lib/ai/config.ts`) :

| Usage | Modèle par défaut | Rôle |
|---|---|---|
| **Image** | `fal-ai/nano-banana-2/edit` | Retouche de la photo produit (fond blanc, retrait de l'étiquette de prix) — sortie **2K** (`resolution: "2K"` + `num_images: 1` figés dans `builders.ts` ; le modèle supporte le 4K mais 2K = 1.5×/$0.12 vs 4K = 2×/$0.16) |
| **Texte** | `fal-ai/any-llm/vision` | Endpoint vision multimodal (photo + champs → description) |
| **LLM texte** | `google/gemini-2.5-flash` | Modèle appelé par `any-llm/vision` (paramètre `model`) pour rédiger la description / catégorisation |

Code : `lib/ai/fal.ts` — **images via la file fal** (`queue.submit` + polling ~4 s + `queue.cancel` pour l'annulation, non bloquant : atelier ET formulaire produit, générations parallèles depuis le 2026-07-04, sans timeout) ; description via `fal.subscribe` (timeout 45 s). Clé d'API fal via l'environnement. Erreurs des actions critiques renvoyées **en valeur** `{ok,error}` + helper `lib/client-errors.ts`.

---

## Reste à faire (dette technique / phases suivantes)

- **DNS `kornercash.ch`** : brancher le domaine sur les 2 déploiements Vercel existants (site + dashboard poussés et en ligne depuis le 2026-07-02 ; accès domaine pas encore obtenu).
- **Paiement Stripe — EN PROD mode test depuis le 2026-07-19** : testé 8/8 en local (2026-07-09, TWINT activé), env vars Vercel posées (`STRIPE_SECRET_KEY` site + dashboard, `STRIPE_WEBHOOK_SECRET`) + endpoint webhook prod créé, remboursement automatique branché côté dashboard. Reste : re-test du parcours en prod (cartes test) + clés live à la mise en service réelle.
- ~~Devise EUR~~ **FAIT (2026-07-19)** : taux réel via l'API Frankfurter (`getTauxEurChf`, cache 6 h, repli 1.06), figé par commande (`taux_eur`).
- **E-mail Resend** : vérifier le domaine `kornercash.ch` en prod.
- **Photos** : produire les photos du reste du stock (1 000–3 000 produits) — ~180 bijoux réels déjà intégrés via l'Atelier (2026-07-03).
- **E-mails transactionnels** : confirmation de commande, expédition (seule la vérification d'inscription est faite).

---

## Historique des décisions techniques

- **2026-06-27** — Stack validée : Next.js + Supabase + Vercel + Tailwind + Resend.
- **2026-06-27** — Français seul au lancement, architecture prête pour l'anglais.
- **2026-06-27** — Recherche produits : PostgreSQL full-text search (pas d'outil externe).
- **2026-06-30** — Dashboard : 6 écrans codés, responsive, étiquettes Code128, remboursements.
- **2026-07-01** — Dashboard déployé sur Vercel (`kornercash-dashboard.vercel.app`, auto-deploy).
- **2026-07-01** — Site e-commerce codé (Next 16 / React 19 / Tailwind 4 / Base UI), 3 clients Supabase, vérif e-mail via Resend.
- **2026-07-01** — DA revue : accent **bleu roi `#3a5a99`** (fin de l'or), **coins arrondis** partout (`--radius: 0.75rem`), polices Montserrat + Open Sans.
- **2026-07-01** — Vente client retirée du site public (rachat = magasin uniquement).
- **2026-07-01** — Migration `add_adresses_jsonb_clients` : adresses multiples (`clients.adresses` jsonb).
- **2026-07-02** — IA dashboard via **fal.ai** (`@fal-ai/client`) : image `fal-ai/nano-banana-2/edit` (alors 4K), texte `fal-ai/any-llm/vision` + `google/gemini-2.5-flash`. Versions de la stack alignées sur les `package.json`.
- **2026-07-02** — Refonte taxonomie (migrations #17/#18) : « Accessoires » → **Mode** (slug `mode`), **Montres** devient sous-catégorie de Mode, nouvelle section **Autres** (Figurines / Cartes Pokémon), **tag `est_luxe` supprimé** (page `/luxe`, badge, colonne DB retirés — site + dashboard redéployés).
- **2026-07-02** — Site déployé sur Vercel (`kornercash-site.vercel.app`) + **Atelier** dashboard (brouillons + jobs IA asynchrones, table `produit_image_jobs`, migrations #20/#21).
- **2026-07-03** — Toggle **`visible_site`** par produit (migration #22) : filtre site + Switch dashboard. Intégration de ~180 bijoux réels via l'Atelier.
- **2026-07-04** — IA image : **résolution 4K → 2K** (coût ÷1.33) + `num_images: 1` ; formulaire produit passé en **file fal + polling** (générations parallèles) ; **caméra live** getUserMedia ; **scan douchette** ; `clients.code_passeport` (migration #23) ; montants au centime exact (`chfExact`).
- **2026-07-09** — **Stripe Checkout** codé côté site (`stripe` `^22.3.0`, migration #24 `add_stripe_session_id_commandes`) + testé 8/8 en local (TWINT activé). PWA sur les 2 apps (manifests + icônes).
- **2026-07-19** — **Stripe EN PROD mode test** : env Vercel + endpoint webhook prod. **Remboursement automatique** côté dashboard (dep `stripe` ajoutée, `lib/stripe/server.ts`, refund du `payment_intent` avant la DB). Décisions : remboursement auto = oui, **Klarna retiré**. **Mot de passe oublié** sur le site (flux recovery + Resend, `RESEND_API_KEY` posée en prod — mode test). **E-mails transactionnels** (confirmation webhook / expédition / remboursement — jamais bloquants) + **taux EUR réel** (Frankfurter, migration #25 `taux_eur`) + **flux Expédition** dashboard (badge « À expédier » orange). **Re-test prod validé** (3 e-mails reçus + refund Stripe `succeeded`) ; fix panier bfcache/multi-onglet (resync `storage`/`pageshow`, `324cb41`).
