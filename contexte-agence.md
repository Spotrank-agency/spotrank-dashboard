# CONTEXTE AGENCE — SPOTRANK
## Document de référence complet pour le développement

---

# 1. VISION & BUSINESS MODEL

## Ce qu'est Spotrank
Spotrank est une **agence de marketing local** qui aide des contractors américains (plombiers, HVAC, électriciens, roofers, peintres, landscapers, handymen, etc.) à générer plus de leads via le digital. L'agence se positionne comme un partenaire "tout inclus" — le contractor paie un abonnement mensuel et Spotrank s'occupe de tout.

## Offre & Pricing
- **Prix** : $297/mois tout inclus
- **Ce que le client reçoit** : site web optimisé + SEO local + optimisation Google Business Profile + gestion de réputation (avis automatisés) + CRM (LeadConnector/GHL) + automations SMS/email + rapports mensuels
- **Zones géographiques** : USA principalement (leads scrappés et ads US), avec expansion possible Canada, France, Belgique, Suisse

## Objectif de scale
- Dashboard pensé pour gérer **50 à 200 clients simultanément**
- Équipe répartie par rôles (setters, closers, CSMs, VAs)
- Maximum d'automatisation pour réduire le travail manuel

## Principes UX du dashboard
- **1-2 clics max** pour toute action courante
- Actions rapides sur chaque carte (sans ouvrir de nouvelle page)
- Alertes intelligentes — seulement ce qui nécessite une action
- Design system strict : police **Onest**, couleur CTA `#1a56e8`, sidebar dark `#0f1f3d`
- **PAS** de design générique "dashboard SaaS violet"
- Simple, épuré, ultra rapide à utiliser

---

# 2. STACK TECHNIQUE

## Architecture actuelle
- Frontend : HTML/CSS/JS standalone (pas de framework, pas de build system)
- Backend : Supabase (PostgreSQL + Auth OTP + Edge Functions + Realtime + Storage)
- SMS : Twilio (via Supabase Edge Function proxy)
- Email : SendGrid (via Supabase Edge Function proxy)
- Visio : Zoom API (meetings, scheduler, webhooks)
- Paiement : Stripe (abonnements $297/mo, MRR tracking)
- CRM client : LeadConnector / GoHighLevel (GHL) pour les sous-comptes clients
- Hébergement : Netlify — déployé sur https://leafy-otter-8c951e.netlify.app/
- IA : Anthropic API (claude-sonnet-4-20250514)

## Configuration API centralisée (SPOT_API)
Toute la config API est dans un objet `SPOT_API` dans le dashboard :
- `baseUrl: '/api'` → remplacer par l'URL Supabase Edge Functions en prod
- **SendGrid** : from_email contact@spotrank.io, reply_to reply@spotrank.io, catégories, unsubscribe groups CAN-SPAM, IP pool dédié, domain authentication requise
- **Twilio** : from_phone +18005POTRANK, messaging service SID pour scheduling, status callback webhooks, validity period 4h, A2P 10DLC registration requise pour SMS marketing US
- **Zoom** : durée 30 min pour tous les calls, waiting room activé, auto-recording désactivé
- **Webhooks** : sendgrid events, twilio status, twilio inbound, zoom events, stripe events — tous sous /api/webhooks/

## Contraintes techniques importantes
- `confirm()` et `prompt()` natifs sont **bloqués** par l'iframe Claude → utiliser des **modals custom** partout
- `contenteditable` capture le focus et interfère avec les clics → `blur()` avant d'ouvrir les modals
- Les 3 fichiers dashboard doivent être dans le **même dossier** pour que l'iframe email editor fonctionne
- Les données sont en **variables JavaScript** (mode démo) → en production, chaque action = appel Supabase persisté
- Les données sont reset à chaque rechargement du fichier HTML
- Le scoring est recalculé côté client → en production, calcul côté serveur via trigger PostgreSQL ou Edge Function

---

# 3. FICHIERS DU PROJET

## Dashboard (3 fichiers — doivent cohabiter dans le même dossier)
- `spotrank-dashboard.html` (~745 Ko) — Dashboard principal : toutes les pages (overview, pipeline, calendrier, automations, clients, notes, settings, performance, team)
- `spotrank-builder.html` (~193 Ko) — Flow Builder : éditeur visuel drag & drop d'automations SMS/Email avec 14 flows préconfigurés
- `spotrank-email-editor.html` (~18 Ko) — Éditeur email WYSIWYG chargé en iframe dans le dashboard. Toolbar : taille texte, B/I/U, listes, alignement, liens, variables

## Funnel Pages
- `funnel-page-1.html` — Page d'acquisition lead Ads : VSL 6 min (débloqué à 4 min) → formulaire de qualification (9 champs + website + GMB) → calendrier Zoom embed
- `funnel-page-2.html` — Page post-booking : confirmation + instructions SMS

## Pages Statiques
- `index.html` — Landing page / site principal Spotrank
- `contact.html` — Page de contact
- `optin.html` — Page opt-in avec formulaire GHL
- `legal.html` — Terms & Conditions
- `privacy.html` — Politique de confidentialité

