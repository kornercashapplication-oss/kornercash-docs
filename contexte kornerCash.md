# CONTEXTE PROJET — KornerCash

## Ce que tu dois savoir en premier

Ce document est un contexte opérationnel destiné à un LLM. Il décrit l'entreprise KornerCash et ce que je dois créer pour elle : un site web e-commerce + un dashboard de gestion, reliés à une base de données commune.

Anatole est prestataire technique — il construit la structure et les process. Il ne fait pas le marketing ni l'acquisition de trafic.

---

## KornerCash c'est quoi ?

Magasin d'achat-revente (style EasyCash) à Lausanne. Ils rachètent et revendent tout type de produits de seconde main. Entre **1 000 et 3 000 produits** en stock actuellement.

**Convention de vocabulaire (important) :**
- **Rachat** = KornerCash rachète un objet à un particulier → toujours en magasin uniquement, jamais en ligne
- **Vente** = KornerCash vend un produit à un client → en magasin + en ligne (le site doit permettre la vente e-commerce)

- **Expédition** : Suisse + France
- **Site internet** : en ligne depuis le 2026-07-02 (site + dashboard déployés, voir ÉTAT TECHNIQUE plus bas)

**Coordonnées :**

- Adresse : Rue de l'Ale 28, 1003 Lausanne
- Mail : Kornercash@gmail.com
- Domaine : kornercash.ch (acheté, appartient à l'entreprise — Anatole n'y a pas encore accès)

---

## Catégories de produits

**5 catégories :**

| Catégorie                 | Sous-catégories                                                   |
| ------------------------- | ----------------------------------------------------------------- |
| **Gaming**                | Consoles (marque/modèle), accessoires, jeux                       |
| **Téléphone**             | Téléphones (par marque), accessoires                              |
| **Mode**                  | Montres, sacs, portefeuilles/porte-cartes, ceintures, chaussures, lunettes, bijoux |
| **Multimédia**            | Ordinateurs, tablettes, appareils photo, divers                   |
| **Autres**                | Figurines, cartes Pokémon                                         |

