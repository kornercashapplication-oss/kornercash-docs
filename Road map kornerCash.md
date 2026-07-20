# ROAD MAP PROJET — KornerCash

Référence design : BackMarket (https://www.backmarket.fr/)

**État général (session 2026-07-20)** : **Phase 6 (Tests) quasi terminée** — Anatole a validé le 2026-07-20 le dashboard sur tablette/téléphone, le process de rachat complet, la synchro site/dashboard et les devises CHF/EUR. Seul le **SEO (Google Search Console)** reste à vérifier, et il dépend du branchement DNS de `kornercash.ch`.

Les deux apps sont **EN LIGNE** sur Vercel — dashboard https://kornercash-dashboard.vercel.app et site https://kornercash-site.vercel.app (repos git connectés = auto-deploy). La **feature IA photos & descriptions** (dashboard, fal.ai) est **déployée + validée en prod** (189 générations bijoux, 0 échec) ; l'**Atelier** a permis d'intégrer ~180 bijoux réels. Le **paiement Stripe Checkout est EN PROD en mode test** (testé 8/8 en local le 09-07 ; env Vercel + endpoint webhook prod posés le 19-07 ; **remboursement Stripe automatique** branché côté dashboard ; décisions tranchées : remboursement auto = OUI, Klarna = NON). **✅ RE-TEST PROD VALIDÉ par Anatole (19-07)** : achat + e-mail confirmation, expédition + e-mail, remboursement + e-mail + refund Stripe vérifié via API (`re_3TuuvG…` succeeded) — 2 correctifs au passage : panier bfcache/multi-onglet (`324cb41`), statut « À expédier » orange (`d5bfe3b`). Reste : clés live à la mise en service. **Faits le 19-07** : taux EUR réel (API Frankfurter, figé par commande) · e-mails transactionnels (confirmation / expédition / remboursement) · flux Expédition dashboard · prototype confirmé supprimé. Restent aussi : DNS `kornercash.ch` (accès domaine pas encore obtenu), vérification du domaine Resend (e-mails vers de vrais clients), photos du reste du stock, **migration Odoo** (produits + clients + fiches récap mensuelles — accès à demander au client, Phase 5bis), coordonnées légales (`entreprise.ts`) + validation fiduciaire des tickets.

---

## Fonctionnalités — Site e-commerce (kornercash.ch)

- Trouver les produits facilement (catégories, recherche, filtres par grade/prix + recherche interne à la collection)
- 3 grades qualité : Parfait état / Très bon état / État correct + défauts spécifiques avec photos (affichés sur la fiche produit)
- Tag transversal : "Bons Plans" (`est_bon_plan`, appliqué manuellement par un employé, toutes catégories). *(Le tag « Luxe » a été supprimé — migration #18.)*
- Visibilité par produit : toggle « Site » (`visible_site`) piloté depuis le dashboard — le site filtre `visible_site=true` partout
- Produits pouvant avoir plusieurs unités
- Compte client obligatoire pour acheter
- Panier multi-produits
- Livraison 5 CHF (architecture prête pour montant minimum / livraison gratuite à partir de X)
- Devises CHF/EUR (sélecteur manuel dans le header)
- Page contact : coordonnées affichées (adresse, email, téléphone) — pas de formulaire
- Remboursement / annulation : le client contacte par email ou téléphone
- CGV + moyens de paiement
- **La vente client (rachat/estimation) a été RETIRÉE du site public** — le rachat se fait uniquement en magasin (dashboard)
- **PWA** : le site s'installe sur l'écran d'accueil du téléphone (manifest + icônes, ouverture plein écran) — ✅ fait et validé le 2026-07-09

## Fonctionnalités — Dashboard (admin.kornercash.ch)

**2 niveaux d'accès :**
- Mot de passe équipe → accès aux opérations quotidiennes : produits, **atelier**, clients, rachats, **et l'écran Ventes (enregistrer + consulter l'historique)**. **Tous les montants sont masqués** (MoneyGate → « ••• »).
- Mot de passe patron → débloque en plus le **Tableau de bord**, le **module Or** et l'**affichage des montants** (chiffres réels).

> Décision audit 2026-07-01 (S1) : l'équipe conserve l'accès à `/ventes` (elle doit pouvoir encaisser au comptoir) ; la confidentialité des chiffres est assurée par le MoneyGate, pas par un blocage de page. `/` (tableau de bord) et `/or` restent patron-only.

**Fonctionnalités :**
- Historique des rachats
- Historique des ventes (accessible équipe — montants masqués par le MoneyGate ; seuls `/` et `/or` sont patron-only)
- Gestion des produits (ajout, modification, suppression, stock, toggle « Site » par produit, filtre sous-catégorie)
- Liste des clients + historique par client (avec image + nom d'article) + ajouter un rachat ou une vente à un client
- Blacklister un client
- Enregistrer un rachat : champs produit + photo pièce d'identité (caméra live → bucket privé) + « Code de la pièce » optionnel + signature sur écran tablette
- Générer une étiquette imprimable (prix, titre, code-barres, code article, date de rachat)
- **IA photos & descriptions produits** (fal.ai) : à la création/édition d'un produit, chaque photo principale passe par une IA fond blanc / **2K** (générations en parallèle via la file fal + polling, validation par job, réordonnancement, rotation) et la description est générée selon un prompt marketing e-commerce (Titre → Intro → Corps → Défauts → CTA, ton adapté à la catégorie, garde-fou anti-invention ; éditable + Régénérer)
- **Atelier de saisie en lot** (`/atelier`) : produits créés en **brouillon** + jobs d'images IA asynchrones (table `produit_image_jobs`, reprise au rechargement), validation avec comparaison photo prise ↔ image générée, « Finaliser » → `en_stock`
- **Scan douchette code-barres** (2026-07-04) : bouton « Scanner » sur la liste Produits (scan → fiche du produit) et dans Nouvelle vente (scan enchaîné type caisse, re-scan = quantité +1 plafonnée au stock). La douchette émule un clavier — composant `scanner-dialog.tsx`, aucune dépendance
- **Ventes — création rapide** : recherche client (badge Blacklisté) + création rapide d'un client (e-mail optionnel) ou d'un produit oublié sans quitter la vente ; montants au centime exact (`chfExact`)
- Remboursement (refund Stripe automatique + e-mail client)
- **Expédition d'une commande en ligne** (2026-07-19) : bouton « Marquer expédiée » + n° de suivi optionnel (`numero_suivi`) + e-mail client — les commandes en ligne payées s'affichent **« À expédier » en orange** (liste + détail ; statut DB inchangé, helper `statutAffichage()`)
- Réservation et acompte — fonctionnement à définir plus tard
- Accessible sur tablette et téléphone (responsive, pas d'app store) — **installable en PWA sur l'écran d'accueil** (manifest « KC Dashboard », plein écran, icône monogramme blanc sur bleu `#3a5a99`) — ✅ fait et validé le 2026-07-09

---

## Roadmap et état actuel

### Phase 0 : Stack technique — ✅ TERMINÉE
- [x] Fichier stack technique validé → `stack techniques.md`

### Phase 1 : Base de données — ✅ TERMINÉE
- [x] Lister toutes les tables nécessaires et leurs relations (`database architecture.md`)
- [x] Créer le schéma dans Supabase (8 tables, colonnes, clés étrangères, index, RLS activé)
- [x] Seed des données de référence (5 catégories + 15 sous-catégories)
- [x] Configurer Supabase Storage (bucket `produits` public + `pieces-identite` privé)
- [x] Policies RLS du site (lecture publique cat/produits ; clients/commandes/ventes = chacun ses données « own » ; rachats/transactions_or verrouillés service key)
- [x] Durcissement sécurité (`set_updated_at` search_path figé)
- [x] Insérer des données de test pour valider la structure (chaîne rachat → produit → vente, puis nettoyage)
- [x] Ajout module Or : table `transactions_or` (migration `create_transactions_or`, verrouillée service key, testée + nettoyée) — *2026-06-30*
- [x] Adresses multiples client : colonne `clients.adresses` (jsonb `'[]'`, migration `add_adresses_jsonb_clients`, non-bloquante pour le dashboard) — *2026-07-01*

**Total : 9 tables (categories, sous_categories, produits, clients, commandes, ventes, rachats, transactions_or, produit_image_jobs), 25 migrations.** Schéma exact des colonnes = source de vérité `dashboard/lib/supabase/types.ts` (celui du site est un sous-ensemble volontaire à 8 tables). Sous-catégories : 18 (15 au seed initial + refonte taxonomie #17).

**Projet Supabase** : ref `nztqbfxsaduockzilrve`, région Frankfurt.
**RLS** : `categories` / `sous_categories` / `produits` = lecture publique ; `clients` / `commandes` / `ventes` = authenticated « own » (chacun ses données) ; `rachats` / `transactions_or` / `produit_image_jobs` = verrouillés service key.

**Décision auth** : le dashboard est une appli serveur qui utilise la *service key* Supabase (accès total, bypass RLS), avec une porte d'entrée à 2 mots de passe (équipe/patron) gérée dans le code — ce ne sont PAS des comptes Supabase. Les clients du site utilisent Supabase Auth, protégés par les policies RLS.

### Phase 2 : Design & Branding — ✅ FONDATIONS FAITES (DA fortement itérée au Bloc C)

Assets de référence ajoutés par Anatole → `assets/` (bannière, photos produits fond blanc, icônes réassurance, tuiles sous-catégories).

> ⚠️ **La DA a beaucoup évolué au moment du codage (Bloc C).** L'ancienne DA « luxe éditoriale » (or / blanc chaud / Archivo + Jost / zéro-arrondi) **N'EST PLUS VALABLE**. La DA réelle actuelle est décrite ci-dessous (bleu, Montserrat + Open Sans, coins arrondis, carrousels). Le fichier `Systeme de design - site.md` a été mis à jour (2026-07-02) : il documente désormais la DA actuelle, avec une section « Historique » pour l'intention initiale.

**DA réelle (état du code, tokens dans `site/app/globals.css` — Tailwind v4 config inline) :**

- **Polices** (chargées via `next/font/google` dans `app/layout.tsx`) :
  - **Montserrat** (`--font-display`) = titres + TOUS les sous-titres / eyebrows / labels / boutons / nav / prix. Boutons + labels de formulaire = **700** ; `.eyebrow` = **600** ; liens de nav = **700** (items des menus déroulants = 400).
  - **Open Sans** (`--font-body`) = corps de texte (poids **400**).
- **Couleurs (tokens) :**
  - `--color-paper` **#fcfbf8** · `--color-ivory` **#f7f5f0** · `--color-placeholder` **#ece9e1** (fonds)
  - `--color-ink` **#121110** (texte principal) · `--color-grey` **#5c574e** (texte secondaire) · `--color-line` **#d0cabd** (bordures visibles)
  - `--color-gold` **#3a5a99** ⚠️ **le token s'appelle encore « gold » mais rend du BLEU roi feutré** — c'est l'ACCENT. **L'or est banni.**
  - `--color-sky` **#e6ecfb** (fonds bleu clair, hero / dégradés)
  - `--color-forest` **#24503a** (CTA « Acheter »)
  - **Grades** : parfait **#245c3a** / très bon **#4f9165** / correct **#86bd8f** (échelle de verts)
  - **Sémantiques** : success **#5b7c63** / error **#9b4b3f**
- `--radius` **0.75rem** → **COINS ARRONDIS partout** (la règle « zéro arrondi » est **LEVÉE**). Ombres douces autorisées (menus, cartes).
- **Boutons** (`components/ui/button.tsx`) : variantes `solid` / `ghost` / `onDark`, Montserrat 700 uppercase, arrondis.

**Logo** : KornerCash (+ déclinaisons claire/sombre, icône seule), généré par Anatole → `assets/logo/`.

### Phase 3 : Dashboard — ✅ TERMINÉ (EN LIGNE)

Dossier `dashboard/` — déployé sur **https://kornercash-dashboard.vercel.app** (Vercel, auto-deploy à chaque push). Stack : Next 16 + React 19 + Tailwind 4 + Base UI (shadcn base-nova). Auth = 2 mots de passe (équipe/patron) en variables d'environnement serveur.

- [x] Page login (2 mots de passe équipe/patron)
- [x] Page liste des produits + ajout/modification/suppression
- [x] Page enregistrer un rachat (formulaire + upload pièce d'identité + signature écran)
- [x] Module Or (rachat/revente d'or au gramme, par carat → `transactions_or`)
- [x] Génération d'étiquette imprimable (Code128, route `/etiquette/[id]`) + tickets
- [x] Page historique des rachats
- [x] Page historique des ventes (accessible équipe, montants masqués par le MoneyGate — seuls Tableau de bord et Or sont patron-only)
- [x] Page liste des clients + historique par client
- [x] Blacklist client
- [x] Valider une vente en magasin et la relier à un client
- [x] Remboursement (statut + remise en stock)
- [x] Responsive tablette et téléphone (nav mobile tiroir + tableaux scroll)
- [x] **IA photos & descriptions produits (fal.ai) — DÉPLOYÉE + VALIDÉE EN PROD** : module `lib/ai/` (builders purs testés vitest + `fal.ts` server-only) + server actions `lib/actions/ia.ts` + composants `PhotoIA` / `DescriptionIA` câblés dans `produit-form.tsx`. Modèle image `fal-ai/nano-banana-2/edit` (**résolution `2K`** — passée de 4K à 2K le 2026-07-04 pour le coût (1.5× vs 2×), `num_images: 1`, `1:1`, fond blanc pur, **pas de `temperature`** = fidélité par prompt + validation) ; description via `any-llm/vision` (`google/gemini-2.5-flash`, temp 0.2, prompt marketing e-commerce : Titre → Intro → Corps → Défauts → CTA, ton adapté à la catégorie, garde-fou anti-invention — pas de limite de mots). **Générations d'images en parallèle via la file fal + polling 4 s** (2026-07-04 — plus de flux bloquant ; timeout 45 s seulement sur la description), multi-photos de référence (max 6), caméra live, rotation (avant/après validation), réordonnancement. Env `FAL_KEY` + 3 modèles en Production Vercel. Détail : `Plan implémentation - IA photos et descriptions produits (dashboard).md` + `Spec - IA photos et descriptions produits (dashboard).md`.
- [x] **Atelier de saisie en lot (`/atelier`) — DÉPLOYÉ (2026-07-02)** : brouillons + jobs IA asynchrones (`produit_image_jobs`), validation au fil de l'eau, « Finaliser ». Validé en prod : 189 générations bijoux (0 échec) le 2026-07-03.
- [x] **Toggle « Site » par produit** (`visible_site`, 2026-07-03) + **filtre sous-catégorie** sur la liste Produits
- [x] **Scan douchette code-barres** (2026-07-04) — liste Produits + mode caisse dans Nouvelle vente
- [x] **Rachats : code de la pièce + photo directe de la pièce d'identité** (2026-07-04)
- [x] **Ventes : recherche + création rapide client/produit, montants au centime exact** (2026-07-03/04)

**Reste (finitions dashboard)** : compléter `lib/domain/entreprise.ts` (coordonnées légales), faire valider les tickets par la fiduciaire, supprimer le prototype `KornerCash-Interne/`.

### Phase 4 : Site e-commerce — ✅ CODÉ (Bloc C) — reste paiement + e-mails + mise en ligne

Dossier `site/`. **Stack** : Next.js **16** (App Router, fichier **`proxy.ts`** au lieu de middleware, `cookies()`/`params` async), React **19**, Tailwind CSS **4** (config inline dans `app/globals.css`), TypeScript 5, `@base-ui/react`, `@supabase/supabase-js`, `@supabase/ssr`, `resend`, `lucide-react`, `sonner`, `class-variance-authority`, `tailwind-merge`, `clsx`. Dev : `tsx`, `dotenv`.

**3 clients Supabase** :
- `lib/supabase/public.ts` — anon publishable key (lectures catalogue SSR)
- `lib/supabase/server.ts` — `@supabase/ssr`, session cookie, RLS « own » (espace client)
- `lib/supabase/admin.ts` — service-role, server-only (Server Actions commande + fiche + adresses)
- `lib/supabase/browser.ts` — auth navigateur · `proxy.ts` — refresh session

**Structure de l'accueil** (`app/page.tsx`), dans l'ordre :
1. **Hero** = image bannière `/banniere-ete-upscale.jpg` pleine largeur, sans bouton (plafonnée à 60vh sur ordi)
2. **Réassurance** = 4 icônes (Contrôlé / Garanti 14j / Livraison / Retour) sur fond blanc, sous le hero
3. **« Meilleures affaires »** = carrousel produits (`components/product/product-carousel.tsx` : flèches ‹ › + dots, ~4,5 visibles, défilement produit par produit, mobile inclus), requête `listMeilleuresAffaires` (bons plans en priorité sinon prix les plus bas), bouton plein « Voir tous les bons plans » → `/bons-plans` — placé **avant** « Trouvez votre marque »
4. **« Trouvez votre marque »** = grille de **tuiles sections** = sous-catégories (`components/category/section-tile.tsx`, config `lib/domain/sections.ts`), pas d'eyebrow
5. **Avis Google** = `components/blocks/avis-google.tsx` : carrousel manuel de **captures d'écran** (`/public/avis/avis-N.png`), note figée **4,8 étoiles**, flèches + dots, fond en dégradé sky (#e6ecfb) → blanc ; contenu curé dans `lib/domain/avis.ts` (`AVIS_IMAGES` + `NOTE_GOOGLE`)

**Les routes** : `/` · `/c/[slug]` (filtres grade/prix/recherche + `?sousCategorie=`) · `/bons-plans` · `/recherche` · `/p/[slug]` (fiche remaniée) · `/connexion` · `/mot-de-passe-oublie` · `/compte` · `/compte/nouveau-mot-de-passe` · `/commande` · `/commande/merci` (retour paiement) · `/contact` · `/cgv` · `/auth/confirm` · `/api/stripe/webhook` + `sitemap`/`robots`. *(Les routes `/luxe` et `/vendre` n'existent plus.)*

**Header** (`components/layout/header.tsx`) : logo à gauche, nav catégories avec **menus déroulants de sous-catégories** (`listCategoriesAvecSous`), recherche, sélecteur devise, compte, panier.

- [x] Setup projet Next.js + Tailwind — *dossier `site/`, 2026-07-01*
- [x] Page accueil (hero bannière, réassurance, meilleures affaires, tuiles marques, avis Google)
- [x] Page collection `/c/[slug]` (grille produits, filtres grade/prix + recherche interne)
- [x] **Fiche produit remaniée** (`components/product/` : `product-header.tsx` + `product-buybox.tsx` + `product-gallery.tsx` + `product-sticky-bar.tsx` [barre d'achat fixe mobile]) : galerie `object-contain` fond blanc, fil d'Ariane, grade, défauts, prix CHF/EUR (décimales conservées via `formatPrix`), bloc **« Vous aimerez aussi »** (`listProduitsSimilaires`)
- [x] Recherche produits (`ilike` multi-champs — FTS GIN à brancher plus tard)
- [x] Panier localStorage + tiroir, récapitulatif commande
- [x] Inscription / connexion compte client (Supabase Auth) + champ « confirmer mot de passe »
- [x] **Espace client enrichi** (`/compte`) : e-mail en **lecture seule**, placeholder « ajouter un numéro » sur téléphone, **adresses multiples** (`clients.adresses` jsonb, menu déroulant + ajout/suppr, `components/compte/adresses-manager.tsx`), **panier affiché au-dessus** de « Mes commandes ». Checkout = sélecteur d'adresse enregistrée.
- [x] **Vérification e-mail à l'inscription** — auto-gérée via **Resend** (détail ci-dessous)
- [x] Page contact (coordonnées affichées)
- [x] Page CGV (à faire valider juridiquement)
- [x] Détection devise CHF/EUR + sélecteur manuel
- [x] SEO (metadata par page, sitemap, robots, JSON-LD produit)
- [x] Tests d'intégration Supabase auto-nettoyants : **15/15** (anon, service-role, RLS lockdown, auth+own, commande). Build + tsc + eslint : **0**.
- [x] **Paiement en ligne — Stripe Checkout, ✅ EN PROD mode test (codé 2026-07-09, prod branchée 2026-07-19)** : `creerSessionPaiement` remplace le placeholder (commande `en_attente` + stock réservé CAS + session Stripe hébergée 35 min → redirection) ; webhook `/api/stripe/webhook` = source de vérité (`completed` → `payee` + `mode_paiement` carte/twint ; `expired`/`failed` → `annulee` + stock restitué, idempotent) ; `/commande/merci` (filet webhook + vidage panier) ; annulation → expiration immédiate + stock rendu. Encaissement dans la devise affichée (CHF ou EUR taux fixe ; TWINT auto en CHF seulement, aucun `payment_method_types` codé en dur). Migration #24 `add_stripe_session_id_commandes`. Env : `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`. Smoke test API mode test OK (CHF + EUR). **✅ TESTÉ 8/8 par Anatole en local (2026-07-09)** : carte 4242, annulation (stock restitué), carte refusée, 3D Secure, EUR, TWINT (activé dans le Dashboard Stripe), panier multi-produits, cohérence /compte + dashboard, remboursement (partie DB). Corrigés pendant les tests : boucle `/connexion`↔`/compte` (fiche client auto-réparée, `fe26345`), panier non vidé sur /merci (3 passes : `b4ae03a`+`a27c792` purge directe du localStorage, puis `324cb41` [19-07, re-test prod] — resync `storage`/`pageshow` dans CartProvider : une page restaurée du bfcache ou un autre onglet ressuscitait l'ancien panier mémoire), commandes annulées masquées de la liste Ventes dashboard (`bc84e42`). **Prod branchée le 2026-07-19** : `STRIPE_SECRET_KEY` posée sur Vercel (site + dashboard, sensitive) + endpoint webhook test `we_1Tuq6v1ywJcOEms7Udyg8pj9` → `https://kornercash-site.vercel.app/api/stripe/webhook` (4 events `checkout.session.*`) + `STRIPE_WEBHOOK_SECRET`. NB : le code était parti en prod dès le 09-07 (le push AGENTS.md a embarqué les commits Stripe) **sans les env** — le bouton paiement plantait entre-temps. **Re-test prod validé le 19-07** (achat + 3 e-mails + refund vérifié). Reste : clés live (Dette technique).
- [x] **E-mails transactionnels — ✅ FAITS (2026-07-19)** : confirmation de commande (envoyée par le webhook à la transition `en_attente`→`payee`, un seul envoi garanti par le CAS, filet /merci compris — récap lignes/livraison/total en devise d'encaissement + adresse) · expédition (dashboard, avec n° de suivi) · remboursement (dashboard, mention délai bancaire si refund Stripe). Envois JAMAIS bloquants (échec Resend = flux intact). ⚠️ Mode test Resend : seuls les e-mails vers `kornercashapplication@gmail.com` partent, jusqu'à la vérification du domaine.

**Vérification e-mail (auto-gérée via Resend, PAS de config Supabase)** :
Inscription → `admin.generateLink({ type: 'signup' })` → e-mail de confirmation envoyé par **Resend** (`lib/email/resend.ts` + `templates.ts`) → compte non confirmé → route `/auth/confirm` fait `verifyOtp(token_hash)` → session. La connexion est **bloquée tant que `email_confirmed_at` est null**. Repli auto-confirmation si `RESEND_API_KEY` absente. Env : `RESEND_API_KEY` + `EMAIL_FROM`.
**Mot de passe oublié (2026-07-19, commit `78dd8b1`)** : lien sur `/connexion` → `/mot-de-passe-oublie` (réponse identique que le compte existe ou non) → `generateLink({type:'recovery'})` + e-mail Resend → `/auth/confirm?type=recovery` ouvre la session → `/compte/nouveau-mot-de-passe` (`auth.updateUser`). `RESEND_API_KEY` posée en prod (mode test : envois uniquement vers `kornercashapplication@gmail.com` tant que le domaine n'est pas vérifié — conséquence : l'inscription est passée du repli auto-confirmation au flux e-mail, les inscriptions d'autres adresses échoueront d'ici la vérification du domaine).

### Phase 5 : Photos produits — 🔨 PIPELINE FAIT + PREMIER LOT RÉEL INTÉGRÉ, reste le reste du stock
- [x] **Pipeline traitement image (fond blanc + 2K)** — intégré au dashboard via l'IA fal.ai à la création produit (voir Phase 3) + Atelier pour le volume
- [x] **Pipeline description (LLM multimodal + prompt marketing)** — intégré au dashboard (feature IA)
- [x] **Premier lot réel : ~180 bijoux** (263 photos iCloud → brouillons Atelier → 189 générations IA → validés/finalisés ; 219 bijoux `en_stock` en DB au 2026-07-04) — voir `Bijoux à intégrer - catalogue.md`
- [ ] Session photo du reste du stock (entre 1 000 et 3 000 produits)
- [ ] Catégorisation + SEO en masse (à cadrer : au fil de l'eau via le dashboard ou en batch)

### Phase 5bis : Migration Odoo — ⏳ EN ATTENTE DE L'ACCÈS (ajouté 2026-07-19)
KornerCash gère aujourd'hui ses données dans **Odoo**. À migrer vers la DB Supabase avant la mise en service :
- [ ] **Accès à Odoo** — à demander au client (**bloquant** pour toute la phase)
- [ ] Migrer tous les **produits** d'Odoo → table `produits`
- [ ] Migrer tous les **clients** d'Odoo → table `clients`
- [ ] Sortir toutes les **fiches récap mensuelles** d'Odoo et les archiver (conservation — à garder même après l'abandon d'Odoo)

### Phase 6 : Tests — 🔨 QUASI TERMINÉE (reste le SEO)
- [x] Tester le parcours d'achat **jusqu'au paiement** (recherche → panier → paiement Stripe : 8/8 en local 09-07, **re-vérifié EN PROD le 19-07** : achat + e-mail de confirmation + vidage panier [corrigé `324cb41`])
- [x] Tester le dashboard sur tablette et téléphone — **validé par Anatole le 2026-07-20**
- [x] Tester le process de rachat complet (formulaire → pièce d'identité → signature → étiquette) — **validé par Anatole le 2026-07-20**
- [x] Tester la synchronisation site/dashboard (vente en magasin → produit disparaît du site) — **validé par Anatole le 2026-07-20**
- [x] Tester les devises CHF/EUR — **validé par Anatole le 2026-07-20**
- [x] Tester les e-mails transactionnels (**prod, 19-07** : confirmation + expédition + remboursement reçus — mode test Resend, vers `kornercashapplication@gmail.com`)
- [ ] Vérifier le SEO (Google Search Console) — *seul item restant de la Phase 6 ; dépend du DNS `kornercash.ch`*

### Phase 7 : Mise en ligne & livraison — 🔨 EN COURS
- [x] **Push du repo `kornercash-site`** (poussé sur `master` le 2026-07-01)
- [x] **Déployer le site sur Vercel** → **https://kornercash-site.vercel.app** (env vars Supabase en Production ; `RESEND_API_KEY` ajoutée le 2026-07-19 — `EMAIL_FROM` non posée = défaut `onboarding@resend.dev`)
- [x] Déployer le dashboard sur Vercel (**en ligne : kornercash-dashboard.vercel.app**)
- [ ] Configurer le DNS (kornercash.ch → Vercel) — *en attente de l'accès au domaine*
- [ ] Configurer Google Analytics
- [ ] Définir le transporteur et les tarifs avec le client
- [ ] Transférer les accès au client (Vercel + GitHub + Supabase)
- [ ] Former les employés à l'utilisation du dashboard

---

## Dette technique / à revoir plus tard

- [ ] **DNS `kornercash.ch` → Vercel** (Phase 7). Site + dashboard sont poussés et déployés sur Vercel ; il ne reste que le branchement du domaine (accès pas encore obtenu du client).
- [x] **Paiement Stripe — mise en prod mode test FAITE (2026-07-19)** : env Vercel posées (site + dashboard) + endpoint webhook test `we_1Tuq6v1ywJcOEms7Udyg8pj9` créé via API. NB : l'expiration auto d'une session abandonnée (stock restitué après 35 min) ne fonctionne qu'avec le webhook — désormais actif en prod.
- [ ] **Stripe — passage en LIVE (état vérifié 2026-07-19)** — **planifié : au retour d'Anatole en Suisse** (étape 1 = demander les infos légales au client). Compte `acct_1TrFC71ywJcOEms7` (CH, CHF, e-mail projet `kornercashapplication@gmail.com`) — **activation PAS commencée** (`details_submitted=false`, `charges_enabled=false`). **À fournir par le client** : forme juridique + raison sociale exacte + numéro IDE (CHE-…), identité du représentant légal (nom, date de naissance, adresse, pièce d'identité), **IBAN de l'entreprise** (versements), téléphone. **Faisable par Anatole avec ces infos** : remplir le formulaire d'activation (le compte est sous l'e-mail projet), puis bascule technique — clés live sur Vercel, endpoint webhook LIVE (nouveau `whsec_`), activer TWINT / désactiver Klarna en mode live, redeploy, paiement réel test + remboursement. À la livraison : transférer la propriété du compte au client (e-mail + accès). URL du webhook à mettre à jour quand le DNS `kornercash.ch` sera branché.
- [x] **Remboursement Stripe automatique — DÉCIDÉ OUI + FAIT (2026-07-19, commit dashboard `e87b9ae`)** : `rembourserCommande` fait le refund Stripe AVANT la DB (`stripe_session_id` → `sessions.retrieve` → `refunds.create({payment_intent})` — remboursement total, devise d'encaissement CHF/EUR gérée par Stripe). Échec Stripe = commande intacte, re-cliquable ; `charge_already_refunded` toléré (reprise d'un essai interrompu après le refund) ; garde `en_attente` (rien encaissé) ; erreurs en valeur `{ok,error}` + toasts différenciés en ligne / magasin. Dep `stripe@^22` + `lib/stripe/server.ts` + env `STRIPE_SECRET_KEY` côté dashboard. Ventes magasin (sans `stripe_session_id`) : DB seule, argent rendu à la main — inchangé.
- [x] **Klarna — DÉCIDÉ NON (2026-07-19)** : supprimé par Anatole dans le Dashboard Stripe (Settings → Payment methods). Aucun code impacté (moyens de paiement dynamiques, rien codé en dur).
- [x] **Devise EUR — ✅ FAIT (2026-07-19)** : `getTauxEurChf()` (devise-server.ts, API Frankfurter, cache 6 h, garde-fou [0.8, 1.4], repli 1.06 sur toute erreur) propagé au DeviseProvider (affichage) et au checkout (encaissement). Migration #25 `add_taux_eur_commandes` : le taux est **figé sur chaque commande EUR** → l'historique /compte et les e-mails s'affichent au taux réellement payé (1.06 pour l'existant). Taux du jour au branchement : 1.0837.
- [ ] **E-mail prod (Resend)** : vérifier le domaine `kornercash.ch` dans Resend (records DNS) puis poser `EMAIL_FROM=KornerCash <no-reply@kornercash.ch>` sur Vercel. `RESEND_API_KEY` déjà posée (2026-07-19). **Tant que le domaine n'est pas vérifié** : mode test = les mails ne partent que vers `kornercashapplication@gmail.com` (adresse du compte Resend) → inscriptions et resets d'autres adresses échoueront.
- [ ] **Photos du reste du stock** (Phase 5) — ~180 bijoux réels déjà intégrés via l'Atelier ; reste le reste des 1 000–3 000 produits.
- [x] **E-mails transactionnels — ✅ FAITS (2026-07-19)** (voir Phase 4). Dashboard : nouvelle infra `lib/email/` (copie du site) + env `RESEND_API_KEY`.
- [ ] **Section « Réservation »** (ajouté 2026-07-19) : écran/flux à coder — la DB est déjà prête (`commandes.statut='reservee'` + colonne `acompte`). Fonctionnement exact (durée, montant d'acompte, annulation) à définir avec le client.
- [ ] **Dashboard** : compléter `lib/domain/entreprise.ts` (coordonnées légales), valider tickets par fiduciaire. *(Prototype `KornerCash-Interne/` : suppression constatée le 2026-07-19 — introuvable dans le vault, jamais commité.)*
- [ ] **Renforcement de l'accès dashboard** : les 2 mots de passe partagés actuels sont **provisoires et faibles** (posés le 2026-07-09, à remplacer avant livraison). Options proposées le 2026-07-09 (comptes individuels Supabase Auth, TOTP, appareils de confiance, passkeys, Cloudflare Access, limite de tentatives…) — **aucune retenue pour l'instant, Anatole y réfléchit**.

---

## Devise

CHF/EUR (prix stockés en **CHF**), taux fixe provisoire **1 CHF ≈ 1.06 €** (à brancher sur une API réelle). Logique dans `lib/domain/devise.ts` (`formatPrix`, `convertir`, `versChf`). Filtre prix : saisi en devise affichée, converti en CHF pour l'URL (`prixMax`).

---

## Historique des décisions importantes

- **Convention rachat/vente** → "Rachat" = KornerCash rachète à un particulier. "Vente" = KornerCash vend à un client. Ne jamais utiliser "achat" (ambigu). _(2026-06-27)_
- **2 apps séparées** → Site et dashboard sont 2 apps Next.js distinctes reliées à la même DB Supabase _(2026-06-27)_
- **Français seul au lancement** → Architecture prête pour l'anglais plus tard _(2026-06-27)_
- **Tags Luxe et Bons Plans** → Ce ne sont pas des catégories mais des tags transversaux _(2026-06-27)_
- **Remboursement par email/tel** → Pas de bouton sur le site pour le lancement _(2026-06-27)_
- **Livraison 5 CHF fixe** → Structure prête pour livraison gratuite à partir de X plus tard _(2026-06-27)_
- **Modèle d'auth** → Dashboard = appli serveur avec clé maître (service key) + 2 mots de passe partagés (équipe/patron) ; site = comptes Supabase pour les clients, protégés par RLS. Les employés créent des fiches clients (sans login) pour le magasin _(2026-06-28)_
- **Module Or intégré au MVP** → KornerCash rachète/revend aussi de l'or au gramme (par carat). Table dédiée `transactions_or`, verrouillée service key, `montant` = colonne générée _(2026-06-30)_
- **Intégration dashboard — cadrage Bloc 0** _(2026-06-30, délégué)_ :
  - Le dashboard reprend le **design du prototype `KornerCash-Interne`** (épuré, accent bleu, Base UI shadcn base-nova) — recodé en Next.js + Supabase (le prototype tourne en localStorage, rien de réutilisable côté code).
  - **Catégories** : le dashboard utilise nos 5 cat + 15 sous-cat (source de vérité) ; les 8 cat maison du prototype sont abandonnées. Sous-cat « Bijoux » ajoutée sous Mode/Accessoires.
  - **KYC** : upload direct vers bucket `pieces-identite` (P2P PeerJS du prototype abandonné).
  - **Auth** : 2 mots de passe (équipe/patron) en variables d'environnement serveur.
- **Site — DA « luxe éditoriale » (initiale)** _(2026-07-01)_ : copy axé prix & marques (« Vos marques préférées à vos prix préférés »), jamais « occasion ». Nav = 5 catégories DB + Luxe/Bons Plans en collections transversales (tags). Wordmark = « KornerCash » (1 mot). ⚠️ **La DA visuelle initiale (or/blanc chaud/Archivo-Jost/zéro-arrondi) a ensuite été abandonnée au codage** (voir décision suivante).
- **Site — DA RÉVISÉE au codage (Bloc C)** _(2026-07-01)_ : passage à **accent bleu roi feutré `#3a5a99`** (l'or est banni ; le token reste nommé « gold » mais rend du bleu), **polices Montserrat (display) + Open Sans (body)**, **coins arrondis partout** (`--radius` 0.75rem, règle « zéro arrondi » levée), **carrousels** (meilleures affaires + avis Google). **`Systeme de design - site.md` a été mis à jour le 2026-07-02 pour documenter cette DA actuelle (+ règle « jamais de bordure sur les cards ») ; l'intention initiale est conservée en section « Historique ».**
- **Site — vente client retirée** _(2026-07-01)_ : plus aucune mention « vendre / estimation / rachat » côté public. Le rachat se fait uniquement en magasin (dashboard). L'e-commerce (`ventes` / commande, KornerCash vend AU client) reste intact.
- **Site — vérification e-mail via Resend** _(2026-07-01)_ : inscription confirmée par e-mail Resend (`generateLink` → `verifyOtp`), connexion bloquée tant que non confirmé. Auto-gérée, sans config Supabase.
- **Audit complet du projet** _(2026-07-01)_ : passe intégrale erreurs / incohérences / liaisons DB + tests. Corrigés : (B1) DEFAULT `statut` invalides vs CHECK (migration `fix_statut_defaults_and_index_auth_user`) ; (P1) index `clients.auth_user_id` ; (O1) `montant_rembourse` écrit au remboursement ; (O2) montants de commande figés sur leur devise d'origine dans `/compte` ; (O3/O4) décrément de stock **atomique (compare-and-swap)** anti double-vente + `Math.floor` (site + dashboard) ; nettoyage code mort (entreprise.ts site, CategoryCard, 4 fonctions catalogue, MAPPING prototype, CSS marquee, images orphelines) ; commentaires DA à jour. **Restent manuels** : activer « Leaked password protection » (Supabase Auth) ; le FTS `produits_search_idx` laissé tel quel (recherche en `ilike`). Vérif : typecheck/lint/build 0 · test:supabase 15/15 · test CAS anti-double-vente OK.
- **Dashboard — IA photos & descriptions produits (fal.ai)** _(2026-07-02)_ : feature codée en mode subagent-driven (10 tâches TDD) puis **déployée en prod**. Modèle image `fal-ai/nano-banana-2/edit` (4K natif, **sans `temperature`** → fidélité par prompt + validation avant-après, l'original n'est pas conservé) ; description via `any-llm/vision` (`google/gemini-2.5-flash`, temp 0.2), gabarit fixe 80-120 mots, éditable + Régénérer. Modèles configurables par env (`FAL_IMAGE_MODEL` / `FAL_TEXT_MODEL` / `FAL_TEXT_LLM` / `FAL_KEY`). ⚠️ Piège Vercel : l'auto-deploy git part en **BLOCKED** (auteur des commits « Anatole » pas membre Vercel) → déclencher le build via `POST /v13/deployments` gitSource avec le token. Reste : test manuel du flux en prod (consomme des crédits fal).
- **Site — mis en ligne sur Vercel** _(2026-07-02)_ : `kornercash-site` poussé + déployé → https://kornercash-site.vercel.app (framework `nextjs`, Deployment Protection off, env vars Supabase en Production). Bannière hero remplacée par `banniere-ete-upscale.jpg` (upscale 6832×2496). Reste avant kornercash.ch : DNS + Resend (domaine à vérifier).
- **Dashboard — Atelier de saisie en lot** _(2026-07-02)_ : écran `/atelier` (accès équipe) — produits créés en **brouillon** + génération d'images IA **asynchrone** (file fal, table `produit_image_jobs`, migrations #20/#21), validation au fil de l'eau, « Finaliser » → `en_stock`.
- **Bijoux réels intégrés via l'Atelier** _(2026-07-03)_ : 263 photos iCloud → ~180 brouillons Mode/Bijoux → 189 générations IA (0 échec) → validés/finalisés (219 bijoux en stock au 2026-07-04). Migration #22 `add_visible_site_to_produits` : toggle « Site » par produit (Switch dashboard, filtre `visible_site=true` sur les 5 requêtes catalogue du site + garde checkout).
- **IA image passée de 4K à 2K** _(2026-07-04)_ : `resolution: "2K"` + `num_images: 1` figés dans `builders.ts` — la résolution est le seul levier de coût du modèle (2K = 1.5×/$0.12, 4K = 2×/$0.16), 2K = meilleur rapport netteté/prix pour l'e-commerce.
- **Formulaire produit : générations IA en parallèle** _(2026-07-04)_ : le flux modale passe du `fal.subscribe` bloquant à la **file fal + polling 4 s** (comme l'Atelier) — jobs en état local du formulaire, actions `lancerPhotoIA`/`verifierPhotosIA`/`annulerPhotoIA` (les anciennes `traiterPhotoIA`/`genererPhotoProduit` sont supprimées). Caméra live getUserMedia (`CameraCapture`/`RefUploader`), erreurs server-action en valeur `{ok,error}` + helper `client-errors.ts`.
- **Rachats + Ventes + scan** _(2026-07-04)_ : « Code de la pièce » (`clients.code_passeport`, migration #23) + photo directe de la pièce d'identité (caméra live → bucket privé) ; scan douchette (Produits + mode caisse) ; recherche/création rapide client et produit dans Nouvelle vente ; montants au centime exact (`chfExact` via `MoneyValue`).
- **PWA sur les 2 apps** _(2026-07-09, ✅ validé par Anatole)_ : manifests (`app/manifest.ts`, plein écran `standalone`) + icônes 192/512/maskable — site = monogramme noir sur ivoire (icône existante), dashboard = **nouvelle icône monogramme blanc sur bleu `#3a5a99`** (il n'avait aucun logo, favicon Next par défaut remplacé). Installation : Android « Ajouter à l'écran d'accueil » / iOS Safari Partager → « Sur l'écran d'accueil ».
- **Mots de passe dashboard provisoires** _(2026-07-09)_ : équipe + patron changés à la demande d'Anatole (valeurs volontairement simples, à remplacer avant livraison). Sécurité d'accès renforcée = options présentées, **décision en attente** (voir Dette technique).
- **Paiement en ligne = Stripe Checkout (page hébergée)** _(2026-07-09)_ : choisi contre Payrexx/Wallee/Datatrans (comparatif frais + intégration + passation — Stripe : carte 2,9 % + 0.30, TWINT 1,9 % + 0.30, 0 abonnement, compte self-service au nom du client). **Encaissement dans la devise affichée** (CHF ou EUR au taux fixe du site — décision Anatole ; TWINT n'apparaît qu'en CHF, géré dynamiquement par Stripe). `commandes.total` reste en CHF (unité de compte du dashboard). Le webhook signé est la seule source de vérité du passage `payee` — fin du placeholder. Clés de TEST en local (`site/.env.local`, gitignoré) ; plugin Stripe + MCP `mcp.stripe.com` installés côté Claude.
- **Paiement testé 8/8 en local + 3 correctifs** _(2026-07-09)_ : parcours complet validé par Anatole (4242, annulation→restock, refusée, 3DS, EUR, TWINT, multi-produits, cohérence, remboursement DB). Correctifs : **fiche client auto-réparée** par `getMaFiche` (fin de la boucle `/connexion`↔`/compte` héritée du reset de données du 03-07) ; **panier vidé fiablement** sur `/commande/merci` (purge directe `localStorage` avant hydratation + `vider()` après — les effets enfants courent avant ceux du provider) ; **liste Ventes dashboard sans les `annulee`** (bruit des paiements abandonnés, données conservées en base). Ouverts : remboursement Stripe auto + Klarna (voir Dette technique).

---

## État de l'infrastructure (pour la passation client)

**GitHub** — compte dédié `kornercashapplication-oss` (séparé du perso, pour faciliter le transfert)
- Repo `kornercash-dashboard` (privé) — poussé sur `main` (déploiement Vercel auto)
- Repo `kornercash-site` (privé) — **poussé** sur `master` + projet Vercel créé et déployé. Reste à faire : DNS `kornercash.ch` (Phase 7)

**Supabase** — projet `kornercash` (région Frankfurt)
- Ref : `nztqbfxsaduockzilrve` · URL : `https://nztqbfxsaduockzilrve.supabase.co`
- ✅ 9 tables + RLS + policies + 2 buckets Storage + seed catégories — 25 migrations (dernières : `add_code_passeport_clients` 2026-07-04, `add_stripe_session_id_commandes` 2026-07-09, `add_taux_eur_commandes` 2026-07-19)

**Vercel** (org `trusttheprocess`, plan Hobby)
- Dashboard : **en ligne** → https://kornercash-dashboard.vercel.app (auto-deploy) — env `FAL_KEY` + 3 modèles `FAL_*` en Production (feature IA)
- Site : **EN LIGNE** → https://kornercash-site.vercel.app (projet `kornercash-site`, repo git connecté = auto-deploy, framework `nextjs`, Deployment Protection désactivée, 3 env vars Supabase en Production). Reste : DNS `kornercash.ch` (accès domaine pas encore obtenu) + branchement Resend quand le domaine sera vérifié.

**Domaine** — `kornercash.ch` (propriété du client, accès pas encore transmis à Anatole)

> À la livraison : transférer le compte GitHub, le projet Supabase et les projets Vercel au client.