---

# 4. ÉQUIPE & RÔLES

## Structure de l'équipe

### Admin (1)
- Accès total à toutes les fonctionnalités
- Peut voir/modifier les notes de tous
- Peut créer/supprimer les notes entreprise
- Peut prévisualiser le dashboard en tant qu'un autre membre ("Preview as")

### Setters (3 : Lucas, Julien, Léa)
- **Rémunération** : commission $75 par show (le lead se présente au call closer)
- **Rôle** : contacter les leads froids, qualifier, booker un call pour le closer
- **Visibilité kanban** : Lead In → Booked (ses leads assignés uniquement)
- **Actions** : SMS, appeler, noter, qualifier, avancer dans le pipeline
- **NE PEUT PAS** : envoyer un lead en Lost, accéder aux settings, à l'équipe, aux clients actifs
- **Après un call** : doit donner son avis (🟢🟠🔴) et choisir "Qualifié → passe au Closer" ou "À recontacter"

### Closers (2 : Marc, Sarah + 1 hybride Thomas setter/closer)
- **Rémunération** : commission $300 par close (deal signé)
- **Rôle** : faire le call Zoom de vente, closer le deal
- **Visibilité kanban** : Booked → Closed (ses leads assignés uniquement)
- **Actions** : marquer Présent/No-show, Closé/Follow-up/Lost
- **Avant le call** : consulte la fiche lead (drawer) avec le briefing IA du setter
- **Le closer est le seul** qui peut envoyer un lead en Lost ou le closer

### CSMs (3 : Sophie, Camille, Nathan)
- **Rémunération** : salaire ~$3000/mois
- **Rôle** : onboarding des nouveaux clients, suivi mensuel, gestion des problèmes
- **Visibilité** : auto-switch vers pipeline onboarding, ses clients assignés

### VAs (2 : Emma, Alexis)
- **Rémunération** : salaire ~$2000-2200/mois
- **Rôle** : support clients, gestion des sites, tâches opérationnelles
- **Visibilité** : clients, sites, notes uniquement

### Membre hybride (Thomas — setter + closer)
- Cumule les permissions des deux rôles, voit Lead In → Closed, perçoit $75/show + $300/close

## Auto-assign
- **Lead In** → auto-assign setter (random parmi les disponibles, status='active')
- **Booked** → auto-assign closer (random parmi les disponibles)
- **Closed** → auto-assign CSM (load balancing par nombre de clients assignés)

---

# 5. SOURCES DE LEADS & ACQUISITION

## Source 1 : Ads (Inbound — leads chauds)

### Parcours du lead
1. Le lead voit une pub Facebook/Google
2. Arrive sur `funnel-page-1.html`
3. Regarde la VSL de 6 minutes (bouton booking débloqué à 4 min via timer JS)
4. Clique "Book my free audit call" → Step 2
5. Remplit le formulaire de qualification
6. Submit → formulaire disparaît → calendrier Zoom embed apparaît
7. Booke un créneau
8. Données envoyées à `/api/leads/qualify` (Supabase Edge Function)

### Données captées par le formulaire Ads

**Bloc "Your info" :**
- First name (texte, requis)
- Phone (tel, requis)
- Business name (texte, requis)
- City (texte, requis)
- State (dropdown 50 US states + DC, requis)
- Trade (recherche avec dropdown 90 trades alphabétique, avec filtre live + fade scroll, requis)

**Bloc "Online presence" (optionnel) :**
- Do you have a website? (oui/non toggle) → si oui : champ Website URL
- Do you have a Google Business Profile? (oui/non toggle) → si oui : bouton "🔍 Find my business on Google Maps" qui ouvre google.com/maps/search/[business]+[city]+[state]+USA dans un nouvel onglet + champ Maps URL

**Bloc "Quick qualification" :**
- Jobs per month (dropdown 1-5 / 5-15 / 15-30 / 30+, requis)
- How do you get clients? (dropdown Word of mouth / Online ads / Directory / Nothing, requis)
- Are you the owner? (oui/non toggle, requis)

### Comportement post-submit
- Lead créé dans Supabase avec source:'ads', stage:'booked', vsl_watched:true
- Toute la qualification pré-remplie dans la fiche
- Le closer auto-assigné retrouve tout dans le drawer avant son call
- **Le setter ne touche JAMAIS les leads Ads**

