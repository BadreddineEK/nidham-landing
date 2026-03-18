# PROMPT GITHUB COPILOT — Mosquée Manager MVP

Tu es un assistant de développement Django expert. Tu vas implémenter un système de gestion pour mosquée (école coranique + cotisations + trésorerie) suivant la spec ci-jointe.

## Contexte projet
- Client : Association mosquée (Meximieux, France)
- Objectif : remplacer Google Sheets par un outil pro, zéro coût mensuel au départ, migratable vers cloud sans réécriture
- Timeline : MVP fonctionnel avril 2026
- Déploiement : Docker Compose (local/mosquée), accès via Cloudflare Tunnel
- Données : sensibles (enfants, parents, finances) → sécurité minimale obligatoire

## Stack imposée
- Python 3.12
- Django 5.x + Django REST Framework
- PostgreSQL 16
- Docker + Docker Compose
- Configuration : `.env` uniquement (aucune valeur en dur)

## Principes architecturaux NON NÉGOCIABLES

### 1. Multi-mosquée dès le jour 1
- Table `Mosque` + colonne `mosque_id` dans TOUTES les tables métier
- Middleware RBAC : vérifier `user.mosque_id == object.mosque_id` à chaque requête
- Aucun accès cross-mosquée possible

### 2. Configuration modulable (pas de valeurs figées)
- Table `MosqueSettings` : tarifs, niveaux école, règles cotisations
- Panneau admin "Paramètres" : modifier config sans toucher au code
- Onboarding : formulaire initial 5 champs obligatoires (nom mosquée, timezone, année scolaire, niveaux, tarif défaut)

### 3. Séparation PII / KPI
- Endpoint KPI : `/api/kpi/summary?mosque=slug` → uniquement agrégats (totaux, compteurs)
- Aucune donnée nominative (noms, tél, email, adresse) dans les KPI
- Page écran : `/kpi-screen/{mosque_slug}` auto-refresh 30–60s, read-only

### 4. Migrations Django (schéma versionné)
- Toute modification modèle → migration
- Pas de manipulation directe SQL en prod

### 5. Même code partout (local → cloud)
- Seule l'infra change (DB, domaine, secrets)
- Code applicatif identique

## Modèle de données minimal (à implémenter)

### core/models.py
```python
class Mosque(models.Model):
    name = models.CharField(max_length=200)
    slug = models.SlugField(unique=True)
    timezone = models.CharField(max_length=50, default='Europe/Paris')
    created_at = models.DateTimeField(auto_now_add=True)

class MosqueSettings(models.Model):
    mosque = models.OneToOneField(Mosque, on_delete=models.CASCADE, related_name='settings')
    school_levels = models.JSONField(default=list)  # ex: ["NP","N1","N2"...]
    school_fee_default = models.DecimalField(max_digits=10, decimal_places=2)
    school_fee_mode = models.CharField(max_length=20, choices=[('annual','Annual'), ('monthly','Monthly')])
    membership_fee_amount = models.DecimalField(max_digits=10, decimal_places=2)
    membership_fee_mode = models.CharField(max_length=20, choices=[('per_person','Per Person'), ('per_family','Per Family')])
    active_school_year_label = models.CharField(max_length=50)  # ex: "2025-2026"

class User(AbstractUser):
    mosque = models.ForeignKey(Mosque, on_delete=models.CASCADE, null=True)
    role = models.CharField(max_length=20, choices=[('ADMIN','Admin'), ('TRESORIER','Trésorier'), ('ECOLE_MANAGER','École Manager')])

class AuditLog(models.Model):
    mosque = models.ForeignKey(Mosque, on_delete=models.CASCADE)
    user = models.ForeignKey(User, on_delete=models.SET_NULL, null=True)
    action = models.CharField(max_length=50)  # ex: "create_payment", "import_excel"
    entity = models.CharField(max_length=50)  # ex: "school_payment", "member"
    entity_id = models.IntegerField(null=True)
    payload = models.JSONField(default=dict)
    created_at = models.DateTimeField(auto_now_add=True)
```

