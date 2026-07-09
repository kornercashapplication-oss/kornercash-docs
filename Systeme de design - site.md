# SYSTÈME DE DESIGN — Site KornerCash

Référence unique de la direction artistique du site e-commerce KornerCash. **Ce document décrit l'état RÉEL du code** (dossier `site/`, à jour au 2026-07-04), pas une maquette. La source de vérité absolue reste `site/app/globals.css` (tokens) et les composants ; ce fichier les documente.

> Contexte métier → `contexte kornerCash.md` · Schéma DB → `database architecture.md` · Stack → `stack techniques.md` · Road map → `Road map kornerCash.md` · Suivi projet → `[[project_kornercash]]`

> ⚠️ **La DA « luxe éditoriale » d'origine est OBSOLÈTE.** L'ancien parti pris (accent or `#A8894B`, blanc chaud, polices Archivo/Jost, **zéro arrondi, zéro ombre**, majuscules très espacées) N'EST PLUS VALABLE. La DA actuelle est plus moderne et chaleureuse : **accent bleu roi feutré, coins arrondis partout, ombres douces autorisées, polices Montserrat/Open Sans, et jamais de bordure autour d'une card/tuile** (délimitation par fond ou ombre). Tout ce qui suit reflète le code en place.

---

## 0. Cadre & décisions

| Sujet | Décision |
|---|---|
| **Direction artistique** | Moderne et rassurante : fonds ivoire chaud, **accent bleu roi feutré unique**, **coins arrondis** (`--radius` 0.75rem), ombres douces autorisées (cartes, flèches de carrousel). L'or a été **banni**. |
| **Ton du copy** | **Axé prix & marques** — on vend l'accès aux **grandes marques à petits prix**, jamais « l'occasion ». Fil rouge : *« Vos marques préférées, à vos prix préférés. »* Contrôle & garantie = réassurance, pas argument principal. Le mot « occasion / seconde main » reste banni du marketing (OK en CGV). |
| **Navigation** | Header à **menus déroulants** : chaque catégorie DB ouvre la liste de ses sous-catégories (`listCategoriesAvecSous`). « Bons Plans » = **collection transversale** (tag `est_bon_plan`), pas une catégorie. *(La collection « Luxe » a été retirée — tag `est_luxe` supprimé.)* |
| **Vente client** | **Retirée du site public** (rachat/estimation = magasin uniquement). Plus aucune page `/vendre` active dans le parcours ni mention « vendre/estimation ». |
| **Stack** | **Next.js 16 + Supabase + Vercel.** Tailwind CSS 4 en config inline (`@theme` dans `globals.css`). |
| **Données** | Le site lit Supabase en lecture publique (RLS). Panier en `localStorage`. Adresses client en colonne `jsonb`. |
| **Wordmark** | **« KornerCash »** (1 mot). |

---

# BLOC A — IDENTITÉ

## 1. Direction artistique

**Ambiance** : boutique en ligne moderne, propre et rassurante. Objets présentés sur fond blanc (photos `object-contain`), respiration, accent bleu discret comme fil conducteur.

**Principes visuels actuels :**
1. **Fond chaud, pas de blanc pur froid** — fonds ivoire/paper (`#fcfbf8` / `#f7f5f0`) ; les visuels produit sont sur fond **blanc** (`bg-white`) pour faire ressortir l'objet.
2. **Un seul accent : le bleu roi feutré** `#3a5a99` (token nommé `--color-gold` pour raisons historiques, mais rend du **bleu**). Aucune autre couleur vive hors grades et sémantiques.
3. **Coins arrondis partout** — piloté par un token unique `--radius: 0.75rem`. Un seul point de réglage.
4. **Ombres douces autorisées** — flèches de carrousel, menus déroulants (léger `box-shadow`).
5. **⚠️ Règle CARDS : jamais de bordure sur une card / tuile / boîte de contenu** *(confirmée avec Anatole 2026-07-02, inscrite en tête de `globals.css`)*. Une carte/tuile se délimite par un **fond** (`bg-ivory` / `bg-white` / `bg-sky`) **ou une ombre**, jamais par un trait. Le token `border-line` reste OK pour les **séparateurs** (`border-t`), les **champs de formulaire** et les **boutons** — mais pas pour cerner un bloc de contenu.
6. **Majuscules espacées, mais mesurées** — boutons, eyebrows et nav en `UPPERCASE` (letter-spacing ~.2–.26em), sans excès.
7. **De l'air** — sections aérées (padding vertical 3–6rem selon la section), titres `clamp()`.