### Liste des 90 trades
Additions & Remodeling, Air Conditioning, Appliances, Appraiser, Architects & Engineers, Art & Mirror Mounting, Audio/Visual & Computers, Awnings, Brick & Stone, Cabinets, Carpenters, Carpet & Upholstery Cleaning, Ceilings, Central Vacuum, Cleaning & Maid Services, Commercial Contractors, Concrete, Construction, Countertops, Decks, Demolition Service, Designers & Decorators, Disability Services, Disaster Recovery Services, Docks, Doors, Drywall & Plaster, Electrical, Excavation, Fans, Fences, Fireplace & Wood Stoves, Flooring & Carpet, Foundations, Fountains & Ponds, Furniture Assembly, Furniture Repair & Refinish, Garage & Garage Doors, General Contractors, Glass & Mirrors, Gutters, Handyman Services, Heating & Furnace Systems, Home Inspection, Home Maintenance, Home Services, Hot Tubs Spas & Saunas, Household Help, HVAC, Insulation, Landscaping, Lawn & Garden Care, Lifting & Moving Heavy Items, Locksmith, Metal Fabrication, Mold & Asbestos Services, Moving, New Home Builders, Organizers, Outdoor Playgrounds, Packing & Unpacking Services, Painting, Paving, Permit Services, Pest Control, Plumbing, Powdercoating, Remodeling, Roofing, Sandblasting Service, Septic Tanks & Wells, Sheds & Enclosures, Siding, Sign Making Service, Skylights, Snow Removal Service, Solar, Sports Equipment Assembly, Stained Glass, Swimming Pools, Tennis Or Game Court, Tile, Tree Service, Wall Coverings, Waste Material Removal, Water Treatment System, Waterproofing, Window Coverings, Windows, Yard & Garden Work

## Source 2 : Cold Call (Outbound — leads froids)

### Données venant du CSV (scraper)
Nom complet, Trade, Ville, Téléphone, Site web URL, Google Maps URL, Nombre d'avis Google, Note Google rating — toutes en lecture seule dans le drawer

### Parcours du lead cold
1. Lead importé via CSV → arrive dans Lead In
2. Auto-assign à un setter disponible
3. Premier SMS/appel du setter
4. Pas de réponse → No Contact (SMS automation : 3 tentatives espacées)
5. 20 tentatives sans réponse → Long Term Nurture (1 SMS/semaine x4)
6. Réponse positive → setter qualifie via le drawer → booke un call → Booked
7. Le closer prend le relais

### Qualification par le setter (dans le drawer)
Avis setter (🟢🟠🔴), Jobs/mois, Source clients, Proprio?, Nb employés, Notes rapides, Briefing IA

---

# 6. PIPELINE SALES — 9 STAGES

### 1. Lead In (Setter)
Nouveau lead cold importé. Auto-assign setter. Flow `flow_lead_in_cold` : premier SMS d'accroche.

### 2. No Contact (Setter)
Contacté sans réponse. Flow `flow_no_contact_cold` : séquence 3 SMS (J+1, J+3, J+5). Badge 💬 X/20. Après 20 tentatives → passe en Nurture.

### 3. Long Term Nurture (Auto)
Plus de 20 tentatives. Flow `flow_nurture_longterm` : 1 SMS/semaine x4 semaines.

### 4. Zoom Call Booked (Both)
Call Zoom booké. Auto-assign closer. Flow `flow_booked` : SMS confirmation + reminder J-1 + H-1. Zoom meeting créé via `bookCallForLead()`. Boutons closer : ✅ Présent / ❌ No Show.

### 5. No Show (Setter)
Lead absent. Détection auto via webhook Zoom (`processZoomMeetingEnded()`). Flow `flow_noshow` : SMS excuse + email J+1 + SMS J+3. Impact scoring : -15.

### 6. Zoom Call Done (Closer)
Call terminé. Flow `flow_call_done`. Boutons : 🎉 Closé / 🔄 Follow up / ❌ Lost. Le setter donne son avis 🟢🟠🔴 via le modal.

### 7. Follow Up (Closer)
Intéressé mais pas signé. Flow `flow_follow_up`. Scénarios A/B/C selon la raison. Tags FU-A, FU-B, FU-C.

### 8. Client Closed 🎉 (Closer)
Deal signé. Auto-assign CSM (load balancing). Tag 'client'. Bascule pipeline onboarding. Flow `flow_closed` : SMS welcome + form + app + vidéos + launch call (5 étapes sur 3 jours).

### 9. Lost (Auto)
Abandon. Seul le closer peut killer un lead. Flow `flow_lost` : tag + notif + réactivation auto 60 jours.

**Règle critique** : le setter n'a JAMAIS l'option Lost. Ses options : "Qualifié → Closer" ou "À recontacter".

---

# 7. PIPELINE ONBOARDING — 9 STAGES

1. Client Signé ✍️ (CSM) — email bienvenue dans les 2h
2. Onboarding Form 📋 (CSM) — formulaire custom values envoyé
3. Form Submitted ✅ (CSM) — formulaire reçu, projet prêt
4. Launch Call Booké 📅 (CSM) — préparer accès GMB + site
5. Launch Call Done 🚀 (CSM) — setup en cours (site, GMB, GHL)
6. En Retainer ⭐ (CSM) — client actif, check-in J+7 obligatoire
7. Check-in Dû 🔔 (CSM) — 7 jours en retainer, appel/message requis
8. At Risk ⚠️ (Admin) — problème actif (CB déclinée, GMB bloqué, plainte)
9. Churned ❌ (Auto) — client perdu, documenter la raison

Tags rapides sur cartes : 💳 CB Déclinée / 🗺 GMB Bloqué

---

# 8. LEAD SCORING DYNAMIQUE

