# 🕌 Mosquée Manager — Feuille de route MVP Avril 2026

## Vision produit
Remplacer les Google Sheets par un outil professionnel **zéro coût mensuel au départ**, capable d'évoluer vers le cloud sans réécriture.

**Périmètre MVP Avril :**
- Admin : École (familles/enfants/paiements) + Cotisations (adhérents/paiements) + Trésorerie (lecture/imports)
- Écran KPI : affichage read-only mosquée (TV/tablette), aucune donnée personnelle
- Backups automatiques chiffrés
- Accès sécurisé depuis partout (Cloudflare Tunnel + Access)

**Hors périmètre MVP :**
- Paiement en ligne
- App adhérents/familles
- Multi-mosquées centralisées
- Notifications automatiques

---

## Stack technique

**Backend**
- Python 3.12 + Django 5.x + Django REST Framework
- PostgreSQL 16 (dès le jour 1, même en local)
- Docker + Docker Compose (même runtime partout)

**Déploiement**
- Local/mosquée : Docker Compose sur Pi4/PC
- Accès distant : Cloudflare Tunnel (gratuit) + Cloudflare Access (auth email)
- Future cloud : même code, DB managée (Render/Supabase)

**Backups**
- `pg_dump` quotidien + chiffrement AES + stockage externe (S3/local)

---

## Principes non négociables

### 1. Configuration via `.env` uniquement
Aucune valeur en dur. Toute config (secrets, URLs, limites) dans `.env`.

### 2. Multi-mosquée dès maintenant
- Table `Mosque` + colonne `mosque_id` partout
- Même si une seule mosquée au début, le schéma supporte multi-tenant

### 3. Séparation PII / KPI
- KPI = uniquement agrégats (totaux, compteurs, moyennes)
- Jamais de noms/tél/email/adresse sur l'écran KPI

### 4. Migrations Django (schéma stable)
- Toute modification du modèle = migration versionnée
- Facilite la montée de version et la reprise de données

### 5. Même code, mêmes services (local → cloud)
- Seule l'infra change (DB locale → DB managée, compose → cloud run)
- Le code reste identique

### 6. Modulable dès la conception
- Paramètres mosquée (tarifs, niveaux, règles) = **panneau de configuration** (pas de valeurs figées en base)
- Extension future : tables optionnelles (documents, autorisations, champs custom)

---

## Modèle de données minimal (MVP)

### core
- `Mosque(id, name, slug, timezone, created_at)`
- `MosqueSettings(id, mosque_id, school_levels[], school_fee_default, school_fee_mode, membership_fee_amount, membership_fee_mode, active_school_year_label, ...)`
- `User(id, email, password_hash, role, mosque_id, is_active, created_at)`
  - role ∈ {ADMIN, TRESORIER, ECOLE_MANAGER}
- `AuditLog(id, mosque_id, user_id, action, entity, entity_id, payload_json, created_at)`

### school
- `Family(id, mosque_id, primary_contact_name, email, phone1, phone2, address, created_at)`
- `Child(id, mosque_id, family_id, first_name, birth_date, level, created_at)`
- `SchoolYear(id, mosque_id, label, start_date, end_date, is_active)`
- `SchoolPayment(id, mosque_id, school_year_id, family_id, child_id nullable, date, method, amount, note)`

### membership
- `MembershipYear(id, mosque_id, year, amount_expected)`
- `Member(id, mosque_id, full_name, email, phone, address, created_at)`
- `MembershipPayment(id, mosque_id, membership_year_id, member_id, date, method, amount, note)`

### treasury
- `TreasuryTransaction(id, mosque_id, date, category, label, direction[in/out], amount, payment_method, note)`

---

## Écrans & fonctionnalités (MVP)

### 1. Onboarding (première connexion admin)
Formulaire rapide (5 champs obligatoires) :
- Nom mosquée
- Fuseau horaire (défaut: Europe/Paris)
- Année scolaire active (label, ex: 2025–2026)
- Niveaux école (liste, ex: NP,N1,N2…N6)
- Tarif école par défaut (montant annuel ou mensuel)

### 2. Panneau Paramètres (modifiable à tout moment, rôle ADMIN uniquement)
Sections :
- **Mosquée** : nom, logo (futur), timezone
- **École** : année active, niveaux, tarifs, règles (réductions famille, etc.)
- **Cotisations** : montant annuel, par personne/par famille, période
- **KPI** : quels indicateurs afficher, objectifs (optionnel)
- **Utilisateurs** : ajouter/retirer admin, modifier rôles, réinitialiser MDP

