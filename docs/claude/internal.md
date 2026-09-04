# VarQi-Internal

Istruzioni Claude Code per il sotto-progetto `VarQi-Internal/` — submodule di `https://github.com/BAX-IoT/VarQI.git`.

> Questo file vive nella root del monorepo. Non creare `CLAUDE.md` né `docs/` dentro il submodule.

## Project Overview

VarQi Internal è un'applicazione Laravel 10 per la gestione dei varchi in impianti di trattamento rifiuti. Gestisce prenotazioni veicoli, movimenti attraverso i gate (ingresso/uscita/intermedi), letture della pesa, integrazione telecamere e validazione dei codici CER. Locale primario italiano (timezone Europe/Rome).

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
./vendor/bin/phpunit tests/Feature/ExampleTest.php   # Single file

# Scheduled commands
php artisan sync:varqi                     # Sync entities between VarQI instances (every 2h)
php artisan sync:pre-registration-load      # Pull LOAD pre-registrations (every 10m)
php artisan sync:pre-registration-unload    # Pull UNLOAD pre-registrations (every 10m)

# Code style
# StyleCI (Laravel preset, PHP 8) runs on push — see .styleci.yml
```

## Architecture

### Core Domain Flow

1. **Reservations** — Vehicles book entry at a gate (truck/trailer plates, company, CER waste code)
2. **Movements** — Track ENTRANCE, EXIT, INTERMEDIATE events with weight readings per reservation
3. **Movement Reports** — Post-movement inspection data with camera image capture
4. **CER Validation** — Waste code verification via external API
5. **Totem (Kiosk)** — Public-facing Livewire interface for vehicle operators at gates

### Key Directories

- `app/Models/` — 28 Eloquent models. Core: `Reservation`, `Movement`, `Gate`, `Lane`, `Camera`, `Device`, `MovementReport`, `Company`, `Cer`
- `app/Http/Livewire/` — 37 reactive components (Livewire 2.x). The `Totem.php` component is the largest and most complex
- `app/Http/Controllers/Admin/` — CRUD for system entities (gates, lanes, cameras, devices, materials, companies, CER codes)
- `app/Enums/` — 22 domain enums using `bensampo/laravel-enum`. Key: `MovementType` (ENTRANCE/EXIT/INTERMEDIATE), `ReservationStatus` (Waiting/Entered/EnteredManual/Exited)
- `app/Classes/` — Business logic: `Ecocerved.php` (CER external service), `WeightScale.php`, `LNUtility.php` (Infor LN helpers), `VarqiApiClient.php` (OAuth2 client for the PreRegistration API)
- `app/Console/Commands/` — `VarqiSync.php`, `PreRegistrationLoadSync.php`, `PreRegistrationUnloadSync.php`
- `app/Http/Helpers.php` — Global helpers including `sendVehicleStatus()` (SOAP XML to ERP), `determinateReservationNumer()`
- `simulator/` — Gate simulation tooling (stoplight and barrier), plus the LN simulator (`simulator/ln/`)

### External Integrations

- **SOAP Webservice** — ERP vehicle status updates via `sendVehicleStatus()` in Helpers.php
- **CER API** — Waste code validation (toggle via `CER_API_ENABLE` env var)
- **VarQi-PreRegistration API** — OAuth2 **Client Credentials Grant** via `app/Classes/VarqiApiClient.php`; consumes zones, companies, suppliers, materials, pre-registrations and transport proposals. Credentials in `VARQI_API_CLIENT_ID` / `VARQI_API_CLIENT_SECRET`
- **LN (Infor LN)** — Legacy SQL Server database for zones/companies/transport data
- **IP Cameras** — Merit Lilin OnBoard LPR cameras via RTSP/HTTP digest auth with PTZ control

### Authorization

- `silber/bouncer` for role/ability-based authorization
- `laravel/sanctum` for API token authentication
- Policies: `MovementPolicy`, `MovementReportPolicy`

### Frontend Stack

Blade templates + Livewire 2.x + Alpine.js 2.7 + Bootstrap 5 + jQuery + Select2. Assets compiled with Vite.

### Database

Both connections are **SQL Server** (`sqlsrv` driver), and `sqlsrv` is the app default:
- **Primary** (`sqlsrv`): VarQi application database
- **Secondary** (`sqlsrv-stpeter`): legacy LN/BAX database — exposes data via SQL views (`VW_BAX_TRASPORTI_LN`, `VW_BAX_ZONE_CARICO_SCARICO`, `VW_BAX_ANAGRAFICA_AZIENDE`). Table names configured via env vars (`DB_LN_TRANSPORTS_TABLE`, `DB_LN_SHIPMENTS_TABLE`, `DB_LN_ZONES_TABLE`, `DB_LN_COMPNIES_TABLE` — note the typo in COMPNIES is intentional, matches the codebase).

### Scheduling

Three scheduled commands in `app/Console/Kernel.php`:
- `sync:varqi` — every 2 hours
- `sync:pre-registration-load` — every 10 minutes
- `sync:pre-registration-unload` — every 10 minutes