> Refonte 2026-07-02 (migration #17) : « Accessoires » renommée **Mode** (slug `mode`), **Montres** rétrogradée en sous-catégorie de Mode, création de la section **Autres**.

**Bons Plans** n'est **PAS une catégorie** → c'est un **tag transversal** (`est_bon_plan`), applicable à n'importe quel produit. *(Le tag `est_luxe` / la collection « Luxe » ont été supprimés — migration #18.)* Voir `database architecture.md`.

---

## Activité Or

En plus des produits de seconde main, KornerCash **rachète et revend de l'or au gramme** (par carat : 24k / 22k / 18k / 14k / 9k). C'est une activité à part : l'or ne suit pas le modèle produit/catégorie, il est mesuré en grammes × prix au gramme.

Géré dans un **module Or** dédié du dashboard → table `transactions_or` (rachat/vente, grammes, carat, prix/g, montant). Voir `database architecture.md`.

---

## Le deal

- **Client** : KornerCash (entreprise existante)
- **Deadline** : 30 juillet 2026
- **Livrable** : site web + dashboard + base de données — le tout livré clé en main
- **Hébergement** : séparé du VPS d'Anatole — tout doit être rendu/transférable à la fin du projet
- **Rôle d'Anatole** : technicien — construit la structure et les process. Pas de marketing, pas d'acquisition de trafic.

---

## Objectif final

1. Créer un **site web e-commerce** (inspiration BackMarket)
2. Créer un **dashboard de gestion** pour l'équipe
3. Créer la **base de données** qui relie tout
4. Tout mettre en ligne et livrer au client

---

## Contraintes importantes

- **Budget** : 200€ pour la création (hors rémunération)
- **Deadline** : 30 juillet 2026
- **Ressources humaines** : Anatole seul, assisté par Claude et Gemini
- **Hébergement** : tout doit être livrable et transférable au client
- **Langue** : français uniquement au lancement (anglais prévu plus tard → voir `stack techniques.md`)
- **Paiement en ligne** : **Stripe Checkout** (décidé le 2026-07-09 — carte, TWINT, Apple/Google Pay ; compte Stripe au nom de KornerCash)
- **Branding** : tout à créer (logo, couleurs, identité visuelle)

---

## Ressources disponibles

- Abonnement Claude = 20€/mois
- Abonnement Gemini = 10€/mois
- VPS / hébergement = à définir (séparé du VPS personnel)
- Base de données = à définir
- **Total investi = à calculer**

---

## Utilisateurs du dashboard

- **5 employés** de KornerCash
- À l'aise avec l'informatique
- Le dashboard doit être simple et intuitif malgré tout

---

## Ce que ce projet N'est PAS

- Anatole ne fait pas le marketing ni l'acquisition de trafic
- Anatole ne gère pas les photos produits au quotidien après livraison
- **DÉCISION FIGÉE** : pas de vente client / estimation en ligne. Les particuliers vendent ou font estimer leurs objets (= **rachat**) **uniquement en magasin**. Le site, lui, **vend au client** (e-commerce classique) — c'est le seul flux transactionnel en ligne.

---

## Pièces d'identité (rachats)

Lors d'un rachat, le dashboard doit permettre d'uploader une photo de la pièce d'identité du vendeur + sa signature. Actuellement KornerCash fait ça par photo stockée sur un ordinateur. Le dashboard reprend ce process en version numérique sécurisée.

**Réalisé** : photo de la pièce prise en direct (caméra live `CameraCapture`) → bucket privé `pieces-identite` ; signature sur écran ; champ optionnel « Code de la pièce » (`clients.code_passeport`, 2026-07-04) saisi au rachat, pré-rempli et affiché sur la fiche client.

---

## Expédition

KornerCash n'expédie pas encore — c'est nouveau avec le site. À définir avec le client :
- Choix du transporteur (La Poste Suisse, DHL...)
- Grille tarifaire (par poids, forfait fixe...)
- Frais de douane Suisse → France (qui paye ?)
Anatole présentera les options au client.

---

## Contexte de l'opérateur

- Localisation : Suisse — Lausanne
- Compétences clés : création grâce à l'IA (site web, SEO, image, vidéo)
- Profil : toute tâche répétitive doit devenir un process. Une tâche est répétitive quand on l'a fait plusieurs fois dans chaque mois.

---

# ÉTAT TECHNIQUE — session 2026-07-04

> Cette section décrit l'état réel du code au 2026-07-04. Le contexte métier ci-dessus ne change pas ; ci-dessous = où en est la construction technique.

## Vue d'ensemble

Deux applications distinctes + une base de données Supabase commune. **Les deux apps sont EN LIGNE** (Vercel, org `trusttheprocess`, plan Hobby, auto-deploy à chaque push).

| Livrable | Dossier | Statut |
| --- | --- | --- |
| **Site e-commerce** | `1 PROJETS/kornerCash/site/` | **EN LIGNE** — https://kornercash-site.vercel.app — repo `kornercashapplication-oss/kornercash-site`, branche `master` |
| **Dashboard de gestion** | `1 PROJETS/kornerCash/dashboard/` | **EN LIGNE** — https://kornercash-dashboard.vercel.app — repo `kornercashapplication-oss/kornercash-dashboard`, branche `main` |
| **Base de données** | Supabase (projet ref `nztqbfxsaduockzilrve`, région Frankfurt) | 9 tables, DB partagée par les 2 apps |

---

## Base de données (Supabase)

- Projet ref : `nztqbfxsaduockzilrve` — région Frankfurt.
- **9 tables** : `categories`, `sous_categories`, `produits`, `clients`, `commandes`, `ventes`, `rachats`, `transactions_or`, `produit_image_jobs` (file des jobs d'images IA de l'Atelier).
- **24 migrations**. Dernières : `add_visible_site_to_produits` (2026-07-03, toggle « Site »), `add_code_passeport_clients` (2026-07-04), `add_stripe_session_id_commandes` (2026-07-09, paiement Stripe). Détail complet : `database architecture.md`.
- **Schéma exact des colonnes** : source de vérité = `dashboard/lib/supabase/types.ts` (complet, 9 tables). ⚠️ `site/lib/supabase/types.ts` est un **sous-ensemble volontairement réduit à 8 tables** (sans `produit_image_jobs` ni `clients.code_passeport`).
- **RLS** :
  - `categories` / `sous_categories` / `produits` → lecture publique.
  - `clients` / `commandes` / `ventes` → `authenticated`, accès « own » (le client ne voit que ses propres données).
  - `rachats` / `transactions_or` / `produit_image_jobs` → verrouillées, accès service-key uniquement.

---

## Site e-commerce (`site/`)

### Stack

- **Next.js 16** (App Router — fichier `proxy.ts` au lieu de `middleware`, `cookies()`/`params` async).
- **React 19**, **Tailwind CSS 4** (config inline dans `app/globals.css`), **TypeScript 5**.
- `@base-ui/react`, `@supabase/supabase-js`, `@supabase/ssr`, `resend` (e-mails), `stripe` (paiement, serveur uniquement), `lucide-react`, `sonner`, `class-variance-authority`, `tailwind-merge`, `clsx`. Dev : `tsx`, `dotenv`.

### 3 clients Supabase

| Client | Fichier | Usage |
| --- | --- | --- |
| **public** | `lib/supabase/public.ts` | anon publishable key — lectures catalogue en SSR |
| **server** | `lib/supabase/server.ts` | `@supabase/ssr`, session cookie, RLS « own » (espace client) |
| **admin** | `lib/supabase/admin.ts` | service-role, server-only — Server Actions commande + fiche + adresses |
| browser | `lib/supabase/browser.ts` | auth navigateur |

`proxy.ts` = refresh de session.

### Pages (les routes)

`/` · `/c/[slug]` (filtres grade/prix/recherche + `?sousCategorie=`) · `/bons-plans` · `/recherche` · `/p/[slug]` (fiche remaniée) · `/connexion` · `/mot-de-passe-oublie` · `/compte` · `/compte/nouveau-mot-de-passe` · `/commande` · `/commande/merci` (retour paiement Stripe) · `/contact` · `/cgv` · `/auth/confirm` · `/api/stripe/webhook` + `sitemap` / `robots`. **Plus de route `/vendre`** — la vente client a été retirée du site public.

**Paiement Stripe Checkout (codé 2026-07-09 — EN PROD mode test depuis le 2026-07-19)** : `creerSessionPaiement` (`lib/actions/commande.ts`) remplace le placeholder `passerCommande` — commande créée `en_attente` + stock réservé (CAS conservé) + session Stripe hébergée (35 min, redirection) ; le **webhook** `/api/stripe/webhook` (`lib/paiement/confirmation.ts`) passe la commande `payee` + `mode_paiement` réel (carte/twint) sur `checkout.session.completed`, et restitue le stock (`annulee`) sur `expired`/`failed` — transitions idempotentes. `/commande/merci` = filet si webhook en retard + vidage du panier. Annulation depuis Stripe → expiration immédiate de la session + stock restitué. **Encaissement dans la devise affichée** (CHF ou EUR au taux fixe — TWINT visible seulement en CHF, automatique) ; `commandes.total` reste en CHF. Env : `STRIPE_SECRET_KEY` + `STRIPE_WEBHOOK_SECRET` (+ `NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY` non utilisée par le Checkout hébergé). Testé 8/8 en local par Anatole (2026-07-09, TWINT activé dans le Dashboard Stripe). Prod branchée le 2026-07-19 : env Vercel (site + dashboard) + endpoint webhook test `we_1Tuq6v1ywJcOEms7Udyg8pj9`. **Remboursement automatique** depuis le dashboard (refund Stripe, 2026-07-19) ; **Klarna retiré** (décision). **Reste** : re-test du parcours en prod (cartes test) + clés live à la mise en service réelle.

Header (`components/layout/header.tsx`) : logo à gauche, nav catégories avec **menus déroulants de sous-catégories** (`listCategoriesAvecSous`), recherche, sélecteur devise, compte, panier.

**PWA** (2026-07-09, ✅ validé par Anatole) : `app/manifest.ts` + icônes `public/icon-192/512/512-maskable.png` — le site s'installe sur l'écran d'accueil du téléphone (nom « KornerCash », ouverture plein écran `standalone`, monogramme noir sur ivoire).

### Structure de l'accueil (`app/page.tsx`), dans l'ordre

1. **Hero** = image bannière `/banniere-ete-upscale.jpg` pleine largeur, sans bouton (plafonnée à 60vh sur ordi).
2. **Réassurance** = 4 icônes (Contrôlé / Garanti 14j / Livraison / Retour) sur fond blanc, sous le hero.
3. **Meilleures affaires** = carrousel de produits (`components/product/product-carousel.tsx` : flèches ‹ › + dots, ~4,5 visibles, défilement produit par produit, mobile inclus) — placée AVANT « Trouvez votre marque » ; requête `listMeilleuresAffaires` (bons plans en priorité sinon prix les plus bas) ; bouton plein « Voir tous les bons plans » → `/bons-plans`.
4. **Trouvez votre marque** = grille de tuiles sections = sous-catégories (`components/category/section-tile.tsx`, config `lib/domain/sections.ts`) ; pas d'eyebrow.
5. **Avis Google** = `components/blocks/avis-google.tsx` : carrousel MANUEL de captures d'écran (`/public/avis/avis-N.png`), note figée **4,8 étoiles**, flèches + dots, fond en dégradé sky (`#e6ecfb`) → blanc ; contenu curé dans `lib/domain/avis.ts` (`AVIS_IMAGES` + `NOTE_GOOGLE`).

> La **vente client** (rachat / estimation) a été **RETIRÉE** du site public — le rachat se fait uniquement en magasin.

### Fiche produit (`/p/[slug]`)

Remaniée : `components/product/product-header.tsx` + `product-buybox.tsx` + `product-gallery.tsx` + `product-sticky-bar.tsx`, avec un bloc « Vous aimerez aussi ».

### Espace client (`/compte`) — refondu en tableau de bord

- **Bandeau d'identité** en tête (e-mail · membre depuis · nombre de commandes) = signature visuelle de l'espace.
- E-mail en lecture seule ; placeholder « ajouter un numéro » sur le téléphone.
- **Fiche auto-réparée** (2026-07-09) : un utilisateur authentifié sans fiche `clients` (fiche supprimée, ex. reset de données) la voit recréée à la volée par `getMaFiche` (rattachement par e-mail sinon création service-role) — sinon `/connexion` ↔ `/compte` bouclaient en redirections infinies.
- **Adresses multiples** (colonne jsonb `clients.adresses`) : menu déroulant + ajout / suppression (`components/compte/adresses-manager.tsx`).
- **Rappel discret du panier** en cours (`components/compte/rappel-panier.tsx`) : une seule ligne, sans récapitulatif ni bouton (le tiroir panier gère le reste).
- **Mes commandes** = lignes compactes avec détail repliable.
- Checkout (`/commande`) = sélecteur d'adresse enregistrée.

### Vérification e-mail (via RESEND, PAS de config Supabase)

- Inscription → `admin.generateLink({type:'signup'})` → e-mail de confirmation envoyé par Resend (`lib/email/resend.ts` + `templates.ts`) → compte non confirmé.
- Route `/auth/confirm` → `verifyOtp(token_hash)` → session (`type=signup` → `/compte` ; `type=recovery` → `/compte/nouveau-mot-de-passe`).
- **Mot de passe oublié (2026-07-19)** : `/mot-de-passe-oublie` → `generateLink({type:'recovery'})` → e-mail Resend (`templateReinitialisation`) → `/auth/confirm?type=recovery` → page nouveau mot de passe (`auth.updateUser`). Réponse identique que le compte existe ou non.
- Connexion **bloquée** si `email_confirmed_at` est `null`.
- Repli auto-confirmation si `RESEND_API_KEY` absente.
- Env : `RESEND_API_KEY` + `EMAIL_FROM`. **`RESEND_API_KEY` posée en prod Vercel (2026-07-19, mode test : envois uniquement vers l'adresse du compte Resend `kornercashapplication@gmail.com`** — les inscriptions d'autres adresses échoueront jusqu'à la vérification du domaine `kornercash.ch` dans Resend).

---

## Dashboard de gestion (`dashboard/`)

- **EN LIGNE** : https://kornercash-dashboard.vercel.app (repo `kornercashapplication-oss/kornercash-dashboard`, branche `main`, Vercel, auto-deploy à chaque push).
- **7 écrans** : Tableau de bord (patron), Ventes, Rachats, Produits, **Atelier**, Clients, Module Or (patron) — + pages imprimables : tickets / étiquettes.
- Stack : **Next 16 + React 19 + Tailwind 4 + Base UI** (shadcn base-nova).
- Auth : **2 mots de passe** (équipe / patron).
- **IA photos & descriptions produits** — CODÉE + DÉPLOYÉE en prod, validée en conditions réelles (189 générations bijoux, 0 échec). Génère une photo produit fond blanc pur en **2K** (passée de 4K à 2K le 2026-07-04 : la résolution est le seul levier de coût du modèle, 2K = 1.5×/$0.12 vs 4K = 2×/$0.16 — meilleur rapport netteté/prix ; verrou `num_images: 1`) + une description marketing e-commerce. Modèles fal.ai : image = `fal-ai/nano-banana-2/edit`, texte = `fal-ai/any-llm/vision` (`lib/ai/config.ts`, `lib/ai/fal.ts`, `lib/ai/builders.ts`). **Générations en parallèle via la file fal** (`queue.submit` + polling 4 s — plus de flux bloquant), multi-photos de référence (max 6), caméra live getUserMedia (`CameraCapture`/`RefUploader`), réordonnancement, rotation 90° (avant ET après validation), upload via fal storage (contourne la limite 4,5 Mo Vercel), erreurs server-action renvoyées **en valeur** `{ok,error}` (+ helper `lib/client-errors.ts`). Env : `FAL_KEY`. Voir `Plan implémentation - IA photos...` + `Spec - IA photos...`.
- **Atelier de saisie en lot** (`/atelier`, accès équipe) : création de produits en **brouillon** (invisibles du site et de la liste Produits) + génération d'images IA **asynchrone** (table `produit_image_jobs`, reprise auto au rechargement), validation avec comparaison photo prise ↔ image générée, puis « Finaliser » (≥ 1 image validée) → `en_stock`. Utilisé pour intégrer les ~180 bijoux réels (2026-07-03).
- **Scan douchette code-barres** (2026-07-04, déployé) : l'équipe scanne l'étiquette d'un produit (Code128 = `code_article`) avec sa douchette (lecteur clavier). Bouton « Scanner » sur la liste Produits (→ ouvre la fiche du produit) et dans Nouvelle vente (→ mode caisse enchaîné, re-scan = quantité +1, plafonné au stock). Composant `components/shared/scanner-dialog.tsx` — zéro dépendance, zéro caméra.
- **Toggle « Site » par produit** (`produits.visible_site`, 2026-07-03) : Switch en 1ère colonne de la liste Produits ; le site filtre `visible_site=true` sur toutes ses requêtes catalogue + garde au checkout.
- **Ventes — confort caisse** : recherche client (badge Blacklisté) + création rapide d'un client (e-mail optionnel) et d'un produit oublié depuis l'écran de vente ; plafond stock anti-survente ; montants affichés **au centime exact** partout (`chfExact` via `MoneyValue`).
- **Produits** : filtre par sous-catégorie (dépendant de la catégorie), état en 3 boutons.
- **PWA + icône** (2026-07-09, ✅ validé par Anatole) : `app/manifest.ts` (« KC Dashboard », plein écran `standalone`, theme `#3a5a99`) + **nouvelle icône = monogramme blanc sur bleu `#3a5a99`** (générée depuis l'icône du site — remplace le favicon Next.js par défaut) : `app/icon.png` + `app/apple-icon.png` + `app/favicon.ico` + `public/icon-192/512/512-maskable.png`.
- Reste : compléter `lib/domain/entreprise.ts` (coordonnées légales — raison sociale, adresse, IDE/TVA, encore en placeholders `À COMPLÉTER`) ; faire valider les tickets par la fiduciaire.

---

## Devise

CHF / EUR (prix stockés en CHF). Taux fixe provisoire **1 CHF ≈ 1.06 €** (à brancher sur une API réelle). Helpers dans `lib/domain/devise.ts` (`formatPrix`, `convertir`, `versChf`).

---

## Direction artistique (ACTUELLE)

> ⚠️ L'ancienne DA « luxe éditoriale » (or / blanc chaud / Archivo + Jost / zéro-arrondi) **n'est plus valable**. Ce qui suit est la DA en vigueur.

### Polices (via `next/font/google`, `app/layout.tsx`)

- **Montserrat** (`--font-display`) = titres + TOUS les sous-titres / eyebrows / labels / boutons / nav / prix.
- **Open Sans** (`--font-body`) = corps de texte (poids 400).
- Poids : labels de formulaire + boutons = **700** ; liens de nav = **700** (items des menus déroulants = 400) ; `.eyebrow` = **600**.

### Couleurs (tokens Tailwind v4 dans `app/globals.css`)

| Token | Hex | Rôle |
| --- | --- | --- |
| `--color-paper` | `#fcfbf8` | fond principal |
| `--color-ivory` | `#f7f5f0` | fond secondaire |
| `--color-ink` | `#121110` | texte |
| `--color-grey` | `#5c574e` | texte secondaire |
| `--color-line` | `#d0cabd` | bordures (visibles) |
| `--color-gold` | `#3a5a99` | **ACCENT = bleu roi feutré** (le token garde le nom « gold » mais rend du BLEU ; l'or est banni) |
| `--color-sky` | `#e6ecfb` | fonds bleu clair |
| `--color-forest` | `#24503a` | CTA « Acheter » |
| `--color-placeholder` | `#ece9e1` | placeholders |

**Grades** (échelle de verts) : parfait `#245c3a` / très bon `#4f9165` / correct `#86bd8f`.
**Sémantiques** : success `#5b7c63` / error `#9b4b3f`.

### Formes

- `--radius: 0.75rem` → **coins arrondis partout** (la règle « zéro arrondi » est LEVÉE). Ombres douces autorisées (menus, cartes).
- **Boutons** (`components/ui/button.tsx`) : variantes `solid` / `ghost` / `onDark`, Montserrat 700 uppercase, arrondis.

---

## Reste à faire (dette technique / phases suivantes)

> Push + déploiement Vercel des 2 apps = **FAIT** (au 2026-07-02). Ce qui reste ci-dessous.

- **DNS `kornercash.ch`** : brancher le domaine sur les 2 apps Vercel — accès au domaine **pas encore obtenu** (le domaine appartient à l'entreprise).
- **Accès associé** (demande 2026-07-09) : accès complet code + DB pour l'associé. Fait : repo `kornercash-docs` (ces notes + assets, convention 3 repos côte à côte dans le README) + consignes lecture/mise à jour des notes dans les AGENTS.md/CLAUDE.md des 2 apps. Reste : création du repo GitHub `kornercash-docs` + push, inviter l'associé en collaborateur Write sur les 3 repos, invitation org Supabase, décision déploiement Vercel (Hobby = pas de membre possible → GitHub Action deploy ou passage Pro), transmission des `.env.local` par canal sécurisé.
- **Sécurité d'accès dashboard** : mots de passe actuels **provisoires et volontairement simples** (posés le 2026-07-09, à remplacer avant livraison). Renforcement (comptes individuels, 2e facteur TOTP, appareils de confiance, passkeys, Cloudflare Access…) : options présentées le 2026-07-09, **aucune retenue — Anatole y réfléchit**.
- **Paiement Stripe** : **en prod (mode test) depuis le 2026-07-19** — testé 8/8 en local (2026-07-09), env Vercel + endpoint webhook prod posés, remboursement automatique branché côté dashboard (commit `e87b9ae`), Klarna retiré. Reste : re-test du parcours en prod (cartes test) + passage en clés live à la mise en service réelle.
- **E-mails transactionnels** : confirmation de commande, expédition.
- **E-mail Resend** : vérifier le domaine `kornercash.ch` dans Resend une fois le DNS obtenu.
- **Devise EUR** : brancher le taux réel (API) — actuellement taux fixe provisoire.
- **Coordonnées légales entreprise** (`lib/domain/entreprise.ts`) + **validation fiduciaire** des tickets.
- **Photos** des produits : un premier lot réel est déjà passé par l'Atelier IA (~180 bijoux intégrés, 219 bijoux en stock en DB au 2026-07-04) — reste la session photo du reste du stock (1 000–3 000 produits).