### 3. Admin École
- CRUD familles (nom, contact principal, tél, email, adresse optionnelle)
- CRUD enfants (prénom, date naissance, niveau, famille liée)
- Enregistrer paiements (date, montant, mode, note)
- Recherche/filtres (par niveau, impayés, nom)
- Import Excel initial (mapping colonnes → champs)
- Export CSV (liste familles + suivi paiements)

### 4. Admin Cotisations
- CRUD adhérents (nom, tél, email, adresse optionnelle)
- Enregistrer paiements (date, montant, mode, note)
- Recherche/filtres (à jour, en retard)
- Import Excel
- Export CSV

### 5. Admin Trésorerie (lecture + imports)
- Liste transactions (date, catégorie, label, direction, montant, mode)
- Filtres par fonds (cotisations, école, irchad, projets…)
- Import Excel (transactions existantes)
- Export CSV

### 6. KPI Écran (read-only, aucun PII)
- Endpoint `/api/kpi/summary?mosque=slug`
- Agrégats :
  - Total enfants, répartition par niveau
  - École : dû / payé / reste
  - Cotisations : nb à jour / nb en retard
  - Trésorerie : entrées/sorties du mois
- Page `/kpi-screen/{mosque_slug}` auto-refresh (30–60s), design simple full-screen

---

## API (Django REST Framework)

**Auth**
- POST `/api/auth/login` → token/session
- POST `/api/auth/logout`

**RBAC**
- Middleware : vérifier rôle + `mosque_id` (aucun accès cross-mosquée)

**School**
- CRUD `/api/school/families/`, `/api/school/children/`, `/api/school/payments/`
- POST `/api/school/import` (Excel)
- GET `/api/school/export` (CSV)
- GET `/api/school/arrears` (impayés)

**Membership**
- CRUD `/api/membership/members/`, `/api/membership/payments/`
- POST `/api/membership/import`
- GET `/api/membership/export`

**Treasury**
- GET/POST `/api/treasury/transactions/`
- POST `/api/treasury/import`

**KPI**
- GET `/api/kpi/summary?mosque=slug` (agrégats uniquement)

**Settings**
- GET/PUT `/api/settings/` (panneau config mosquée, ADMIN only)

---

## Imports Excel (stratégie MVP)

### École
- Colonnes attendues : Nom famille, Téléphone, Email, Prénom enfant, Niveau, Date paiement (optionnel), Montant (optionnel)
- Normalisation : téléphones (format uniforme), emails (lowercase)
- Idempotence : si nom famille + téléphone existent → skip ou update (paramétrable)
- Log erreurs + rapport import (nb lignes OK/KO)

### Cotisations
- Colonnes : Nom, Prénom, Téléphone, Email, Montant annuel, Payé (Oui/Non), Date paiement
- Même normalisation + idempotence

### Trésorerie
- Colonnes : Date, Type (entrée/sortie), Catégorie/Fonds, Objet, Montant, Mode
- Validation : date valide, montant > 0, catégorie dans liste autorisée

---

## Backups (MVP)

**Automatisation**
- Conteneur `backup` dans docker-compose
- Cron quotidien : `pg_dump` → fichier daté
- Chiffrement AES avec passphrase (`.env`)
- Copie vers S3/local selon config

**Restore**
- Commande documentée : `docker exec postgres psql < backup.sql`
- Test de restauration obligatoire avant livraison

---

## Sécurité minimale

- HTTPS via Cloudflare Tunnel
- Cloudflare Access : whitelist emails autorisés (admin mosquée)
- RBAC côté Django : vérification rôle + `mosque_id` à chaque requête
- KPI sans PII (uniquement agrégats)
- Logs audit : création/modif paiements, imports, exports
- Pas de stockage browser (`localStorage` interdit en iframe sandbox)

---

## Définition "Done" pour avril

