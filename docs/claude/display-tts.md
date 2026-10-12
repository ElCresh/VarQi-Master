# VarQi-Display-TTS

Istruzioni Claude Code per il sotto-progetto `VarQi-Display-TTS/` — submodule di `git@github.com:BAX-IoT/VarQi-Display-TTS.git`.

> Questo file vive nella root del monorepo. Non creare `CLAUDE.md` né `docs/` dentro il submodule.

## Project Overview

Servizio Python che gira su un Raspberry Pi al varco: riceve comandi HTTP da VarQi-Internal, li accoda e li esegue in sequenza, pronunciando annunci vocali (TTS) e scrivendo messaggi sul display a pannello del piazzale. Non ha database né autenticazione: è un device di rete passivo.

## Stack

- **Python 3.9**, versione obbligatoria per compatibilità (venv locale in `.venv`, git-ignored): niente sintassi successiva alla 3.9 (`match`, tipi `X | Y`, ecc.). Falcon 3.1.3 non si installa sulle versioni recenti di Python
- **Falcon 3.1** servito con `wsgiref.simple_server` (single process)
- TTS via **Vocalware** (API HTTP, migrato da gTTS), riproduzione audio con `pygame` (`DoorBell.mp3` come jingle)
- Display pilotato via socket TCP (`DISPLAY_IP` / `DISPLAY_PORT`)
- In produzione gira come processo Python diretto in una sessione `screen` (vedi "Deploy in produzione")

## File