## Philosophie
Le score n'est JAMAIS figé. Il se recalcule automatiquement à chaque événement. Base = 50, plafonné 0-100.

## Formule complète

**Avis setter (signal le plus important)** :
- Très intéressé 🟢 : +35
- Moyennement 🟠 : +15
- Pas intéressé 🔴 : -20

**Source** :
- Ads + VSL vue : +25
- Cold : +0

**Réactivité** :
- Reply au 1er SMS : +15
- Reply après 5+ SMS : +5
- Pas de reply (5+ tentatives) : -10

**Présence digitale (CSV ou formulaire)** :
- < 20 avis Google : **+20**
- 20-80 avis : +5
- 80+ avis : -5
- Rating < 4.2 : **+20**
- Rating 4.2-4.5 : +5
- Rating > 4.5 : 0
- Pas de site web : +10
- Site basique : +5
- Site pro : -5

**Profil (setter remplit)** :
- Proprio oui : +10
- Proprio non : -10
- Solo : +10
- Petite équipe (2-5) : +5
- Moyenne/Grande (6+) : +10

**Pénalités** :
- No-show : -15

## Couleurs
- 🟢 Vert **75+** : super lead, closer priorise
- 🟠 Orange **45-74** : lead correct, signaux manquants
- 🔴 Rouge **< 45** : mauvais lead, laisser en nurture auto

## Lead parfait
Setter très intéressé (+35) + Ads/VSL (+25) + 8 avis (+20) + rating 3.5 (+20) + solo (+10) + pas de site (+10) + proprio (+10) = 130 → plafonné à 100

## Événements déclenchant le recalcul
saveQualField(), closerMarkPresent/NoShow/Closed/FollowUp/Lost(), submitCallResult(), addLead(), recalcAllScores() au boot

---

# 9. FICHE LEAD (DRAWER)

S'ouvre en cliquant sur le nom du lead (kanban ou calendrier).

## Section 1 : Profil Google (CSV — lecture seule)
Avis Google (nombre + étoiles), Site web (lien cliquable), Google Maps (lien cliquable). Badge "Import CSV".

## Section 2 : Qualification (setter remplit)
- Avis setter : 3 boutons 🟢🟠🔴 (surligné avec couleur correspondante)
- Jobs/mois (input nombre)
- Source clients (dropdown)
- Proprio ? (dropdown)
- Nb employés (dropdown solo/petite/moyenne/grande)
- Chaque modification → recalcul score temps réel

## Section 3 : Notes setter + Briefing IA
- Textarea libre (impressions brutes, objections, contexte)
- Bouton "✨ IA — Générer le briefing closer" → appel API Anthropic → résumé 5-6 lignes
- Fallback mock intelligent `generateMockBriefing()` si API indisponible
- Indicateur ✨ sur la carte kanban quand briefing existe

## Section 4 : Équipe
Setter/Closer assignés, Tentatives (X/20), Dernière action

---

# 10. CARTE KANBAN

## Affiché
Nom (cliquable → drawer), Trade + avis Google "23★" + source (📢/🧊), Score coloré (🟢🟠🔴), ● qualification (vert=rempli/gris=vierge), ✨ briefing IA, Badges (VSL✓/Hot/No-Show/Reschedule/💬X/20), ⚡Urgent, Dernière action en couleur (rouge>2j/orange>1j), Boutons (Fiche/SMS/Appeler), Boutons closer (Présent/No-Show sur Booked, Closé/Follow-up/Lost sur Call Done), Tags onboarding (💳CB/🗺GMB)

## Retiré
Téléphone, preview note, badge langue, badge Nurture, tag ville, label "score"

---

# 11. CALENDRIER

## Vue Admin
Onglets par membre (Tous + un par setter/closer). Grille semaine 30min (7h-20h). Stats bar. Calls cliquables → menu contextuel. Badges ✓/✗.

## Modal qualification de call
1. "Le prospect s'est-il présenté ?" → Oui / Non
2. Si Non-show : valider directement, pas d'impression demandée
3. Si Oui : résultat du call (setter: Qualifié/À recontacter | closer: Closé/Follow-up/Lost) + avis setter 🟢🟠🔴 + note optionnelle

---

# 12. AUTOMATIONS (14 FLOWS)

Flow map : lead_in → flow_lead_in_cold | no_contact → flow_no_contact_cold | nurture → flow_nurture_longterm | booked → flow_booked | noshow → flow_noshow | call_done → flow_call_done | follow_up → flow_follow_up | closed → flow_closed | lost → flow_lost | active_ob → flow_active_ob | checkin → flow_checkin | ob_form → flow_ob_form | launch_booked → flow_launch_booked | cb_declined → flow_cb_declined

Flow Builder : éditeur visuel drag & drop. Blocs : SMS, Email, Wait, Wait Until, Wait Before Event, Wait Reply, Condition (split A/B), Loop, End (protégé). Variables : {{firstname}}, {{trade}}, {{city}}, {{zoom_link}}, {{booking_link}}, etc.

---

# 13. NOTES (WIKI)

