# DATABASE ARCHITECTURE — KornerCash

Schéma complet de la base de données Supabase (PostgreSQL). 9 tables.

Source de vérité du schéma : `dashboard/lib/supabase/types.ts` (types générés depuis Supabase, complet — 9 tables, à régénérer après chaque migration). ⚠️ `site/lib/supabase/types.ts` est un **sous-ensemble volontairement réduit à 8 tables** (sans `produit_image_jobs` ni `clients.code_passeport` ; `visible_site` y a été ajouté à la main). Projet Supabase ref `nztqbfxsaduockzilrve` (région Frankfurt).

Pour le contexte métier → `contexte kornerCash.md` · Pour la stack → `stack techniques.md`

---

## Vue d'ensemble : les 9 tables

| Table | Rôle |
|---|---|
| `categories` | Les 5 grandes catégories |
| `sous_categories` | Sous-catégories rattachées à une catégorie |
| `produits` | L'objet en stock (fiche + exemplaire physique fusionnés) |
| `clients` | Fiche personne unique : acheteurs + vendeurs |
| `commandes` | L'enveloppe d'une vente (parent) |
| `ventes` | Le détail produit par produit d'une commande (enfant) |
| `rachats` | Enregistrement légal d'un rachat à un particulier |
| `transactions_or` | Rachats/reventes d'or au gramme (module Or) |
| `produit_image_jobs` | File d'attente des générations d'image IA (atelier de saisie en lot) |

---

## Les migrations

Historique appliqué (ordre chronologique) :

