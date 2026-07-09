# kornercash-docs — Documentation projet KornerCash

Notes de contexte du projet KornerCash (site e-commerce + dashboard de gestion + base de données Supabase commune).

Ces notes sont la **source de vérité** sur le projet : toute IA (Claude Code, etc.) et tout développeur doit les lire **avant** de modifier le code, et les mettre à jour **après**.

## Convention : 3 repos clonés côte à côte

```
kornercash/
├── docs/       ← ce repo (kornercash-docs)
├── dashboard/  ← kornercash-dashboard (Next.js, branche main) → kornercash-dashboard.vercel.app
└── site/       ← kornercash-site (Next.js, branche master) → kornercash-site.vercel.app
```

## Les notes

| Note | Contenu |
|---|---|
| `contexte kornerCash.md` | **Point d'entrée** : le métier, le vocabulaire (rachat/vente), l'état technique à jour |
| `database architecture.md` | Schéma Supabase complet (tables, migrations) — la DB est **partagée par les 2 apps** |
| `Plan technique dashboard.md` | Architecture du dashboard |
| `Systeme de design - site.md` | Design system du site (couleurs, formes, boutons) |
| `stack techniques.md` | Choix techniques et leurs raisons |
| `Road map kornerCash.md` | Roadmap et phases du projet |
| `Spec - IA photos et descriptions produits (dashboard).md` | Spec de la feature IA photos/descriptions |
| `Plan implémentation - IA photos et descriptions produits (dashboard).md` | Plan d'implémentation de cette feature |
| `Bijoux à intégrer - catalogue.md` | Intégration du stock bijoux (terminée) |

## Règles de travail

1. **Début de session** : `git pull` dans ce repo (et dans les repos code).
2. **Avant de coder** : lire au minimum `contexte kornerCash.md` + `database architecture.md`.
3. **Après toute modification** de code ou de schéma : mettre à jour la ou les notes concernées, puis commit + push de ce repo.
4. La base Supabase est **commune** au site et au dashboard : toute migration peut impacter les deux apps et doit être documentée dans `database architecture.md`. Une seule personne applique des migrations à la fois — prévenez-vous avant.

## Assets

`assets/` contient les logos, bannières, tuiles de sections et le template IA (`assets/autres/carré 700 700.png`, pré-requis de la feature IA). Les dossiers `assets/nouveau produit/` et `assets/produits/` (photos brutes, ~560 Mo) ne sont **pas versionnés** — les photos produits finales vivent dans Supabase Storage.