- [ ] Admin accessible via URL Cloudflare, login OK
- [ ] Onboarding : saisie config initiale (5 champs)
- [ ] Panneau Paramètres : modification config mosquée (tarifs, niveaux)
- [ ] Import Excel école + cotisations fonctionnels (1 fichier réel testé)
- [ ] CRUD familles/enfants + enregistrement paiements
- [ ] CRUD adhérents + paiements cotisations
- [ ] Consultation impayés école (filtre)
- [ ] KPI écran : affichage continu, lecture seule, données cohérentes
- [ ] Backup quotidien configuré + test restore effectué une fois
- [ ] Documentation : README (install/run), guide déploiement mosquée

---

## Checklist tâches (ordre chronologique)

### Phase 1 : Fondations (Semaine 1)
1. Repo structure + `docker-compose.yml` + `.env.example`
2. Django project + DRF + Postgres (test connexion)
3. Modèles `Mosque`, `MosqueSettings`, `User`, migrations
4. Auth basique (login/logout) + middleware RBAC

### Phase 2 : École (Semaine 2)
5. Modèles `Family`, `Child`, `SchoolYear`, `SchoolPayment` + migrations
6. API CRUD familles/enfants
7. API enregistrement paiements école
8. Import Excel école (version simple, 1 fichier test)
9. Filtre impayés (endpoint dédié)

### Phase 3 : Cotisations (Semaine 3)
10. Modèles `Member`, `MembershipYear`, `MembershipPayment` + migrations
11. API CRUD adhérents + paiements
12. Import Excel cotisations

### Phase 4 : Config & KPI (Semaine 4)
13. Onboarding : formulaire initial (5 champs)
14. Panneau Paramètres : écran modif config (API + UI)
15. KPI endpoints : `/api/kpi/summary` (agrégats)
16. Page KPI écran : `/kpi-screen/{slug}` auto-refresh

### Phase 5 : Déploiement (Semaine 5)
17. Cloudflare Tunnel + Access (doc + test)
18. Backup automatique (cron + chiffrement) + test restore
19. Documentation README + guide mosquée
20. Test end-to-end avec données réelles anonymisées

---

## UI admin : choix rapide

**Option A (recommandée MVP)** : Django Admin + quelques pages custom (formulaires simples)
- Avantage : livraison rapide, robuste, 0 frontend complexe
- Inconvénient : UI "basique" (mais fonctionnelle)

**Option B** : Frontend dédié (React/Next)
- Avantage : UI moderne, expérience optimale
- Inconvénient : +2 semaines dev, complexité accrue

**Décision finale** : toi (mais A = MVP avril garanti).

---

## Variables `.env` (exemple complet)

```bash
# Django
DJANGO_SECRET_KEY=xxx
DJANGO_DEBUG=false
ALLOWED_HOSTS=localhost,*.trycloudflare.com,mosque.example.com

# Database
DATABASE_URL=postgres://user:pass@db:5432/mosque_db

# Mosquée (défaut onboarding)
DEFAULT_MOSQUE_SLUG=meximieux

# Auth initiale
ADMIN_EMAIL=admin@mosquee.fr
ADMIN_PASSWORD=ChangeMe123!

# Cloudflare Tunnel
CLOUDFLARE_TUNNEL_TOKEN=xxx

# Backups
BACKUP_PASSPHRASE=xxx
BACKUP_TARGET=s3
AWS_ACCESS_KEY_ID=xxx
AWS_SECRET_ACCESS_KEY=xxx
AWS_BUCKET_NAME=mosquee-backups

# Timezone
TIMEZONE=Europe/Paris
```

---

## Migration future (quand ça devient "pro")

**Étape 1** : DB managée
- Export `pg_dump` → import vers Render/Supabase Postgres
- Update `DATABASE_URL` dans `.env`
- Bénéfice : PITR, backups gérés, scaling auto

**Étape 2** : API + Web sur cloud
- Build Docker image → push registry
- Deploy sur Render/Fly/Railway
- Tunnel Cloudflare devient optionnel (domaine direct)

**Étape 3** : KPI écran comme client léger
- Kiosque reste sur Raspberry Pi (ou tablet)
- Affiche `/kpi-screen/{slug}` depuis API cloud

**Coût estimé passage cloud** : 10–30€/mois selon usage (DB + compute).

---

## Contacts & responsabilités

**Dev** : Badreddine (toi)
**Référent mosquée** : [à compléter après onboarding]
**Validation fonctionnelle** : bureau mosquée
**Support technique** : toi (phase MVP), puis éventuel partage avec association

---

**Dernière mise à jour** : 18 février 2026
**Version** : 1.0 (pré-dev)