### school/models.py
```python
class Family(models.Model):
    mosque = models.ForeignKey('core.Mosque', on_delete=models.CASCADE)
    primary_contact_name = models.CharField(max_length=200)
    email = models.EmailField(blank=True, null=True)
    phone1 = models.CharField(max_length=20, blank=True)
    phone2 = models.CharField(max_length=20, blank=True)
    address = models.TextField(blank=True)
    created_at = models.DateTimeField(auto_now_add=True)

class Child(models.Model):
    mosque = models.ForeignKey('core.Mosque', on_delete=models.CASCADE)
    family = models.ForeignKey(Family, on_delete=models.CASCADE, related_name='children')
    first_name = models.CharField(max_length=100)
    birth_date = models.DateField(null=True, blank=True)
    level = models.CharField(max_length=20)  # ex: "N1", "N2"...
    created_at = models.DateTimeField(auto_now_add=True)

class SchoolYear(models.Model):
    mosque = models.ForeignKey('core.Mosque', on_delete=models.CASCADE)
    label = models.CharField(max_length=50)  # ex: "2025-2026"
    start_date = models.DateField()
    end_date = models.DateField()
    is_active = models.BooleanField(default=True)

class SchoolPayment(models.Model):
    mosque = models.ForeignKey('core.Mosque', on_delete=models.CASCADE)
    school_year = models.ForeignKey(SchoolYear, on_delete=models.CASCADE)
    family = models.ForeignKey(Family, on_delete=models.CASCADE)
    child = models.ForeignKey(Child, on_delete=models.SET_NULL, null=True, blank=True)
    date = models.DateField()
    method = models.CharField(max_length=20, choices=[('virement','Virement'), ('cheque','Chèque'), ('cb','CB'), ('especes','Espèces')])
    amount = models.DecimalField(max_digits=10, decimal_places=2)
    note = models.TextField(blank=True)
    created_at = models.DateTimeField(auto_now_add=True)
```

### membership/models.py
```python
class MembershipYear(models.Model):
    mosque = models.ForeignKey('core.Mosque', on_delete=models.CASCADE)
    year = models.IntegerField()  # ex: 2025
    amount_expected = models.DecimalField(max_digits=10, decimal_places=2)

class Member(models.Model):
    mosque = models.ForeignKey('core.Mosque', on_delete=models.CASCADE)
    full_name = models.CharField(max_length=200)
    email = models.EmailField(blank=True, null=True)
    phone = models.CharField(max_length=20, blank=True)
    address = models.TextField(blank=True)
    created_at = models.DateTimeField(auto_now_add=True)

class MembershipPayment(models.Model):
    mosque = models.ForeignKey('core.Mosque', on_delete=models.CASCADE)
    membership_year = models.ForeignKey(MembershipYear, on_delete=models.CASCADE)
    member = models.ForeignKey(Member, on_delete=models.CASCADE)
    date = models.DateField()
    method = models.CharField(max_length=20, choices=[('virement','Virement'), ('cheque','Chèque'), ('cb','CB'), ('especes','Espèces')])
    amount = models.DecimalField(max_digits=10, decimal_places=2)
    note = models.TextField(blank=True)
    created_at = models.DateTimeField(auto_now_add=True)
```

### treasury/models.py
```python
class TreasuryTransaction(models.Model):
    mosque = models.ForeignKey('core.Mosque', on_delete=models.CASCADE)
    date = models.DateField()
    category = models.CharField(max_length=50)  # ex: "cotisations", "education", "irchad", "projets"...
    label = models.CharField(max_length=200)
    direction = models.CharField(max_length=10, choices=[('in','Entrée'), ('out','Sortie')])
    amount = models.DecimalField(max_digits=10, decimal_places=2)
    payment_method = models.CharField(max_length=20)
    note = models.TextField(blank=True)
    created_at = models.DateTimeField(auto_now_add=True)
```

## API à implémenter (DRF)

### Auth
- `POST /api/auth/login` → token (ou session Django)
- `POST /api/auth/logout`

