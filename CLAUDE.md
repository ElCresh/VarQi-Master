# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this monorepo root.

`C:\Users\Andrea\Projects\VarQi\` è il **monorepo** VarQi, piattaforma di gestione varchi e logistica dei rifiuti per BAX-IoT: prenotazioni, movimenti ai gate, pesate, telecamere, e pre-registrazione dei trasporti da parte dei fornitori. I due sotto-progetti sono **git submodule** con repo proprie; le istruzioni Claude vivono **solo qui**, mai dentro i submodule.

## Sotto-progetti

### VarQi-Internal — Applicazione di piazzale
- Path: `./VarQi-Internal/` — submodule di `https://github.com/BAX-IoT/VarQI.git`
- Stack: Laravel 10 + Livewire 2.x + Alpine 2.7 + Bootstrap 5 + SQL Server (`sqlsrv`) + PHP 8.1+
- Gestisce prenotazioni, movimenti (ENTRANCE/EXIT/INTERMEDIATE), pesate, telecamere LPR, validazione CER e il totem operatore. È il **client** OAuth2 delle API di PreRegistration e l'unico a parlare con LN (Infor) e con il webservice SOAP dell'ERP.
- Istruzioni complete: @docs/claude/internal.md

### VarQi-PreRegistration — Portale fornitori
- Path: `./VarQi-PreRegistration/` — submodule di `https://github.com/BAX-IoT/VarQI-PreRegistration.git`
- Stack: Laravel 13 + Livewire 4 + AdminLTE 3 / Bootstrap 4 + MySQL + Passport + PHP 8.2+
- Portale dove i fornitori registrano pre-registrazioni, proposte di trasporto e certificati. È il **server** OAuth2 (Passport, Client Credentials Grant) che espone le API consumate da Internal.
- Istruzioni complete: @docs/claude/preregistration.md

### Integrazione fra i due
Internal chiama le API di PreRegistration sotto `routes/api.php` (middleware `client`) con Client Credentials Grant, tramite `app/Classes/VarqiApiClient.php`. I comandi schedulati `sync:pre-registration-load` e `sync:pre-registration-unload` (ogni 10 minuti) e `sync:varqi` (ogni 2 ore) sono l'unico canale di comunicazione. **Toccare il contratto API significa modificare entrambe le repo**: è una modifica cross-repo, mai unilaterale.

## Regola sui dati Claude

`CLAUDE.md`, `docs/`, `.claude/` e `.superpowers/` stanno **esclusivamente in questa root**. Nei submodule `VarQi-Internal/` e `VarQi-PreRegistration/` non deve mai finire nulla di tutto ciò: le loro repo sono codice e basta, e i rispettivi `.gitignore` bloccano questi path. Se un piano o uno spec riguarda solo uno dei due sotto-progetti, va comunque scritto sotto `docs/` di root.

| Cosa | Dove |
|---|---|
| Istruzioni per Claude, per sotto-progetto | `docs/claude/internal.md`, `docs/claude/preregistration.md` |
| Piani e spec (superpowers) | `docs/superpowers/plans/`, `docs/superpowers/specs/` |
| Brief e report di esecuzione (sdd) | `.superpowers/sdd/` |
| Permessi locali | `.claude/settings.local.json` |

Versionati nella repo monorepo: `CLAUDE.md`, `README.md` e `docs/claude/`. Il resto (`docs/superpowers/`, `.superpowers/`, `.claude/`) è git-ignored e resta locale.

## Quando si lavora da root vs da sotto-progetto

| Scenario | WD consigliato |
|---|---|
| Feature di piazzale (movimenti, pesate, telecamere, totem, CER, LN, SOAP) | `VarQi-Internal/` |
| Feature portale fornitori (pre-registrazioni, proposte di trasporto, certificati, anagrafiche) | `VarQi-PreRegistration/` |
| Contratto API fra i due, OAuth, comandi `sync:*` (tocca entrambi) | root |
| Pianificazione cross-cutting, deploy, documentazione | root |

**Memoria**: la memoria di progetto è quella della root (`C--Users-Andrea-Projects-VarQi`). Lavorando da un sotto-progetto la memoria caricata è un'altra: per feedback e decisioni che valgono per tutto VarQi, avvia la sessione da root.

## Convenzioni cross-cutting

- **Git lo gestisce l'utente.** Claude non esegue **mai** comandi git che cambiano stato (`add`, `commit`, `push`, `branch`, `checkout`, merge, reset), in nessuna delle tre repo, senza richiesta esplicita. I comandi git di sola lettura (`status`, `diff`, `log`) sono ammessi. Ai punti naturali di commit si segnala a testo un "suggested commit point", non lo si esegue.
- **Sui submodule non si committa mai**, nemmeno se autorizzati a committare in root: si lascia la working copy modificata e si riporta cosa è cambiato. Solo un'autorizzazione esplicita dell'utente, sotto-repo per sotto-repo e volta per volta, cambia questa regola.
- **Commenti nel codice sempre in inglese**, in entrambi i sotto-progetti, anche quando la conversazione è in italiano. La spiegazione discorsiva resta in italiano, il codice no.
- **Piani salvati**: prima di ripianificare da zero, consultare `docs/superpowers/plans/` e la memoria di progetto. Se esiste già un piano approvato, si riparte da quello.
- **Consultare prima di bloccare scelte di stack/architettura**: si presentano i trade-off e si chiede, non si danno per assodate scelte significative (auth, driver, librerie).
- **Lettere accentate italiane sempre** (`può`, `è`, `perché`): mai apostrofo-sostituto (`puo'`, `e'`). UTF-8 funziona ovunque.