**À éviter** : le retour de l'or, les bordures autour des cards/tuiles, les dégradés criards, les couleurs vives multiples, les emojis décoratifs.

---

## 2. Palette de couleurs

Tokens définis dans `site/app/globals.css`, bloc `@theme` (Tailwind v4 → utilisables en classes `bg-*`, `text-*`, `border-*`). Valeurs **copiées du code**.

### 2.1 Couleurs de base

| Token | Hex | Rôle |
|---|---|---|
| `--color-paper` | `#fcfbf8` | Fond principal (ivoire chaud) |
| `--color-ivory` | `#f7f5f0` | Sections alternées / blocs clairs |
| `--color-ink` | `#121110` | Texte principal, blocs sombres, boutons pleins |
| `--color-grey` | `#5c574e` | Texte secondaire, méta (contraste renforcé, AA) |
| `--color-line` | `#d0cabd` | Bordures fines 1px (assombri = cartes/inputs visibles) |
| `--color-gold` | `#3a5a99` | **Accent UNIQUE = bleu roi feutré** (token nommé « gold » mais bleu ; l'or est banni) |
| `--color-sky` | `#e6ecfb` | Teinte bleu clair (fonds, bandeaux, dégradé section avis) |
| `--color-forest` | `#24503a` | Vert sapin foncé — CTA « Acheter » |
| `--color-placeholder` | `#ece9e1` | Emplacements photo (avant image) |

### 2.2 Couleurs sémantiques

| Token | Hex | Usage |
|---|---|---|
| `--color-success` | `#5b7c63` | Confirmation (commande passée, paiement OK) |
| `--color-error` | `#9b4b3f` | Erreur de formulaire (brique sourde) |

### 2.3 Couleurs des 3 grades qualité

Échelle de **verts** (du plus foncé = meilleur au plus clair). Affichés en pastille discrète (`GradePastille`).

| Grade (DB `produits.grade`) | Token | Hex |
|---|---|---|
| **Parfait état** (meilleur) | `--color-grade-parfait` | `#245c3a` (vert foncé) |
| **Très bon état** | `--color-grade-tresbon` | `#4f9165` (vert moyen) |
| **État correct** | `--color-grade-correct` | `#86bd8f` (vert clair) |

### 2.4 Tags transversaux (badges d'angle sur la carte)

| Tag (DB) | Rendu badge (code `product-card.tsx`) |
|---|---|
| `est_bon_plan` = true | **« Bon plan »** — texte `ivory` sur fond `ink`, uppercase, coin haut-droite, arrondi. |

*(Le badge « ✦ Luxe » et le tag `est_luxe` ont été supprimés — migration #18.)*

> Un produit sans tag n'a aucun badge → carte épurée.

---

## 3. Typographie

Deux polices Google Fonts chargées via `next/font/google` dans `app/layout.tsx` (variables CSS `--font-montserrat` / `--font-opensans`, mappées vers `--font-display` / `--font-body` dans `@theme`).

| Police | Variable | Poids chargés | Rôle |
|---|---|---|---|
| **Montserrat** | `--font-display` | 400 / 500 / 600 / 700 | **Titres (h1–h3)**, wordmark, eyebrows/labels, nav, boutons, prix, numéros |
| **Open Sans** | `--font-body` | 300 / 400 / 500 / 600 / 700 | **Corps de texte** (poids 400 par défaut) |

**Règles de poids (vérifiées dans le code) :**
- **Corps** (`body`) : Open Sans **400** (relevé de 300, jugé trop fin).
- **Titres** `h1/h2/h3` : Montserrat **600**, `letter-spacing: -0.02em`.
- **Eyebrow** (`.eyebrow`) : Montserrat **600**, `0.68rem`, `letter-spacing: 0.26em`, UPPERCASE. Variante `.eyebrow--gold` = couleur accent bleu.
- **Boutons** : Montserrat **700** (bold), UPPERCASE, `tracking-[0.22em]`.
- **Nav** : liens Montserrat **700** (bold) — le conteneur est `font-medium` mais tous les liens (catégories, « Bons plans », nav mobile) sont `font-bold` ; items des menus déroulants = **400**.
- **Labels de formulaire** : **700**.

> Signature actuelle : **tout ce qui est titre/label/prix/nav/bouton est en Montserrat** ; seul le corps de paragraphe est en Open Sans. (Ancienne règle « Jost 300 pour les paragraphes » supprimée.)

### Échelle typographique (exemples relevés dans le code)

| Élément | Police | Taille | Poids |
|---|---|---|---|
| Titre section (h2, ex. avis) | Montserrat | `clamp(1.8rem, 3.2vw, 2.5rem)` | 600 |
| Nom produit (carte) | Montserrat | `0.98rem` | 600 |
| Prix (carte) | Montserrat | `0.92rem` (`tracking-[0.04em]`) | — |
| Eyebrow / label | Montserrat | `0.68rem` | 600 |
| Corps / description | Open Sans | ~`0.95–1rem` | 400 |

---

# BLOC B — TOKENS & COMPOSANTS

## 4. Tokens de système

### 4.1 Layout

| Token | Valeur |
|---|---|
| Conteneur `.wrap` | `max-width: 1280px`, centré, `padding-inline: 28px` |
| Grille sections (accueil) | 2 col mobile → 3 (`sm`) → **5 col** (`lg`), gap `1rem` |
| Carrousel produits | ~4,5 cartes visibles en desktop (`lg:w-[calc((100%-4rem)/4.5)]`), 2 en mobile |
| Carrousel avis | 3,5 captures visibles en desktop, 1 en mobile |

### 4.2 Rayons / bordures / ombres

| Token | Valeur |
|---|---|
| **`--radius`** | **`0.75rem`** — appliqué partout via `rounded-[var(--radius)]` + règle globale sur `button/input/select/textarea` |
| **Bordure** | `border-color` global = `var(--color-line)` (règle `* { border-color: … }`), **mais la bordure n'est jamais dessinée sur une card/tuile** (règle §1.5). Usage réel : séparateurs (`border-t`/`border-y`), champs de formulaire, boutons, box du menu déroulant du header. |
| **Ombres** | **autorisées, douces** — ex. flèches de carrousel, box du menu déroulant `shadow-[0_12px_30px_rgba(18,17,16,0.10)]`. C'est le fond ou l'ombre qui délimite une card, pas un trait. |
| **Focus visible** | `outline: 2px solid var(--color-gold); outline-offset: 3px` (liseré bleu) |
| **Sélection texte** | fond `ink`, texte `ivory` |

### 4.3 Images

| Emplacement | Traitement |
|---|---|
| Carte produit | ratio `3/3.4`, **`object-contain` sur fond blanc + padding** (`p-4`), zoom `scale-1.03` au hover |
| Tuile section (accueil) | carré (`aspect-square`), `object-cover`, zoom `scale-1.04` au hover, libellé **intégré dans l'image** |
| Capture avis | hauteur fixe `210px`, `object-contain` sur fond blanc |
| Hero | image bannière pleine largeur `/banniere-ete-upscale.jpg`. Mobile/tablette : image entière (`h-auto w-full`). **Ordi (`lg`) : hauteur plafonnée `lg:h-[60vh]` + `object-cover` recadré vers le haut (`object-[center_35%]`).** |

### 4.4 Motion

| Interaction | Transition |
|---|---|
| Hover liens / boutons | `transition-colors duration-300` |
| Zoom image au hover | `duration-[600ms] ease-out`, `scale-1.03`/`1.04` |
| Apparition (`.fade-up`) | fade + `translateY(8px)`, `0.45s ease` |
| Carrousels | scroll natif `scroll-smooth` + snap |

---

## 5. Composants UI

### 5.1 Boutons (`components/ui/button.tsx`)

`class-variance-authority`. Base commune : Montserrat **700**, UPPERCASE, `tracking-[0.22em]`, bordure 1px, **arrondi** `rounded-[var(--radius)]`, `transition-colors duration-300`, focus bleu.

| Variante | Style |
|---|---|
| `solid` (défaut) | Fond `ink`, texte `ivory`. Hover → transparent, texte `ink`. |
| `accent` | Fond `gold` (bleu), texte `paper`. Hover → transparent, texte bleu. |
| `cta` | Fond `#24503a` (vert forest), texte `paper`. Hover → transparent, texte vert. Usage « Acheter ». |
| `ghost` | Transparent, bordure + texte `ink`. Hover → fond `ink`, texte `ivory`. |
| `onDark` | Sur fond sombre : bordure + texte `ivory`. Hover → fond `ivory`, texte `ink`. |

Tailles : `default` (`px-38 py-4`, `0.74rem`) · `sm` (`0.68rem`) · `full` (pleine largeur).

### 5.2 Carte produit (`components/product/product-card.tsx`)

Cœur du design. Anatomie : visuel (ratio 3/3.4, fond blanc, `object-contain`) + badges d'angle conditionnels + infos **centrées**.

| Élément | Source DB / rendu |
|---|---|
| Photo | `produits.photos[0]` (fallback « Photo à venir ») |
| Badge Bon plan (haut-droite) | si `est_bon_plan` |
| Titre | `produits.titre_fr` (Montserrat 600, centré) |
| Grade | pastille `GradePastille` (§2.3) |
| Prix | `produits.prix` → composant `Prix` (format devise, §6) |
| Lien | `/p/{slug_fr}` |

### 5.3 Carrousel produits (`components/product/product-carousel.tsx`)

Carrousel horizontal, **défilement produit par produit** : flèches `‹ ›` (rondes, ombre douce, superposées) + **dots** de pagination. Scroll natif snap. ~4,5 cartes visibles en desktop, 2 en mobile (fonctionne au tactile). Message si liste vide.

### 5.4 Tuile de section (`components/category/section-tile.tsx`)

Tuile carrée image (`object-cover`) avec le **libellé intégré dans l'image** (pas de texte superposé en HTML). Config dans `lib/domain/sections.ts` : **17 tuiles** = sous-catégories (dont Bijoux, Cartes Pokémon et Figurines ajoutées le 2026-07-03). Chaque tuile pointe vers `/c/{categorieSlug}?sousCategorie={slug}` (ou `/c/{categorieSlug}` seul). L'ordre des catégories de la nav est piloté en DB (`categories.ordre`, migration #19 : Mode · Multimédia · Téléphone · Gaming · Autres).

### 5.5 Section Avis Google (`components/blocks/avis-google.tsx`)

Preuve sociale. **Carrousel manuel** (flèches rondes `‹ ›` + dots) de **captures d'écran** d'avis (`/public/avis/avis-N.png`). Titre « Ils nous font confiance » + note figée **4,8** étoiles (jaune `#fbbc04`) + « · Avis Google ». Fond en **dégradé** `sky (#e6ecfb) → paper`. Contenu curé dans `lib/domain/avis.ts` (`AVIS_IMAGES` + `NOTE_GOOGLE`).

### 5.6 Header + navigation (`components/layout/header.tsx`)

Logo à gauche · **nav catégories à menus déroulants** (chaque catégorie ouvre ses sous-catégories, données `listCategoriesAvecSous`) · lien « Bons plans » · sélecteur de devise · compte · panier. Plus de lien « Vendre ». *(Pas de recherche dans le header : la recherche vit dans les filtres des pages collection — `components/collection/collection-filters.tsx` — et la route `/recherche`.)*

### 5.7 Fiche produit (PDP)

Composants dédiés : `product-header.tsx` · `product-buybox.tsx` · `product-gallery.tsx` · `product-sticky-bar.tsx` (barre d'achat fixe mobile) · `product-defauts.tsx` (liste + photos des défauts, colonne `produits.defauts` jsonb) · `product-actions.tsx` · `product-grid.tsx`. Galerie image principale + miniatures (fond blanc, `object-contain`). Fil d'Ariane, description, specs, défauts, section **« Vous aimerez aussi »** (produits similaires même catégorie), CTA « Ajouter au panier ». Prix via §6. Un produit `visible_site=false` ou hors `en_stock` → **404**.

### 5.8 Espace client (`/compte`) & panier

Conçu comme un **tableau de bord**. Signature visuelle = un **bandeau d'identité** en haut : fond `bg-sky` arrondi (pas de bordure), **avatar rond aux initiales** (`bg-gold`, `initiales(prenom, nom)`), eyebrow « Espace client », « Bonjour {prénom} », méta e-mail · membre depuis {année} · nb commandes, et « Se déconnecter » discret à droite.

- E-mail en **lecture seule** ; placeholder « ajouter un numéro » sur le téléphone.
- **Adresses multiples** : colonne `jsonb` `clients.adresses`, gestion via `components/compte/adresses-manager.tsx` (menu déroulant + ajout/suppression).
- **Rappel de panier discret** (`RappelPanier`) affiché **au-dessus** de « Mes commandes ».
- **Mes commandes** : lignes compactes `bg-ivory` arrondies (pas de bordure), détail **repliable** (`<details>` + `ChevronDown` qui pivote) ; pastille de statut colorée (`payee`/`expediee` = bleu `gold`, `livree` = `success`, `annulee` = `error`, autres = `grey`). Devise **figée** à celle de la commande.
- Checkout : sélecteur d'adresse enregistrée.

### 5.9 Formulaires

Même système (champs arrondis `--radius`, bordure 1px `--line`, focus bleu, labels Montserrat 700). Pages : `/connexion`, `/compte`, `/contact` (coordonnées, pas de formulaire), `/cgv`.

---

## 6. Devises (CHF / EUR) — `lib/domain/devise.ts`

- Prix stockés en **CHF** (`produits.prix`).
- Taux **fixe provisoire** : `TAUX_CHF_VERS_EUR = 1.06` (1 CHF ≈ 1.06 €, EUR/CHF ≈ 0.94) — à brancher sur l'API réelle en Phase suivante.
- `formatPrix` : prix rond → suffixe `.–` ; sinon 2 décimales conservées. CHF → `CHF 3'290.–` / `CHF 14.90` (apostrophe suisse) · EUR → `3 290 €` / `15.79 €` (espace fine).
- `convertir` (arrondi entier, filtres prix) · `versChf` (saisie → stockage).
- Résolution devise : cookie `kc_devise`, sinon pays (CH→CHF, FR→EUR, défaut CHF) + sélecteur manuel dans le header.

---

## 7. Structure de l'accueil (`app/page.tsx`)

Ordre **exact** des sections rendues :

1. **Hero** — image bannière `/banniere-ete-upscale.jpg` pleine largeur, **sans bouton** (`fade-up`). Plafonnée à **60vh sur ordi** (`lg:h-[60vh] lg:object-cover lg:object-[center_35%]`) ; image entière en mobile/tablette.
2. **Bande de réassurance** — 4 icônes sur fond **blanc**, bordures haut/bas : **Contrôlé** (« Chaque article vérifié. ») · **Garanti** (« Garantie 14 jours. ») · **Livraison** (« En Suisse & en France. ») · **Retour** (« Sur simple demande. »). Icônes `/icons/icon-*.png`.
3. **« Meilleures affaires »** — eyebrow « Petits prix » + titre centré, puis **carrousel produits** (`listMeilleuresAffaires(12)` : bons plans en priorité, sinon prix les plus bas). Bouton plein « Voir tous les bons plans » → `/bons-plans`. Placée **AVANT** « Trouvez votre marque ».
4. **« Trouvez votre marque »** — titre centré (pas d'eyebrow) + grille des **tuiles sections** (`SECTIONS`).
5. **Avis Google** — section carrousel `AvisGoogle` (§5.5).

---

## 8. Inventaire des pages → routes Next.js

| Page | Route |
|---|---|
| Accueil | `/` |
| Collection / catégorie | `/c/[slug]` (filtres grade / prix / recherche + `?sousCategorie=`) |
| Collection Bons Plans | `/bons-plans` (tag `est_bon_plan`) |
| Recherche | `/recherche` |
| Fiche produit | `/p/[slug]` (404 si `visible_site=false` ou hors `en_stock`) |
| Confirmation e-mail | `/auth/confirm` (route handler `verifyOtp`) |
| Connexion / inscription | `/connexion` |
| Espace client | `/compte` |
| Commande / paiement | `/commande` (paiement placeholder, statut `payee` sans encaissement) |
| Contact | `/contact` |
| CGV | `/cgv` |

+ `sitemap` / `robots`. Panier = tiroir global (pas une route). *(La route `/vendre` a été **supprimée du code** — plus aucun dossier `app/vendre/`.)*

---

## 9. Architecture technique du site

- **Stack** : Next.js **16** (App Router, `proxy.ts` au lieu de `middleware`, `cookies()`/`params` async), React **19**, Tailwind CSS **4** (config inline `@theme`), TypeScript 5. UI : `@base-ui/react`, `lucide-react`, `sonner`, `class-variance-authority`, `tailwind-merge`, `clsx`. E-mails : `resend`.
- **Supabase** (`@supabase/supabase-js`, `@supabase/ssr`). Projet ref `nztqbfxsaduockzilrve`, région Frankfurt. **3 clients** : `lib/supabase/public.ts` (anon, lectures catalogue SSR), `server.ts` (session cookie, RLS « own »), `admin.ts` (service-role, server-only, Server Actions). + `browser.ts` (auth navigateur) et `proxy.ts` (refresh session).
- **RLS** : `categories` / `sous_categories` / `produits` = lecture publique ; `clients` / `commandes` / `ventes` = authenticated « own » ; `rachats` / `transactions_or` / `produit_image_jobs` = verrouillés service-key.
- **Filtrage catalogue** : toutes les requêtes produits du site (`lib/queries/catalogue.ts`) filtrent `statut='en_stock'` **ET** `visible_site=true` (+ exclusion du produit système « Autres ») ; même garde au checkout (`lib/actions/commande.ts`).
- **Vérification e-mail via Resend** (pas de config Supabase) : inscription = `admin.generateLink({type:'signup'})` → e-mail Resend (`lib/email/resend.ts` + `templates.ts`) → route `/auth/confirm` fait `verifyOtp(token_hash)` → session. Connexion bloquée si `email_confirmed_at` null. Repli auto-confirmation si `RESEND_API_KEY` absente. Env : `RESEND_API_KEY` + `EMAIL_FROM`.
- **Git / déploiement** : repo `kornercashapplication-oss/kornercash-site` (branche `master`), **en ligne** sur https://kornercash-site.vercel.app (auto-deploy à chaque push).

---

## 10. Base de données & schéma

- **9 tables** : `categories`, `sous_categories`, `produits`, `clients`, `commandes`, `ventes`, `rachats`, `transactions_or`, `produit_image_jobs` (jobs IA de l'Atelier dashboard, service-role only).
- **23 migrations**. Dernières : `add_statut_brouillon_produits` + `create_produit_image_jobs` (2026-07-02), `add_visible_site_to_produits` (2026-07-03), `add_code_passeport_clients` (2026-07-04).
- **Schéma exact des colonnes** → `dashboard/lib/supabase/types.ts` (source de vérité, complet). Ne jamais deviner un nom de colonne : le lire ici. *(`site/lib/supabase/types.ts` = sous-ensemble volontaire à 8 tables, sans `produit_image_jobs` ni `code_passeport`.)*

---

## 11. Dashboard interne (contexte)

Le back-office (dossier `dashboard/`) est **en ligne** : `https://kornercash-dashboard.vercel.app` (Vercel, auto-deploy). **7 écrans** (Tableau de bord, Ventes, Rachats, Produits, **Atelier** [saisie en lot + génération d'images IA asynchrone, table `produit_image_jobs`], Clients, Module Or) + pages imprimables (tickets/étiquettes). Next 16 + React 19 + Tailwind 4 + Base UI. Auth à 2 mots de passe (équipe / patron). **L'ajout de produits se fait via le dashboard**, pas par script. Le dashboard pilote aussi la **visibilité site** de chaque produit (toggle `visible_site`). Reste : compléter `lib/domain/entreprise.ts` (coordonnées légales), valider les tickets par la fiduciaire.

---

## 12. Contenu de référence (coordonnées)

| Champ | Valeur |
|---|---|
| Adresse | Rue de l'Ale 28, 1003 Lausanne |
| Téléphone | **à obtenir du client** (le numéro de la maquette d'origine est faux) |
| Email | Kornercash@gmail.com |
| Retrait | Gratuit en boutique |
| Livraison | Suisse + France |

---

## 13. Reste à faire (dette technique / phases suivantes)

- **DNS `kornercash.ch`** : brancher le domaine sur le déploiement Vercel existant (repo poussé + site en ligne depuis le 2026-07-02).
- **Paiement réel** (Stripe ou équivalent) — actuellement commande placeholder (`payee` sans encaissement).
- **Devise EUR** : brancher un **taux réel** (remplacer le 1.06 fixe).
- **Resend** : vérifier le domaine `kornercash.ch` en prod.
- **Photos** : reste du stock (1 000–3 000 produits) — ~180 bijoux réels déjà intégrés via l'Atelier IA du dashboard.
- **E-mails transactionnels** : confirmation de commande, expédition.

---

*Ce document est la référence de style unique. Toute décision de style part d'ici ; il doit être mis à jour à chaque évolution notable de la DA du site.*