### Settings (ADMIN only)
- `GET /api/settings/` → lecture config mosquée
- `PUT /api/settings/` → modification (tarifs, niveaux, règles)

### School
- `GET/POST /api/school/families/`
- `GET/PUT/DELETE /api/school/families/{id}/`
- `GET/POST /api/school/children/`
- `GET/POST /api/school/payments/`
- `POST /api/school/import` (upload Excel)
- `GET /api/school/export` (CSV)
- `GET /api/school/arrears` (liste impayés)

### Membership
- `GET/POST /api/membership/members/`
- `GET/POST /api/membership/payments/`
- `POST /api/membership/import`
- `GET /api/membership/export`

### Treasury
- `GET/POST /api/treasury/transactions/`
- `POST /api/treasury/import`

### KPI (read-only, no PII)
- `GET /api/kpi/summary?mosque=slug`
  Retourne JSON :
  ```json
  {
    "total_children": 68,
    "children_by_level": {"N1": 12, "N2": 15, ...},
    "school_amount_due": 20400.00,
    "school_amount_paid": 14200.00,
    "membership_paid_count": 78,
    "membership_unpaid_count": 17,
    "treasury_month_in": 12450.00,
    "treasury_month_out": 8230.00
  }
  ```

## Imports Excel (stratégie)

### school/import
- Colonnes attendues : `Nom famille`, `Téléphone`, `Email`, `Prénom enfant`, `Niveau`, `Date paiement` (opt), `Montant` (opt)
- Normalisation : téléphones (format uniforme), emails (lowercase)
- Idempotence : si `primary_contact_name + phone1` existent → skip ou update (param)
- Log erreurs + rapport import (nb OK/KO)

### membership/import
- Colonnes : `Nom`, `Prénom`, `Téléphone`, `Email`, `Montant annuel`, `Payé`, `Date paiement`

### treasury/import
- Colonnes : `Date`, `Type`, `Catégorie`, `Objet`, `Montant`, `Mode`

## UI admin (choix)

**Option A (recommandée MVP)** : Django Admin + quelques pages custom
- Onboarding : page `/onboarding` (POST form → create Mosque + Settings)
- Panneau Paramètres : page `/settings` (GET/POST form → update MosqueSettings)
- KPI écran : page `/kpi-screen/{mosque_slug}` (template HTML simple, auto-refresh JS)

**Option B** : Frontend séparé (React/Next)
- Si choisi : créer API complète + frontend indépendant

## Sécurité minimale

- Middleware RBAC : vérifier `request.user.mosque_id == obj.mosque_id` sur chaque requête
- Cloudflare Access : whitelist emails (config externe, pas dans le code)
- KPI endpoint : aucune donnée nominative (filtrer côté serializer)
- AuditLog : logger création/modif paiements, imports, exports

## Backups

- Conteneur `backup` dans docker-compose
- Script cron : `pg_dump` quotidien → chiffrement AES → copie vers S3 ou local
- Variable `.env` : `BACKUP_PASSPHRASE`, `BACKUP_TARGET` (s3/local)

## Docker Compose (structure attendue)

```yaml
version: '3.8'
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data

  backend:
    build: ./backend
    command: python manage.py runserver 0.0.0.0:8000
    environment:
      DATABASE_URL: ${DATABASE_URL}
      DJANGO_SECRET_KEY: ${DJANGO_SECRET_KEY}
      DJANGO_DEBUG: ${DJANGO_DEBUG}
    depends_on:
      - db
    ports:
      - "8000:8000"

  cloudflared:
    image: cloudflare/cloudflared:latest
    command: tunnel run
    environment:
      TUNNEL_TOKEN: ${CLOUDFLARE_TUNNEL_TOKEN}
    depends_on:
      - backend

  backup:
    image: postgres:16-alpine
    command: /backup.sh
    environment:
      DATABASE_URL: ${DATABASE_URL}
      BACKUP_PASSPHRASE: ${BACKUP_PASSPHRASE}
      BACKUP_TARGET: ${BACKUP_TARGET}
    volumes:
      - ./backups:/backups
      - ./scripts/backup.sh:/backup.sh

volumes:
  postgres_data:
```