| # | Version | Nom | Apport |
|---|---|---|---|
| 1 | `20260628070540` | `init_schema_kornercash` | Schéma initial (8 tables sauf module Or) |
| 2 | `20260628071233` | `seed_categories` | Les 5 catégories de base |
| 3 | `20260628071723` | `create_storage_buckets` | Buckets `produits` + `pieces-identite` |
| 4 | `20260628072130` | `rls_policies_site_public` | Policies RLS du site |
| 5 | `20260628072420` | `harden_set_updated_at_search_path` | Sécurise `set_updated_at` (`search_path` figé) |
| 6 | `20260630072722` | `create_transactions_or` | Table module Or |
| 7 | `20260630081203` | `seed_sous_categorie_bijoux` | Sous-catégorie Bijoux |
| 8 | `20260630083531` | `add_cout_achat_produits` | Colonne `produits.cout_achat` (marge) |
| 9 | `20260630091007` | `normalize_produits_statut_grade_checks` | CHECK sur `produits.statut` / `grade` |
| 10 | `20260630091039` | `normalize_commandes_checks` | CHECK sur `commandes.canal` / `statut` |
| 11 | `20260630093910` | `add_mode_paiement` | Colonne `mode_paiement` (commandes + rachats) |
| 12 | `20260630135032` | `add_mode_paiement_especes_carte` | Valeur `especes_carte` autorisée |
| 13 | `20260630140026` | `add_remise_commandes` | Colonne `commandes.remise` |
| 14 | `20260630140813` | `add_remise_ventes_and_autres_produit` | Colonne `ventes.remise` + produit système « Autres » |
| 15 | `20260701100915` | `add_adresses_jsonb_clients` | Colonne `clients.adresses` (jsonb, adresses multiples du site — non-bloquant dashboard) |
| 16 | `20260701195633` | `fix_statut_defaults_and_index_auth_user` | Corrige les défauts de `statut` (`produits`='en_stock' / `commandes`='en_attente') + index sur `clients.auth_user_id` |
| 17 | `20260702100451` | `refonte_taxonomie_sections_mode_autres` | « Accessoires » → **Mode** (slug `mode`) ; **Montres** devient sous-catégorie de Mode (ancienne catégorie supprimée) ; nouvelle catégorie **Autres** (sous-cat **Figurines** / **Cartes Pokémon**) ; produits reclassés + `est_luxe` remis à `false` |
| 18 | `20260702100948` | `drop_produits_est_luxe` | Suppression définitive de la colonne `produits.est_luxe` (tag Luxe retiré partout) |
| 19 | `20260702115737` | `reorder_categories_nav` | Réordonne `categories.ordre` (1→5) pour la barre de nav : **Mode · Multimédia · Téléphone · Gaming · Autres** |
| 20 | `20260702122252` | `add_statut_brouillon_produits` | Ajoute la valeur `brouillon` au CHECK `produits.statut` (produit en cours de saisie dans l'atelier, invisible du site) |
| 21 | `20260702122305` | `create_produit_image_jobs` | Nouvelle table `produit_image_jobs` (file de génération d'images IA async, atelier) : RLS service-role, trigger `set_updated_at`, FK `ON DELETE CASCADE` |
| 22 | `20260703191259` | `add_visible_site_to_produits` | Colonne `produits.visible_site` (booléen, défaut `true`) — toggle « Site » par produit dans le dashboard |
| 23 | `20260704131759` | `add_code_passeport_clients` | Colonne `clients.code_passeport` (texte, nullable) — code de la pièce d'identité saisi au rachat, stocké sur la fiche client (libellé UI : « Code de la pièce ») |

---

## Schéma des relations

```
                    auth.users (Supabase Auth)
                          │
                          │ auth_user_id
                          ▼
   ┌──────────────┐   ┌─────────┐
   │  categories  │   │ clients │◄───────────────┐
   └──────┬───────┘   └────┬────┘                │
          │ 1              │                      │
          │ N              │ client_id            │ client_id
   ┌──────┴────────────┐   │                      │
   │  sous_categories  │   ▼                      │
   └──────┬────────────┘ ┌────────────┐      ┌─────────┐
          │              │ commandes  │      │ rachats │
          │              └─────┬──────┘      └────┬────┘
          │                    │ 1                │ 1
  categorie_id /               │ N                │ N
  sous_categorie_id            ▼                  │ rachat_id
          │              ┌──────────┐             │
          ▼              │  ventes  │             ▼
   ┌──────────────┐      └────┬─────┘      ┌──────────────┐
   │   produits   │◄──────────┘            │   produits   │
   └──────────────┘   produit_id           └──────────────┘
        (le même table produits — montré 2x pour la lisibilité)


   ┌─────────┐
   │ clients │◄──────────┐ client_id (nullable)
   └─────────┘           │
                  ┌──────────────────┐
                  │ transactions_or  │   ← table indépendante (module Or)
                  └──────────────────┘     ne touche pas produits
```

### Les liens en clair (clés étrangères)

| Table | Colonne | pointe vers | Signification |
|---|---|---|---|
| `sous_categories` | `categorie_id` | `categories.id` | Sa catégorie parente |
| `produits` | `categorie_id` | `categories.id` | Sa catégorie |
| `produits` | `sous_categorie_id` | `sous_categories.id` | Sa sous-catégorie (optionnel) |
| `produits` | `rachat_id` | `rachats.id` | De quel rachat il provient (optionnel) |
| `clients` | `auth_user_id` | `auth.users.id` | Son compte en ligne (optionnel) |
| `commandes` | `client_id` | `clients.id` | L'acheteur (optionnel en magasin) |
| `ventes` | `commande_id` | `commandes.id` | Sa commande parente |
| `ventes` | `produit_id` | `produits.id` | Le produit vendu (nullable — produit « Autres » ou produit supprimé) |
| `rachats` | `client_id` | `clients.id` | Le vendeur (obligatoire) |
| `transactions_or` | `client_id` | `clients.id` | La contrepartie — le vendeur pour un rachat d'or (optionnel) |

### Les 4 flux qui partent du client

```
clients (une personne)
   ├── commandes (ce qu'il a ACHETÉ)         → ventes → produits
   ├── commandes statut=réservée (ses RÉSERVATIONS)
   ├── rachats (ce qu'il t'a VENDU)           → produits (rachat_id)
   └── transactions_or (or qu'il t'a vendu / que tu lui revends)
```

Tout se regroupe sur son profil via son `id`.

---

## Détail des tables

Le schéma ci-dessous reflète exactement `dashboard/lib/supabase/types.ts`. « Nullable » = colonne optionnelle en base.

### 1. `categories`

| Colonne | Type | Nullable | Rôle |
|---|---|---|---|
| `id` | uuid (PK) | non | Clé primaire |
| `nom_fr` | texte | non | Nom affiché (ex: "Téléphone") |
| `slug_fr` | texte unique | non | URL |
| `image_url` | texte | oui | Visuel catégorie |
| `ordre` | entier | non (défaut) | Ordre d'affichage |

Contenu : 5 catégories (`nom_fr` live, dans l'ordre `ordre` de la nav depuis migration #19) → Mode · Multimédia · Téléphone · Gaming · Autres (slugs : `mode` · `multimedia` · `telephone` · `gaming` · `autres`). Hiérarchie à 2 niveaux (catégorie → sous-catégorie) : ex. **Montres** est une sous-catégorie de **Mode** ; **Figurines** / **Cartes Pokémon** sous **Autres** (migration #17).
**Bons Plans** n'est PAS une catégorie → c'est un tag (`est_bon_plan` sur `produits`). *(L'ancien tag `est_luxe` a été supprimé — migration #18.)*

### 2. `sous_categories`

| Colonne | Type | Nullable | Rôle |
|---|---|---|---|
| `id` | uuid (PK) | non | Clé primaire |
| `categorie_id` | uuid (FK → categories) | non | Catégorie parente |
| `nom_fr` | texte | non | Nom affiché (ex: "Consoles") |
| `slug_fr` | texte unique | non | URL |
| `ordre` | entier | non (défaut) | Ordre d'affichage |

### 3. `produits`

| Colonne | Type | Nullable | Rôle |
|---|---|---|---|
| `id` | uuid (PK) | non | Clé primaire |
| `titre_fr` | texte | non | Nom du produit (FR) |
| `description_fr` | texte long | oui | Description IA / SEO (FR) |
| `slug_fr` | texte unique | non | URL propre (FR) |
| `categorie_id` | uuid (FK → categories) | non | Grande catégorie |
| `sous_categorie_id` | uuid (FK → sous_categories) | oui | Sous-catégorie |
| `marque` | texte | oui | Marque (autocomplete) |
| `grade` | texte | oui | `Parfait état` / `Très bon état` / `État correct` (CHECK) |
| `prix` | nombre | non | Prix de vente en CHF |
| `cout_achat` | nombre | oui | Coût d'entrée en stock en CHF — pour calculer la marge (CHECK `>= 0`) |
| `quantite` | entier | non (défaut 1) | Nombre d'exemplaires (CHECK `>= 0`) |
| `photos` | tableau de texte (`text[]`) | non (défaut `{}`) | URLs des photos ordonnées |
| `defauts` | jsonb | non (défaut `[]`) | `[{description, photo}]` |
| `code_barres` | texte | oui | Étiquette — **généré auto** = `code_article` (encodable Code128) |
| `code_article` | texte | oui | Code interne KornerCash |
| `date_rachat` | date | oui | Date d'entrée en stock (vide si import en masse) |
| `statut` | texte | non (défaut) | `en_stock` / `vendu` / `reserve` / `archive` / `brouillon` (CHECK ; `archive` = soft-delete ; `brouillon` = en cours de saisie atelier, exclu du site ET de l'écran Produits) |
| `est_bon_plan` | booléen | non (défaut) | Tag Bons Plans |
| `visible_site` | booléen | non (défaut `true`) | Toggle « Site » — produit affiché ou non sur le site public (Switch dans la liste Produits du dashboard) |
| `rachat_id` | uuid (FK → rachats) | oui | Provenance |
| `created_at` | timestamp | non (auto) | Création |
| `updated_at` | timestamp | non (auto) | Dernière modif |

Note : produit fongible (2 boosters identiques) = 1 ligne, `quantite` = 2. Produit unique (2 iPhone différents) = 2 lignes distinctes.

### 4. `clients`

| Colonne | Type | Nullable | Rôle |
|---|---|---|---|
| `id` | uuid (PK) | non | Clé primaire |
| `auth_user_id` | uuid (FK → auth.users) | oui | Compte en ligne |
| `prenom` | texte | non | Prénom |
| `nom` | texte | non | Nom |
| `email` | texte | oui | Contact + compte |
| `telephone` | texte | oui | Contact |
| `adresse_rue` | texte | oui | Adresse « magasin » (dashboard) — pré-remplie ensuite |
| `adresse_code_postal` | texte | oui | — |
| `adresse_ville` | texte | oui | — |
| `adresse_pays` | texte | oui | Suisse / France |
| `adresses` | jsonb (défaut `[]`) | non | **Adresses multiples du site** — tableau d'objets adresse géré par l'espace client |
| `est_blackliste` | booléen | non (défaut) | Blacklisté oui/non |
| `piece_identite_url` | texte | oui | Photo pièce d'identité (rachats) — stockée une fois |
| `code_passeport` | texte | oui | Code de la pièce d'identité — saisi au rachat, mis à jour sur la fiche (libellé UI : « Code de la pièce ») |
| `created_at` | timestamp | non (auto) | Création |
| `updated_at` | timestamp | non (auto) | Dernière modif |

**Deux façons de stocker l'adresse (coexistent) :**
- `adresse_*` (rue / code postal / ville / pays) = les 4 colonnes plates, utilisées par le **dashboard** (fiche client magasin, une seule adresse). Ne pas les supprimer.
- `adresses` (jsonb, défaut `[]`) = **adresses multiples** gérées par le **site** (espace client `/compte` : ajout / suppression / menu déroulant, `components/compte/adresses-manager.tsx`). Le checkout du site sélectionne une adresse enregistrée depuis ce tableau.

Note : les champs de contact/adresse sont optionnels en base pour permettre la création d'un vendeur en magasin sans adresse. Le formulaire du site impose ses propres champs obligatoires.

### 5. `commandes` (l'enveloppe — parent)

| Colonne | Type | Nullable | Rôle |
|---|---|---|---|
| `id` | uuid (PK) | non | Clé primaire |
| `client_id` | uuid (FK → clients) | oui | L'acheteur (optionnel en magasin) |
| `date` | timestamp | non (défaut) | Date de la vente |
| `canal` | texte | non | `en_ligne` / `magasin` (CHECK) |
| `statut` | texte | non (défaut) | `en_attente` / `payee` / `expediee` / `livree` / `reservee` / `remboursee` / `annulee` (CHECK) |
| `total` | nombre | non (défaut) | Montant total CHF (produits − remise + port) |
| `mode_paiement` | texte | oui | `especes` / `carte` / `twint` / `especes_carte` (CHECK) |
| `remise` | nombre | non (défaut 0) | Réduction accordée en CHF (montant, CHECK `>= 0`). `total` = sous-total − remise + port |
| `frais_port` | nombre | non (défaut 0) | Livraison (5 CHF en ligne, 0 magasin) |
| `acompte` | nombre | non (défaut 0) | Montant versé — réservations (0 sinon) |
| `devise` | texte | non (défaut `CHF`) | Devise affichée/payée — `CHF` / `EUR` (CHECK) |
| `adresse_rue` | texte | oui | Adresse livraison (copie figée) |
| `adresse_code_postal` | texte | oui | — |
| `adresse_ville` | texte | oui | — |
| `adresse_pays` | texte | oui | Suisse / France |
| `numero_suivi` | texte | oui | Suivi colis (en ligne) |
| `montant_rembourse` | nombre | oui | Si remboursement |
| `created_at` | timestamp | non (auto) | Création |

Cette table absorbe les réservations (`statut`=réservée + `acompte`) et les remboursements (`statut`=remboursée + `montant_rembourse`).

### 6. `ventes` (le détail — enfant)

| Colonne | Type | Nullable | Rôle |
|---|---|---|---|
| `id` | uuid (PK) | non | Clé primaire |
| `commande_id` | uuid (FK → commandes) | non | Commande parente |
| `produit_id` | uuid (FK → produits) | oui | Produit vendu (nullable : produit « Autres » ou produit supprimé) |
| `titre_produit` | texte | non | Copie du titre au moment de la vente |
| `quantite` | entier | non (défaut) | Nombre d'exemplaires |
| `prix_unitaire` | nombre | non | Copie du prix de vente du jour (CHF) |
| `remise` | nombre | non (défaut 0) | Réduction sur cette ligne en CHF (CHECK `>= 0`). Net ligne = prix_unitaire × quantite − remise |

`titre_produit` et `prix_unitaire` sont figés : un produit unique disparaît du stock après vente, la ligne doit garder une trace de ce qui a été vendu et à quel prix.

### 7. `rachats`

| Colonne | Type | Nullable | Rôle |
|---|---|---|---|
| `id` | uuid (PK) | non | Clé primaire |
| `client_id` | uuid (FK → clients) | non | Le vendeur (obligatoire) |
| `date` | date | non (défaut) | Date du rachat |
| `montant` | nombre | non | Total versé au vendeur (CHF) |
| `mode_paiement` | texte | oui | `especes` / `carte` / `twint` / `especes_carte` (CHECK) |
| `signature_url` | texte | oui | Signature du vendeur pour ce rachat |
| `note` | texte | oui | Commentaire libre |
| `created_at` | timestamp | non (auto) | Création |

Les objets rachetés sont dans `produits` (via `rachat_id`). 1 rachat de 3 objets = 1 rachat + 3 produits.

### 8. `transactions_or` (module Or)

| Colonne | Type | Nullable | Rôle |
|---|---|---|---|
| `id` | uuid (PK) | non | Clé primaire |
| `type_transaction` | texte | non | `rachat` (KornerCash rachète l'or à un particulier) / `vente` (KornerCash le revend) |
| `grammes` | nombre | non | Poids en grammes (CHECK `> 0`) |
| `carat` | texte | non | Titre de l'or : `24k` / `22k` / `18k` / `14k` / `9k` (CHECK) |
| `prix_gramme` | nombre | non | Prix au gramme en CHF (CHECK `>= 0`) |
| `montant` | nombre **(généré)** | oui | = `grammes` × `prix_gramme` — non modifiable manuellement |
| `client_id` | uuid (FK → clients) | oui | La contrepartie (le vendeur pour un rachat) |
| `vendeur_nom` | texte | oui | Nom libre si aucune fiche client (fallback, ex: "Anonyme") |
| `date_transaction` | timestamp | non (défaut) | Date de la transaction |
| `created_at` | timestamp | non (auto) | Création |
| `updated_at` | timestamp | non (auto) | Dernière modif |

L'or n'entre pas dans `produits` : modèle gramme/carat trop différent. Le module Or vit dans sa propre table. Vocabulaire aligné sur le reste : **rachat** / **vente** (jamais "achat").

---

### 9. `produit_image_jobs` (file de génération d'images IA — atelier)

File d'attente des générations d'image IA de l'**atelier de saisie en lot** (`/atelier`). Découple l'attente (~2 min via fal `nano-banana-2/edit`) de la saisie : 1 produit → **N jobs** (plusieurs images possibles). Alimentée par la **file fal** (`queue.submit`) et suivie par polling.

| Colonne | Type | Nullable | Rôle |
|---|---|---|---|
| `id` | uuid (PK) | non | Clé primaire |
| `produit_id` | uuid (FK → produits, **ON DELETE CASCADE**) | non | Le produit (en `brouillon`) auquel le job appartient |
| `statut` | texte | non (défaut `en_cours`) | `en_cours` / `a_valider` / `validee` / `echec` / `annulee` (CHECK) |
| `fal_request_id` | texte | oui | `request_id` renvoyé par `fal.queue.submit` (suivi + reprise après rechargement) |
| `ref_photos` | text[] | non (défaut `{}`) | URLs fal des photos de référence (pour régénérer sans re-shooter) |
| `preview_url` | texte | oui | URL fal temporaire du résultat, en attente de validation employé |
| `photo_url` | texte | oui | URL Supabase publique finale, après « Garder » (validation) |
| `rotation` | entier | non (défaut 0) | Rotation appliquée à la validation (multiple de 90°) |
| `erreur` | texte | oui | Message si `statut='echec'` |
| `created_at` | timestamp | non (auto) | Création |
| `updated_at` | timestamp | non (auto, trigger) | Dernière modif |

**RLS activée sans policy** = accès **service-role uniquement** (dashboard) — le site n'y touche jamais. Le produit reste `brouillon` (invisible du site, car le catalogue filtre `en_stock`) jusqu'au clic **« Finaliser »** (≥ 1 image validée) qui le passe `en_stock`/`vendu`.

---

## Décisions de modélisation

- **Fusion produit/article** → une seule table `produits`, car la majorité des objets sont uniques. La quantité gère le cas des objets identiques.
- **Pas de table panier** → panier en localStorage (côté navigateur).
- **Tag en booléen** → `est_bon_plan` plutôt qu'une table tags + jointure. *(L'ancien `est_luxe` a été retiré — migration #18.)*
- **Photos et défauts en colonnes** (`text[]` / jsonb) sur `produits`, pas en tables séparées.
- **Réservations + remboursements fusionnés dans `commandes`** (via statut + champs).
- **Adresse** : deux mécanismes coexistent → 4 colonnes plates `adresse_*` (dashboard, adresse unique magasin) + `adresses` jsonb (site, adresses multiples de l'espace client). L'adresse retenue est recopiée/figée sur `commandes` à la date de commande.
- **Snapshot** : `titre_produit` + `prix_unitaire` recopiés sur `ventes` (best practice e-commerce).
- **Remise à deux niveaux** : remise globale sur `commandes.remise` (montant CHF) + remise par ligne sur `ventes.remise`.
- **Multilingue** : suffixe `_fr` sur les colonnes de texte affiché. Anglais plus tard = colonnes `_en`.
- **Module Or** → table séparée `transactions_or` (le modèle gramme/carat ne rentre pas dans `produits`). `montant` en **colonne générée** (`grammes` × `prix_gramme`) pour garantir la cohérence.
- **Marge produit** → colonne `cout_achat` sur `produits` (coût d'entrée par produit, optionnel), distincte de `rachats.montant` (un rachat peut couvrir plusieurs produits). Permet les KPI bénéfice/marge du dashboard.
- **Produit système « Autres »** → 1 ligne `produits` permanente (`code_article='AUTRES'`, `prix=0`, `quantite=999999`, `statut='en_stock'`) pour les ventes imprévues. Prix saisi à la vente, **stock jamais décrémenté**, **masqué** de l'écran de gestion Produits mais disponible dans la recherche de la vente. Une ligne `ventes` pointant sur ce produit peut avoir `produit_id` non nul, mais `produit_id` reste globalement nullable pour survivre à une suppression. ⚠️ Côté **site**, cette ligne est `en_stock` + `visible_site=true` en DB : seule l'exclusion applicative `SANS_AUTRES` (`code_article.is.null,code_article.neq.AUTRES`, `site/lib/queries/catalogue.ts` + garde checkout) l'empêche d'apparaître.
- **Visibilité d'un produit sur le site** → double filtre applicatif dans les 5 requêtes catalogue du site (`site/lib/queries/catalogue.ts`) : `statut='en_stock'` **ET** `visible_site=true` (+ garde au checkout `site/lib/actions/commande.ts`). Un `brouillon` (atelier) est donc invisible par construction ; côté dashboard, la liste Produits exclut aussi les brouillons (`.neq("statut","brouillon")`, `dashboard/lib/queries/produits.ts`) — `STATUT_BROUILLON` est volontairement hors de `STATUTS_PRODUIT` (`lib/domain/constants.ts`).

---

## Sécurité & implémentation (Phase 1 réalisée)

- **RLS activé** sur les 9 tables. Policies du site (vérifiées live) :
  - Lecture publique (`public`, SELECT) : `categories`, `sous_categories`, `produits` (le catalogue est visible par tous).
  - `clients` : 3 policies `authenticated` **own** (INSERT / SELECT / UPDATE).
  - `commandes` : 2 policies `authenticated` **own** (INSERT / SELECT).
  - `ventes` : 1 policy `authenticated` **own** (SELECT uniquement).
  - `rachats`, `transactions_or` et `produit_image_jobs` : **aucune policy** (0) → tables verrouillées, accessibles seulement via la clé service du dashboard / des Server Actions (données internes sensibles).
- **Modèle d'accès (3 clients Supabase côté site)** :
  - `lib/supabase/public.ts` — clé publishable anon, lectures catalogue SSR.
  - `lib/supabase/server.ts` — `@supabase/ssr`, session cookie, RLS « own » pour l'espace client.
  - `lib/supabase/admin.ts` — clé service-role (server-only, ignore le RLS) : Server Actions commande + fiche + adresses.
  - `lib/supabase/browser.ts` — auth navigateur · `proxy.ts` — refresh session (Next 16 : plus de `middleware.ts`).
  - Le dashboard utilise également la clé service (ignore le RLS).
- **Règles de suppression (ON DELETE)** :
  - `sous_categories.categorie_id` → cascade
  - `rachats.client_id` → restrict (protège l'historique légal d'un rachat)
  - `ventes.commande_id` → cascade
  - `ventes.produit_id` → set null (la ligne de vente survit si le produit est supprimé)
  - `produits.rachat_id` → set null (le produit survit si on supprime le rachat)
  - `produits.categorie_id` → restrict (on ne supprime pas une catégorie qui contient des produits)
  - `transactions_or.client_id` → set null (la transaction or survit si on supprime la fiche client ; le nom reste dans `vendeur_nom`)
  - `commandes.client_id` → set null · `clients.auth_user_id` → set null
  - `produit_image_jobs.produit_id` → cascade (supprimer le produit supprime ses jobs)
- **Index** : sur toutes les clés étrangères + index GIN full-text **français** (`titre_fr` + `description_fr` + `marque`) pour la recherche du site. `transactions_or` : index sur `client_id` et `date_transaction`. Index dédié sur `clients.auth_user_id` (migration #16). Index de statut : `produits_statut_idx`, `commandes_statut_idx`, `produit_image_jobs_statut_idx` (+ `produit_image_jobs_produit_id_idx`).
- **Storage** : bucket `produits` (public, 5 Mo, images) + `pieces-identite` (privé, 10 Mo, images/PDF).
- **Trigger** : `updated_at` mis à jour automatiquement sur `produits`, `clients`, `transactions_or` et `produit_image_jobs` (fonction `set_updated_at`, `search_path` figé pour la sécurité).

> **Note fiches clients** : un employé peut créer une **fiche client** depuis le dashboard (client physique en magasin) → ligne dans `clients` avec `auth_user_id` à NULL (aucun compte de connexion). Un **compte en ligne** (le client s'inscrit sur le site) remplit ce champ. La vérification d'e-mail est gérée côté site via Resend (`admin.generateLink({type:'signup'})` → route `/auth/confirm`), pas par la config Supabase.
