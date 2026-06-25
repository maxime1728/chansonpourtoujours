# Mapping backend — Chanson Pour Toujours (CPT)

> **Source : état LIVE Airtable + Make, vérifié le 2026-06-24** (et NON les blueprints locaux de `cm-audit`, qui sont périmés).
> CPT reproduit le backend de « Chanson Mémoire » en marque **cadeau seulement**, sur une **infra séparée**.

---

## 0. Ce qui existe vraiment (live)

- **Airtable** : une seule base **« Chanson Mémoire »** (`appIADNKzDOVtpjWj`), **11 tables**. **Aucune base CPT encore** → à créer.
- **Make** (org `labmarketing.ca`, équipe « My Team » `422966`) :
  - Dossier **« Chanson Mémoire » (`321788`)** = le funnel CM **actuel**, 11 scénarios « CM - … ».
  - Dossier **« Song » (`313868`)** = système **LEGACY** (Closebot/Digitact, « Song Survey », « *Memories* »), 12 scénarios. **À ignorer pour CPT.**
- **Stripe** : sous-compte CPT déjà créé par Maxime (clés à brancher).
- Les docs locaux `cm-audit` (CM_mapping_airtable.md à 4 tables) sont **périmés** : la réalité = 11 tables + couche marketing + support + séquences.

---

## A. Airtable — base à dupliquer en « Chanson Pour Toujours » (11 tables)

| Table | Rôle | Champs clés |
|---|---|---|
| **Clients** | 1 client = 1 courriel | email (PK), contact_name, consent_status/date, projects (link), rollups (total_projects, client_purchases, songs_preachat) |
| **Projects** | Le cœur (~90 champs) | token ; **sondage** : deceased_name, Relationship, music_style, voice, mood, **Occasion**, what_made_unique, memories, memory_to_keep, song_type, language ; **état** : commercial_status, funnel_step ; **Stripe** : stripe_session_id, stripe_payment_intent, amount, purchase_date, cgv_acceptees_at ; **preuve Loi 25** : delivery_signature_name/at, acceptance_ip, acceptance_user_agent, delivery_accessed_at, delivery_acceptance_text_version ; **tracking** : checkout_started_at, preview_played_at/count, downloaded_at/count ; **cadeaux** : pdf_url, signet_url, pdf_template, signet_template, signet_text ; **upsells** : instrumental_task_id/url, video_task_id/url ; **corrections post-achat** : correction_request, adjusted_lyrics, adjusted_style_prompt, mode_correction, approval_status, ref_id, refaire, cover_task_id ; **plafond** : cap_help_email/at ; **waitlists** : waitlist_memoire, waitlist_video ; **nurture** : nurture_step/next_at/status ; **CAPI** : event_id, capi_lead/checkout/purchase_sent ; **attribution** : utm_*, fbclid, fbc, fbp, landing_page ; **liens** : generations, upsells, Pub, Conversations, Inscriptions, Clics |
| **Generations** | 1 version de chanson | generation_no, type, lyrics, song_title, suggestions, cloudinary_audio_url, song_id, preview_slug, full_slug, generation_status, suno_task_id, post_purchase, gen_music_style/mood/voice, lyrics_timing, incident_*, sentinelle_retries |
| **Upsells** | Add-ons achetés | Project, type, price, status, purchased_date, delivery_url |
| **Pubs** | Créatifs Meta (scorecard) | ad_name (= utm_content, jointure), ad_id, campaign, adset, taxonomie (format/angle/hook_id/visual_id/version), rollups (spend, revenue, purchases), ROAS, CPA |
| **Pubs_Performance** | Perf Meta jour × pub | perf_key (ad_id_date), spend, impressions, video_3s, link_clicks, hook_rate, ctr |
| **Hook_Bank** | Banque d'accroches | hook_id, hook_text, angle, source, status |
| **Conversations** | File support courriel | expediteur, sujet, message, brouillon_ia, statut, repondre_de, envoyer |
| **Inscriptions** | Séquences courriel | client, Projet, sequence, step, statut, next_at |
| **Clics** | Journal clics liens | campagne, Projet, clicked_at |

**Adaptations de champs pour CPT** (front déjà aligné) : `deceased_name` → **recipient_name**, `what_made_unique` → **what_makes_special**, ajouter **special_phrase** ; `Occasion` existe déjà ; `song_type` CPT = toujours **cadeau**. Les 5 occasions CPT regroupées (Amour, Anniversaire, Famille, Célébration, Juste parce que).

---

## B. Make — dossier « Chanson Mémoire » (321788) à cloner en « Chanson Pour Toujours »