## Checklist développement (ordre strict)

1. [ ] Init repo + Django project + DRF + docker-compose + Postgres (test connexion)
2. [ ] Models core (Mosque, MosqueSettings, User, AuditLog) + migrations
3. [ ] Auth API (login/logout) + middleware RBAC
4. [ ] Models school (Family, Child, SchoolYear, SchoolPayment) + migrations
5. [ ] API CRUD school (families, children, payments)
6. [ ] Import Excel école (version simple)
7. [ ] Models membership (Member, MembershipYear, MembershipPayment) + migrations
8. [ ] API CRUD membership + import
9. [ ] Models treasury (TreasuryTransaction) + migrations + API
10. [ ] Onboarding : page `/onboarding` (form + création mosquée)
11. [ ] Panneau Paramètres : page `/settings` (modif config)
12. [ ] KPI endpoint : `/api/kpi/summary` (agrégats uniquement)
13. [ ] KPI écran : page `/kpi-screen/{slug}` (auto-refresh)
14. [ ] Cloudflare Tunnel config (doc + test)
15. [ ] Backup automatique (cron + chiffrement) + test restore
16. [ ] Documentation README + guide déploiement mosquée
17. [ ] Tests end-to-end avec données anonymisées

## Sorties attendues (pour chaque commit/PR)

**Format de commit** :
```
[MODULE] Description courte

- Changement 1
- Changement 2

Status: [EN_COURS | TERMINÉ | BLOQUÉ]
Prochaine étape: [description]
```

**À chaque fin de tâche majeure, fournis** :
1. Résumé de ce qui a été fait (3–5 lignes)
2. Fichiers modifiés (liste)
3. Commandes pour tester (docker-compose up, migrations, etc.)
4. Blocages éventuels
5. Prochaine tâche recommandée

**Exemple de sortie** :
```
✅ TERMINÉ : Models core + migrations

Fichiers :
- backend/core/models.py (Mosque, MosqueSettings, User, AuditLog)
- backend/core/migrations/0001_initial.py
- backend/core/admin.py (enregistrement Django Admin)

Commandes test :
$ docker-compose up -d db
$ docker-compose run backend python manage.py migrate
$ docker-compose run backend python manage.py createsuperuser

Status : TERMINÉ
Prochaine étape : Auth API (login/logout) + middleware RBAC
```

## Contraintes de code

- Docstrings en français pour les modèles/vues
- Type hints Python (PEP 484)
- Tests unitaires minimaux (au moins 1 test par endpoint critique)
- Pas de `print()` en prod (utiliser `logging`)
- Validation côté serializer (DRF) + côté modèle (clean methods)

## Questions fréquentes (anticiper)

**Q : Multi-mosquée = base partagée ou bases séparées ?**
R : Base partagée, isolation par `mosque_id` via middleware.

**Q : UI admin = Django Admin ou frontend custom ?**
R : Django Admin + pages custom (option A) pour MVP avril. Frontend séparé plus tard si besoin.

**Q : Cloudflare Tunnel = comment tester en local ?**
R : `cloudflared tunnel --url http://localhost:8000` (tunnel temporaire) ou token permanent (config dans compose).

**Q : Backups = où stocker ?**
R : MVP = local (volume monté) + copie manuelle. Future = S3 auto.

## Contexte métier (pour comprendre les données)

- **École** : enfants inscrits, familles, paiements mensuels ou annuels (300€/an en moyenne)
- **Cotisations** : adhésion mosquée (150€/an par personne ou par famille)
- **Trésorerie** : toutes les transactions (dons, charges, salaires imam, travaux...)
- **Fonds** : catégories comptables (cotisations, éducation, irchad, projets, zakat, événementiel...)
- **KPI** : indicateurs pour écran TV mosquée (affichage public, aucune donnée sensible)

---

**IMPORTANT** : Référer à `ROADMAP-MOSQUE-MVP.md` pour les détails complets.

Commence par la checklist étape 1 : init repo + Django + Postgres + docker-compose.

Fournis la sortie selon le format indiqué ci-dessus.