2 onglets (Entreprise : lecture seule employés / Mes notes : perso). Éditeur identique à l'email editor. Auto-save 800ms. Suppression via modal custom. Insertion lien via modal custom. Scrolls indépendants sidebar/éditeur. Titre modifiable sans re-render.

---

# 14. DATA MODEL LEAD

Identité : id, name, firstname, lastname, trade, city, phone, email
Pipeline : pipeline, stage, source, score
Tracking : contact_attempts, max_attempts, vsl_watched, _hasReplied, tags[], urgent, lastAction, lastActionDate
Assignation : assignedSetter, assignedCloser, assignedCSM
Google (CSV) : google_reviews, google_rating, website_url, gmaps_url
Links : booking_link, zoom_link, payment_link, form_link
Qualification : { setter_impression, jobs_per_month, client_source, is_owner, num_employees, website_quality }
Notes : quickNote, aiBriefing, notes[]

---

# 15. CUSTOM VALUES CLIENT

**Type A (agence)** : client_onboarding_form_link, company_name, gmb_access_instruction_video, launch_call_calendar_link, software_explanation_video, website_booking_link, website_link, website_our_work_link, website_testimonials_link

**Type B (client → site)** : Full Name, Business Phone, Business Name, Tax ID, Website URL, About You, Service Areas (max 14), Services, What Makes You Stand Out, Hours, Instagram, Facebook, BBB, TikTok, Yelp, Shipping Address, Return Customer Offer, Logo URL, Photos email

Bouton "Deploy to Site" pour injecter les values dans le template.

---

# 16. SCHÉMA DB SUPABASE SUGGÉRÉ

Tables : leads, clients, team_members, booked_calls, notes, automation_flows, automation_logs, actions

---

# 17. DÉCISIONS CLÉS PRISES EN SESSION

1. Le setter ne peut JAMAIS envoyer un lead en Lost — seul le closer a ce pouvoir
2. L'avis setter n'est demandé que si le lead s'est présenté (pas sur no-show)
3. Solo = positif (+10) — décideur unique, besoin urgent
4. Avis Google < 20 et rating < 4.2 = +20 chacun (cible idéale Spotrank)
5. Pas de scraping auto GMB pour leads Ads — trop peu fiable
6. Pas de last name dans le formulaire funnel
7. Formulaire Ads inclut website + GMB avec toggles + bouton "Find on Maps"
8. City et State séparés (dropdown 50 states)
9. Bouton "Note" devenu "Fiche" (ouvre le drawer complet)
10. Notes utilisent le même éditeur que les emails

---

# 18. PENDING / À FAIRE

**Haute priorité** :
- Supabase migration (variables JS → PostgreSQL + Realtime)
- Import CSV (interface upload + parsing + création leads)
- Runtime engine des flows (waits/conditions/loops — nécessite job scheduler backend)
- Détection reply SMS (webhook Twilio inbound → _hasReplied → recalcul score)
- Bouton "Chercher sur Google" dans le drawer