| Scénario | Rôle | Déclencheur |
|---|---|---|
| **CM - MAKE A - Lyrics** | Crée Client+Project, génère les paroles (Anthropic), crée la 1re Generation | Webhook du sondage (`souvenirs.html`) |
| **CM - MAKE C- gen** | Génère la chanson (Suno) à partir des paroles confirmées | Webhook confirmation (`revision.html`) |
| **CM - MAKE C-cb (callback Suno)** | Callback Suno → pose l'audio sur la Generation | Webhook Suno (par requête) |
| **CM - MAKE D - Stripe (achat)** | checkout completed → purchased, purchased_generation_no, CAPI Purchase | Webhook Stripe |
| **CM - Sentinelle** | Récupère/relance les chansons bloquées (interroge Suno) | Cron 30 min |
| **CM - Relance cover (post-approbation)** | Relance Suno cover/regen après approbation d'une correction | Cron / champ `refaire` |
| **CM - Alerte chanson bloquee >10h** | Alerte interne | Cron horaire |
| **CM - MAKE Insights (depense Meta)** | Pull quotidien API Meta → Pubs_Performance | Cron quotidien |
| **CM - Jointure Pub (utm_content → Pub)** | Lie Project ↔ Pub | Automation |
| **CM - Jointure Hook (hook_id → Hook_Bank)** | Lie Pub ↔ Hook_Bank | Automation |
| **CM - courriel entrant (support)** | Route Mailgun → Conversations + brouillon Claude | Webhook Mailgun |

---

## C. Câblage front CPT ↔ backend (placeholders à brancher)

| Page CPT | Endpoints / webhooks | Backend |
|---|---|---|
| `souvenirs.html` | webhook MAKE A (`TON_WEBHOOK_MAKE_CPT`) + backup Netlify Forms | MAKE A → Client/Project/Generation |
| `revision.html` | `/api/lire-projet` (poll), `/api/generate-lyrics` (retry/regen), webhook B (`TON_WEBHOOK_MAKE_B_CPT` = C-gen) | MAKE A + Suno |
| `attente-chanson.html` | `/api/lire-projet` (poll `audio_generated`) → `/apercu` | C-cb |
| `apercu.html` | `/api/lire-versions`, `/api/creer-checkout` (Stripe), `/api/suivi-funnel`, `/api/essayer-style`, `/api/generate-lyrics`, `/api/aide-plafond` | MAKE D + Stripe |
| `page-achat-revision.html` | `/api/lire-projet`, `/api/accepter-livraison`, `/api/decortique` | livraison + corrections |
| `page-chanson.html` | `/api/lire-projet`, `/api/telecharger`, `/api/lancer-cadeau`, `/api/lancer-signet`, `/api/creer-upsell`, `/api/choix-memoire` | livraison + cadeaux + upsells + waitlists |

Pixel/CAPI : `TON_PIXEL_ID_CPT` (front) + champs `capi_*` + `event_id` (dédup Purchase = stripe_session_id).

---

## D. À construire pour CPT (lot backend)

1. **Airtable** : dupliquer la base → **« Chanson Pour Toujours »** (11 tables, mêmes champs ; renommages recipient_name/what_makes_special/special_phrase ; song_type=cadeau). *Pilotable via MCP Airtable.*
2. **Make** : nouveau dossier **« Chanson Pour Toujours »** + cloner les 11 scénarios `321788`, re-pointer la base Airtable + les webhooks + Stripe. *MCP Make disponible.*
3. **Stripe** (sous-compte CPT, créé) : produits **79,97 $** (chanson) + **15,99 $** (instrumentale) + **13,99 $** (paroles karaoké, à confirmer) ; clés dans Netlify.
4. **Netlify** : porter les ~40 functions (lire-projet, generate-lyrics, creer-checkout, accepter-livraison, telecharger, decortique, lancer-cadeau/signet/cover/instrumentale/paroles-vivantes, callbacks, crons…) dans le repo CPT + env vars (base Airtable CPT, Stripe CPT, webhooks Make CPT, sous-domaines Mailgun info/support/achat, Anthropic, Cloudinary, Suno, pixel/CAPI Meta). Vérifier la **liste réelle** des functions sur le repo/Netlify chansonmemoire live (cm-audit peut être périmé).
5. **Brancher** tous les placeholders du front (`TON_WEBHOOK_*_CPT`, `TON_PIXEL_ID_CPT`, `/api/*`).

> **Accès MCP au 2026-06-24** : Airtable ✅ live, Make ✅ live. Stripe/Netlify : non vérifiés. Le sous-compte Stripe et la config live restent des actions humaines (Maxime).
