# VarQi-PreRegistration

Istruzioni Claude Code per il sotto-progetto `VarQi-PreRegistration/` — submodule di `https://github.com/BAX-IoT/VarQI-PreRegistration.git`.

> Questo file vive nella root del monorepo. Non creare `CLAUDE.md` né `docs/` dentro il submodule.

## Project Overview

VarQi PreRegistration è il portale fornitori: un'applicazione Laravel 13 dove trasportatori e fornitori registrano in anticipo i trasporti in ingresso e in uscita dall'impianto (pre-registrazioni), rispondono alle proposte di trasporto e caricano i certificati. Espone inoltre le API OAuth2 consumate da VarQi-Internal. Interfaccia bilingue IT/EN (`lang/it`, `lang/en`), locale primario italiano.

## Commands

```bash
# Install dependencies
composer install
npm install

# Development servers
php artisan serve          # Backend
npm run dev                # Frontend (Vite)

# Production build
npm run build

# Database
php artisan migrate
php artisan migrate:fresh --seed   # Reset and seed

# Tests
./vendor/bin/phpunit                                # All tests
./vendor/bin/phpunit --filter=TestClassName          # Single test class

# OAuth2 (Passport)
php artisan passport:client --client --name="VarQI Sync"   # Machine-to-machine client for Internal
```

## Architecture

### Core Domain Flow

1. **Pre-registrations** — A supplier books a transport (materials, plates, dates, zone). Lifecycle in `PreRegistrationStatus`: Draft(1) → Reserved(2) → Sent(3) → Processed(4) → Arrived(5) → Entered(6) → Done(7)
2. **Validation** — Staff approve, reject or suspend a pre-registration (`PreRegistrationValidationResults`: Approved/Rejected/Suspended). Every transition is journalled in `PreRegistrationLog`
3. **Transport proposals** — The plant publishes transports needing a carrier (`TransportProposalStatus`: Active/Ended); suppliers answer with a `TransportProposalResult`
4. **Shipments** — Consignment lines attached to a pre-registration, kept in sync with LN through Internal
5. **Master data** — `Company`, `Supplier`, `SubTransporter`, `Zone`, `Material`, `MaterialType`, `Macrogroup`, `Cert`

`ReservationType` (UNLOAD=0, LOAD=1, UNLOAD_AND_LOAD=2, SPECIAL=3) discrimina il flusso: Internal ha due comandi di sync distinti, uno per LOAD e uno per UNLOAD.

### Key Directories

- `app/Models/` — 15 Eloquent models. Core: `PreRegistration`, `PreRegistrationLog`, `TransportProposal`, `TransportProposalResult`, `Shipment`, `Company`, `Supplier`, `SubTransporter`, `Zone`, `Material`, `MaterialType`, `Macrogroup`, `Cert`, `Upload`, `User`
- `app/Livewire/` — 30 componenti Livewire 4, organizzati per entità con il pattern `Form.php` + `Table.php` (`PreRegistration/`, `TransportProposal/`, `Company/`, `Supplier/`, `Zone/`, `Material/`, …). `PreRegistration/Form.php` e `PreRegistration/ValidationForm.php` sono i più complessi
- `app/Http/Controllers/Api/` — 8 controller, la superficie API consumata da Internal
- `app/Http/Controllers/Admin/` — CRUD delle anagrafiche di sistema
- `app/Enums/` — 9 enum con `bensampo/laravel-enum`
- `app/Policies/`, `app/Notifications/User/` — autorizzazioni e notifiche utente

### API (server OAuth2)

Tutte le rotte di `routes/api.php` stanno in un unico gruppo con middleware `client` (alias di `App\Http\Middleware\CheckToken`, registrato in `app/Http/Kernel.php`), autenticazione **OAuth2 Client Credentials Grant** via Laravel Passport. Non c'è più Password Grant.

| Rotta | Metodi |
|---|---|
| `pre-registration` | `index`, `store`, `show` |
| `pre-registration/updateLnId`, `updateStatus`, `{ln_id}/updateShipments` | `POST` |
| `transport-proposal` | `store`, `show` |
| `company`, `supplier`, `zone` | `store`, `show` |
| `macrogroup`, `material`, `material-type` | `show` |

`show('*')` su `pre-registration` restituisce l'elenco completo con una trasformazione `_id` → `code`/`name` che non è presente in `show($id)`: divergenza nota, documentata come debito tecnico.

**Il contratto API è condiviso con VarQi-Internal.** Modificarlo significa aggiornare anche `VarQi-Internal/app/Classes/VarqiApiClient.php` e i comandi `sync:*` — modifica cross-repo, mai unilaterale.

### Authentication & Authorization

- `laravel/passport` — server OAuth2 per le API M2M
- `silber/bouncer` — ruoli e abilities per il portale
- `lab404/laravel-impersonate` — impersonificazione utenti per il supporto
- `infinitypaul/laravel-password-history-validation` — divieto di riuso password

### Frontend Stack

Blade + Livewire 4 + Alpine.js 2.7 + AdminLTE 3 su Bootstrap 4 + jQuery + Select2 + SweetAlert2 + TinyMCE. Build con **Vite** (`vite.config.js`, alias `$` → jQuery); `laravel-mix` è ancora in `devDependencies` ma non è più il build attivo.

### Database

**MySQL** (`DB_CONNECTION=mysql`). Nessuna connessione secondaria: i dati LN arrivano solo indirettamente, scritti da Internal via API (`updateLnId`, `updateShipments`).

### Documenti e allegati

`barryvdh/laravel-dompdf` e `setasign/fpdf` + `fpdi` per la generazione e il merge dei PDF, `simplesoftwareio/simple-qrcode` per i QR dei trasporti, `shuchkin/simplexlsxgen` per gli export.

### Scheduling

Nessun comando schedulato: l'app è passiva rispetto al sync: è Internal a chiamarla ogni 10 minuti.