- `main.py` — rotte Falcon, server WSGI multi-thread, logging e task processor (un unico thread di lavoro che svuota la coda `tasks`, una voce alla volta; dopo 60 s senza messaggi il display torna al nome del varco e da lì viene rinfrescato ogni 60 s)
- `utils.py` — `speak()` (Vocalware + audio) e `display()` (socket verso il pannello; restituisce `None` se va a buon fine, altrimenti il testo dell'errore)
- `deploy/varqi-tts.service.example` — unità systemd di esempio (non ancora in uso in produzione)
- `tools/fake_display.py` — finto pannello per i test locali: server TCP (default `127.0.0.1:1470`) che decodifica i pacchetti di `display()` e stampa `[clear]`, `[regular]`, `[flash]`, `[bold]` o `[scroll]` con il testo (`--raw` stampa anche i byte)
- `settings.py` — configurazione locale, git-ignored; template in `settings.py.example` (`PORT`, `DEP_NAME` (nome del varco, default `"DEP1"`), `LOG_FILE`/`LOG_MAX_BYTES`/`LOG_BACKUP_COUNT` (default `varqi-tts.log`, 1 MB, 5 backup), credenziali Vocalware, `DISPLAY_IP`, `DISPLAY_PORT`)

## Test in locale

1. `.venv\Scripts\python tools/fake_display.py` in un terminale
2. in `settings.py` locale `DISPLAY_IP = "127.0.0.1"`, `DISPLAY_PORT = 1470`
3. `.venv\Scripts\python main.py` e chiamare le rotte da browser (`http://localhost:8080/display/TEST`, `/dep1/7`, `/status/display`)

`/speak/...` chiama la vera Vocalware con le credenziali di `settings.py` e suona sulle casse del PC.

## API HTTP (tutte `GET`, risposta testuale `true`)

| Rotta | Effetto |
|---|---|
| `/speak/{lang}/{message}` | Accoda un annuncio vocale |
| `/display/{message}` | Mostra un messaggio sul display |
| `/dep1/{reservation}` … `/dep8/{reservation}` | Mostra il numero di prenotazione (flash → bold → regular) |
| `/status/display` (anche con `/` finale) | Restituisce l'ultimo messaggio inviato con successo, oppure `[DISP ERR]` se il pannello non risponde |

## Integrazione con VarQi-Internal

Internal registra il Raspberry come `Device` di tipo `DeviceType::RASPBERRY_TTS` associato al gate e chiama `{device->url}speak/…`, `display/…` e `status/display/` (vedi `Reservation/CurrentList.php`, `Reservation/Row.php`, `DisplayStatus.php`). Cambiare rotte o formato delle risposte qui significa aggiornare anche quei componenti Livewire: modifica cross-repo.

## Deploy in produzione (verificato su `varqi-rpi-dep1`, 2026-10-12)

**Docker non è usato**: è installato sul Raspberry ma senza container. Il servizio gira come processo Python diretto:

- codice in `~/varqi-rpi` (utente `pi`), configurazione in `settings.py` locale, in ascolto su `0.0.0.0:8080`
- lanciato in una sessione GNU screen staccata chiamata `tts` da uno snippet in fondo a `~/.bashrc` di `pi`:
  ```sh
  if ! screen -list | grep -q "tts"; then
          cd varqi-rpi
          screen -S tts -dm bash -c "python3 main.py"
  fi
  ```
- parte al boot solo se c'è l'autologin di `pi` sulla console (dedotto dal PID basso e dall'avvio il giorno del boot, non verificato), che apre una shell interattiva e quindi esegue `.bashrc`. Senza autologin non parte finché qualcuno non entra come `pi`
- ogni login interattivo di `pi` (anche via SSH) riesegue lo snippet: se la sessione `tts` è giù il servizio riparte, e la shell di quel login si ritrova in `~/varqi-rpi` per via del `cd`
- `grep -q "tts"` intercetta qualunque sessione screen con `tts` nel nome
- nessun supervisore: se `main.py` va in crash la sessione screen si chiude e il servizio resta giù finché lo snippet non viene rieseguito
- log in `~/varqi-rpi/varqi-tts.log` con rotazione per dimensione (default 1 MB × 6 file, circa 6 MB al massimo) e anche sulla console di `screen -r tts` (staccarsi con `Ctrl+A D`). Le richieste HTTP riuscite non vengono loggate
- in Internal il `Device` corrispondente ha `url` = `http://<ip-raspberry>:8080/`

`Dockerfile` e `docker-compose.yaml` (versione del 2023, mai allineati al passaggio a `settings.py`) sono stati rimossi dal repo il 2026-10-12. L'immagine `elcresh/varqi-rpi:1.0.0` resta pubblicata su Docker Hub ma non è usata.

## Criticità note

Corrette nella working copy del submodule (2026-10-12), **da committare e da portare sul Raspberry**:

- **memory leak**: il task processor creava un `threading.Timer` (un thread nuovo) a ogni tick. Sulle 3.9 precedenti alla correzione di bpo-37788 ogni thread non joinato lascia un lock in `threading._shutdown_locks`: circa 5,7 milioni di thread in 4 mesi, compatibile con i circa 627 MB di RSS osservati. Ora c'è un unico thread di lavoro con `Event.wait()`
- il task processor non muore più: ogni passo è in `try/except` (`[TASK_ERR]` + traceback). Prima un'eccezione, per esempio da Vocalware, fermava la coda per sempre, mentre le rotte HTTP continuavano a rispondere `true`
- timeout su Vocalware (5 s di connessione, 20 s di lettura) e sui socket del display (1 s). Rimosso `check_internet()` (ping a google.com prima di ogni annuncio)
- risposta di Vocalware controllata (status 200 + contenuto MP3), altrimenti `[TTS_ERR]` e nessuna riproduzione. In caso di errore di rete si logga solo il tipo di eccezione, non l'URL con gli ID dell'account
- parametri di Vocalware passati con `params=` (URL encoding corretto, checksum sul testo grezzo)
- audio: `mixer.init()` una volta sola, niente busy-wait, limite di 60 s per clip
- `/status/display` riflette lo stato reale: `[DISP ERR]` se l'ultimo invio è fallito. La caduta e il ritorno del pannello vengono loggati una volta sola (`[DISP_ERR]` / `[DISP_OK]`), non a ogni tentativo
- `/display/{testo}` invia il messaggio una volta sola e lo tiene 4 s (prima: due invii consecutivi con pulizia in mezzo, cioè sfarfallio)
- server WSGI multi-thread: un client bloccato non ferma più le altre richieste
- `sendall()` al posto di `send()` verso il pannello
- nome del varco configurabile con `DEP_NAME` (default `"DEP1"`)
- a riposo il nome del varco viene rimandato ogni 60 s, non a ogni tick

Ancora aperte:

- testo non codificato nell'URL lato Internal (`/`, `?`, `#` nel messaggio rompono la rotta)
- nessuna autenticazione sulle rotte HTTP
- testo inviato al pannello in UTF-8: le lettere accentate probabilmente escono sbagliate (da provare sul pannello vero) e il limite dei 16 caratteri è contato in lettere, non in byte
- `/display/` usato prima di qualunque prenotazione chiude con `DEP1_NA`
- `.idea/` versionata (compreso `workspace.xml`); il `.gitignore` non blocca i path Claude
- da verificare dopo il rilascio che l'RSS resti stabile (`ps -o rss,etime -C python3` subito e dopo qualche giorno)
- deploy con `screen` + `.bashrc`, senza supervisore. Unità systemd pronta in `deploy/varqi-tts.service.example` (riavvio automatico, avvio al boot senza autologin, log in `journalctl -u varqi-tts`), da provare sul Raspberry: il punto critico è l'audio fuori dalla sessione utente (`SDL_AUDIODRIVER=alsa` + gruppo `audio`). Con systemd si può mettere `LOG_FILE = ""` e lasciare la rotazione a journald