**Moyenne priorité** :
- Dashboard setter "Ma journée"
- Scoring decay (baisse auto si pas d'interaction)
- Confirmation call setter J-1
- VSL non-bookés → flow de recontact

**Basse priorité** :
- Intégration Calendly/Cal.com
- Notifications push
- App mobile
- Rapports mensuels PDF auto-générés

---

# 20. CORRECTIONS CRITIQUES APPLIQUÉES — 2026-04-17

## C1 — Protection XSS (innerHTML) ✅

### DOMPurify ajouté
```html
<!-- <head>, avant le premier <script> -->
<script src="https://cdn.jsdelivr.net/npm/dompurify@3.1.6/dist/purify.min.js"
  integrity="sha384-..." crossorigin="anonymous"></script>
```

### Helper safeHTML() créé (juste avant les données globales)
```js
function safeHTML(str) {
  if (str === null || str === undefined) return '';
  if (typeof DOMPurify !== 'undefined') {
    return DOMPurify.sanitize(String(str), { FORCE_BODY: false });
  }
  // Fallback minimal si CDN bloqué
  return String(str).replace(/&/g,'&amp;').replace(/</g,'&lt;')...
}
```

### Points d'injection sécurisés
- `renderNotes()` — `n.title` et `n.ownerId` firstname → `safeHTML()`
- `renderNoteEditor()` — `note.title` (attribut `value`), `note.content` → DOMPurify avec ALLOWED_TAGS restreints, `note.updatedAt` → `safeHTML()`
- `aiRewriteNote()` — réponse API Anthropic → `DOMPurify.sanitize()` avec ALLOWED_TAGS: ['br','strong','em','ul','li','p']
- `#ld-tags` — `lead.tags[]` → `safeHTML()` par tag
- `renderLeadNotes()` — `n.author`, `n.authorColor`, `n.date` → `safeHTML()`
- `escapeHtml()` — upgradé pour déléguer à `safeHTML()` (rétrocompatibilité)

### Règle à appliquer lors de toute nouvelle feature
Toute valeur issue d'un champ utilisateur (lead.name, note.title, tags, quickNote…) doit passer par `safeHTML()` avant d'être interpolée dans un template literal HTML.

---

## C2 — Guards rôle côté client ✅

### Helper currentUserCan() créé
```js
function currentUserCan(action) {
  const roles = currentUser?.member?.roles || [];
  switch (action) {
    case 'mark_lost':
    case 'mark_closed':
    case 'mark_followup': return isAdmin || isCloser;
    case 'mark_present':
    case 'mark_noshow':   return isAdmin || isCloser || isSetter;
    default: return false; // fail-safe
  }
}
```

### Guards appliqués dans les 5 fonctions closerMark*
- `closerMarkPresent('mark_present')` — setter/closer/admin
- `closerMarkNoShow('mark_noshow')` — setter/closer/admin
- `closerMarkClosed('mark_closed')` — **closer/admin seulement**
- `closerMarkFollowUp('mark_followup')` — closer/admin
- `closerMarkLost('mark_lost')` — **closer/admin seulement** (règle métier fondamentale)

### Bonus W1 corrigé en passant
Tous les `lead.lastActionDate = 'Just now'` remplacés par `new Date().toISOString()` dans ces 5 fonctions.

### À compléter en production
Doubler par des Supabase RLS policies sur la table `leads` (filtrage par JWT role).

---

## C3 — Emails @example.com supprimés ✅

### sendCampaignEmail() — filtre avant construction des recipients
```js
const EMAIL_REGEX = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
const recipients = batch
  .filter(c => c.email && EMAIL_REGEX.test(c.email))
  .map(c => ({ email: c.email, name: c.name }));
if (recipients.length === 0) continue;
```

### sendFlowEmail() — guard + log warning
```js
if (!lead.email || !EMAIL_REGEX.test(lead.email)) {
  console.warn(`sendFlowEmail: lead ${lead.id} ignoré — pas d'email valide`);
  return { success: false, error: 'No valid email address' };
}
```

### Impact
- Zéro bounce dur vers @example.com en production
- Réputation IP SendGrid préservée
- Leads sans email ciblés uniquement par SMS (flow SMS déjà protégé par guard `if (!lead.phone) return`)

---

# 21. FEATURE — ÉDITION SITE CLIENT DANS CLAUDE DESIGN

## Contexte
Les CSMs peuvent éditer les pages des sites clients directement depuis l'onglet "Site & Formulaires" de la fiche client, sans ouvrir un éditeur de code.

## Workflow utilisateur
1. Ouvrir la fiche d'un client → onglet "Site & Formulaires"
2. Cliquer **Éditer** sur une page buildée → bouton bleu "C Éditer"
3. Le dashboard :
   - Fetch le fichier HTML source (`template-01-prolocal.html` etc.)
   - Injecte un commentaire d'en-tête avec le contexte client (nom, trade, ville, date)
   - Copie le HTML dans le presse-papiers
   - Ouvre `claude.ai/design` dans un nouvel onglet
   - Affiche une modal 3 étapes : ✓ Copié → Ouvrir Claude Design → Coller + décrire les modifs
4. L'utilisateur colle dans Claude Design et décrit ses modifications

## GitHub par client
Champ `githubRepo` ajouté dans le modèle client (ex: `'spotrank-agency/elite-plumbing-co'`).
- Bandeau GitHub affiché dans Site & Formulaires : statut lié (vert) / non lié (gris)
- Clic → `openGithubModal(clientId)` → modal de configuration avec input + preview live
- `saveGithubRepo()` valide le format `org/repo`, nettoie les URLs github.com/, peut délier
- Clients préconfigurés : Elite Plumbing (`spotrank-agency/elite-plumbing-co`), Santos Electric (`spotrank-agency/santos-electric`)

## Nouvelles fonctions
- `renderSiteForms(c, body, context)` — rendu complet de l'onglet (refactorisé)
- `openGithubModal(clientId)` — modal de config repo GitHub
- `saveGithubRepo(repoOverride?)` — sauvegarde / déliaison du repo
- `openInClaudeDesign(clientId, pageName, fileKey)` — fetch HTML + clipboard + ouverture Design

## Nouvelles modals
- `#modal-github-repo` — config du dépôt GitHub (input + preview + délier)
- `#modal-claude-design` — guide 3 étapes pour coller dans Claude Design

## MAP pages → fichiers
| Page | fileKey |
|---|---|
| Home | `template-01-prolocal` |
| Optin / VSL | `optin` |
| Funnel Page 1 | `funnel-page-1` |
| Funnel Page 2 | `funnel-page-2` |
| Legal | `legal` |
| Privacy | `privacy` |
| Contact | `contact` |

## Notes techniques
- `fetch()` requiert un serveur HTTP (Netlify / localhost) — en `file://` le fallback affiche un toast d'instruction manuelle
- Tous les champs interpolés dans les templates HTML passent par `safeHTML()` (C1 maintenu)
- En prod : le fetch devra pointer vers le repo GitHub du client (API GitHub Raw ou Supabase Storage)

---

# 21. FEATURE — ÉDITION DANS CLAUDE DESIGN (2026-04-19)

