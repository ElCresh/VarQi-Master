# VarQi

Monorepo di VarQi, la piattaforma di gestione varchi e logistica dei rifiuti: prenotazioni, movimenti ai gate, pesate, telecamere e pre-registrazione dei trasporti da parte dei fornitori.

| Sotto-progetto | Path | Repo | Cos'è |
|---|---|---|---|
| Internal | `VarQi-Internal/` | `BAX-IoT/VarQI` | App Laravel 10 + Livewire 2 + SQL Server — gestione varchi, movimenti, pesate, totem di piazzale |
| PreRegistration | `VarQi-PreRegistration/` | `BAX-IoT/VarQI-PreRegistration` | App Laravel 13 + Livewire 4 + MySQL — portale fornitori e server OAuth2 (Passport) delle API di sync |

Entrambi sono git submodule. Clone completo:

```bash
git clone --recurse-submodules git@github.com:ElCresh/VarQi-Master.git
# oppure, su un clone esistente:
git submodule update --init --recursive
```

I due sotto-progetti dialogano via API: Internal è il **client** OAuth2 (Client Credentials Grant) che sincronizza pre-registrazioni, proposte di trasporto e anagrafiche dal PreRegistration.

Documentazione e istruzioni per Claude Code: `CLAUDE.md` e `docs/claude/`.