## Objectif
Permettre d'éditer les sites des clients directement dans `claude.ai/design`, et stocker chaque site dans un dépôt GitHub privé de l'organisation `spotrank-agency`.

## Ce qui a été implémenté

### Modèle de données
- Champ `githubRepo` ajouté à tous les clients (ex: `'spotrank-agency/elite-plumbing-co'`)
- Valeur vide `''` = non lié, format attendu : `org/repo`

### Onglet "Site & Formulaires" — changements visuels

#### Bandeau GitHub (nouveau, sous le deploy bar)
- Vert + lien cliquable si repo lié
- Gris + bouton "+ Lier" si non lié
- Clic → `openGithubModal(clientId)` → modal de configuration

#### Bouton "Éditer" (remplace l'ancien bouton ghost)
- Style `.btn-claude-design` : CTA bleu #1a56e8 avec icône "C" (Claude)
- Visible uniquement sur les pages au statut `done`
- Onclick → `openInClaudeDesign(clientId, pageName, fileKey)`

### Fonctions JS ajoutées (toutes documentées)

#### `openInClaudeDesign(clientId, pageName, fileKey)`
1. Fetch le fichier HTML (`${fileKey}.html`) depuis le même dossier
2. Injecte un commentaire contextuel en haut (client, page, date)
3. Copie le HTML dans le presse-papiers via `navigator.clipboard`
4. Ouvre `claude.ai/design` dans un nouvel onglet
5. Affiche `modal-claude-design` avec les instructions en 3 étapes
6. **Fallback** si `fetch` échoue (file:// protocol) : toast informatif + ouverture quand même

#### `openGithubModal(clientId)`
- Pré-remplit l'input avec le repo existant
- Preview temps réel du lien GitHub pendant la saisie
- Bouton "Délier" visible uniquement si repo déjà lié

#### `saveGithubRepo(repoOverride?)`
- Valide le format `org/repo` (doit contenir `/`)
- Nettoie automatiquement les préfixes `https://github.com/`
- `null` = délier, omis = lire l'input
- Rafraîchit l'onglet automatiquement après sauvegarde

### Modals ajoutées (HTML)
- `#modal-github-repo` — config du repo GitHub
- `#modal-claude-design` — instructions 3 étapes (copié ✓ → ouvrir → coller)

### CSS ajouté
- `.github-bar` + `.github-bar.linked` — bandeau GitHub
- `.btn-claude-design` — bouton CTA bleu édition
- `.claude-design-icon` — petit badge "C" dans le bouton

### MAP des pages → fichiers
| Page | fileKey |
|---|---|
| Home | `template-01-prolocal` |
| Optin / VSL | `optin` |
| Funnel Page 1 | `funnel-page-1` |
| Funnel Page 2 | `funnel-page-2` |
| Legal | `legal` |
| Privacy | `privacy` |
| Contact | `contact` |

## Convention GitHub
- Organisation : `spotrank-agency` (privée)
- Format repo : `spotrank-agency/nom-du-client` (ex: `spotrank-agency/elite-plumbing-co`)
- Les fichiers HTML du client sont dans la branche `main` à la racine du repo
- Déploiement : Netlify connecté au repo → auto-deploy sur push `main`

---

# 22. INTÉGRATION GITHUB PAGES — 2026-04-19

## Ce qui a été implémenté

### Configuration
```js
const GITHUB_CONFIG = {
  org:          'Spotrank-agency',        // Organisation GitHub
  branch:       'main',                   // Branche de déploiement
  templateFile: 'template-01-prolocal',  // Template HTML par défaut
  apiBase:      'https://api.github.com',
};
```

### Fonctions GitHub (toutes documentées)
- `githubHeaders()` — headers Bearer avec PAT
- `githubRepoSlug(client)` — "Elite Plumbing Co." → `elite-plumbing-co`
- `githubFullRepo(client)` — `Spotrank-agency/elite-plumbing-co`
- `createClientRepo(clientId)` — crée le repo privé dans l'org
- `pushClientTemplate(clientId)` — push index.html avec données client injectées
- `enableGithubPages(clientId)` — active Pages sur branche main
- `setCustomDomain(clientId, domain)` — configure domaine custom
- `rollbackClientSite(clientId)` — revient au commit précédent
- `deployClientSite(clientId)` — orchestre les 3 étapes en séquence

### Workflow de déploiement complet
1. Clic "Créer & Déployer" → `deployClientSite(id)`
2. Crée repo `Spotrank-agency/nom-client` (privé)
3. Push `index.html` avec en-tête contextuel (client, date, repo)
4. Active GitHub Pages → URL : `https://spotrank-agency.github.io/nom-client/`
5. Si domaine custom → `setCustomDomain()` → met à jour `c.siteUrl`

### UI
- Bouton **"Créer & Déployer"** (premier deploy) ou **"Redéployer"** (update)
- Bouton **"↩ Rollback"** visible si repo lié
- Champ GitHub Token dans Settings → Intégrations & API (🐙)

---

## DETTE TECHNIQUE — localStorage API Keys

**État actuel** : Les clés API (GitHub token, Twilio, SendGrid, Anthropic…)
sont persistées dans `localStorage` sous la clé `spotrank_api_keys`.

**Fonctions concernées** :
- `saveAllApiKeys()` → écrit dans localStorage
- `loadApiKeys()` → lit depuis localStorage au boot
- `clearApiKeys()` → efface tout (bouton "Effacer tout" dans Settings)

**Risques** :
- Accessible à tout script JS sur la page
- Données sur un seul appareil (pas multi-device)
- Pas chiffré côté serveur

**TODO-SUPABASE — PRIORITÉ HAUTE** :
Avant la mise en production multi-utilisateur, migrer vers **Supabase Vault** :
```js
// Remplacer dans saveAllApiKeys() :
await supabase.rpc('set_secret', { name: 'github_token', value: token });

// Remplacer dans loadApiKeys() :
const { data } = await supabase.rpc('get_secrets');
```

**Marqueurs dans le code** : chercher `TODO-SUPABASE` dans `spotrank-dashboard.html`

---

## GitHub Team — Infrastructure

- **Organisation** : `Spotrank-agency` (créée le 2026-04-19)
- **Plan** : Free (à upgrader en Team $4/mois pour repos privés + GitHub Pages)
- **Token** : PAT classique, scopes `repo` + `workflow` + `write:org` + `pages`
- **Stockage token** : localStorage (TODO-SUPABASE)
- **Coût hébergement** : $0 pour tous les sites clients
- **URL par défaut** : `https://spotrank-agency.github.io/nom-client/`
- **URL custom** : configurée via `setCustomDomain()` + DNS CNAME → `spotrank-agency.github.io`

---

# 23. FEATURE — FLOWS CLIENTS PAR MÉTIER — 2026-04-20

## Vue d'ensemble
Système complet de flows SMS/email automatiquement assignés à chaque client selon son métier, avec éditeur de templates dans Settings et gestion manuelle par client.

## Données — CLIENT_FLOW_TEMPLATES
Bibliothèque globale de templates de flows, organisée par métier :
- `_all` — Flows communs à tous les métiers (3 flows : avis Google, check-in mensuel, réactivation)
- `Plumber` — 2 flows spécifiques (urgence + alerte hiver)
- `HVAC` — 2 flows (maintenance AC été + chauffage hiver)
- `Roofer` — 2 flows (dommages tempête + inspection annuelle)
- `Electrician` — 2 flows (promo borne EV + bilan sécurité)
- `Landscaper` — 2 flows (nettoyage printemps + automne)

## Modèle client — champ flows[]
Chaque client a un tableau `flows[]` :
```js
{
  id:       'boot_1_abc12',  // unique — préfixe: boot/manual/timestamp
  name:     'Demande d\'avis Google',
  icon:     '⭐',
  trigger:  'Après prestation terminée (J+1)',
  channel:  'sms',            // 'sms' | 'email' | 'both'
  steps:    [{ delay, type, content }],
  active:   true,
  sent:     0,
  openRate: '—',
  addedAt:  '2026-04-20T...',
  source:   'auto',           // 'auto' = métier | 'manual' = ajouté manuellement
  fromTemplate: true,
}
```

## Fonctions clés
- `getFlowsForTrade(trade)` — fusionne flows _all + flows spécifiques au métier
- `resolveFlowVars(content, client)` — résout {{client_name}}, {{city}}, {{booking_link}}, {{gmb_link}}…
- `initClientFlows()` — auto-init au boot pour les clients existants (IIFE)
- `addClient()` — modifié pour auto-assigner les flows au métier + mettre à jour automationsActive
- `renderClientAutomations(c, body)` — refaite entièrement, même structure que la page agence
- `toggleClientFlow(clientId, flowId, active)` — on/off sur un flow client
- `removeClientFlow(clientId, flowId)` — suppression avec modal de confirmation custom
- `confirmRemoveClientFlow(clientId, flowId)` — exécute la suppression après confirmation
- `openAddFlowModal(clientId)` — modal de sélection de flow manuel (tous les templates filtrés)
- `addFlowToClient(templateId, tradeGroup)` — ajoute un flow manuellement avec source:'manual'
- `renderClientFlowsTab(wrapper)` — Settings → Flows clients (vue par métier + toggle par défaut)
- `toggleFlowTemplate(tradeKey, flowName, active)` — active/désactive un template dans la bibliothèque

## Settings — Onglet "Flows clients"
Nouvel onglet dans Settings > sidenav "Flows clients" :
- Vue par métier avec tous les templates
- Toggle "activé par défaut" par flow (affecte les nouveaux clients uniquement)
- Aperçu des steps avec délai + contenu tronqué
- Info : pour modifier le contenu → Flow Builder

## Variables résolues dans les steps
| Variable | Valeur |
|---|---|
| {{client_name}} | client.bizName |
| {{city}} | client.city |
| {{trade}} | client.trade |
| {{phone}} | client.phone |
| {{booking_link}} | customA.launch_call_calendar_link ou AGENCY_VARS |
| {{gmb_link}} | customB.current_website_url |
| {{report_link}} | https://spotrank.io/rapport/{id} |
