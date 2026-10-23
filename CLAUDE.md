# Mappa Cronaca Italia

Pagina web con una mappa dell'Italia che raggruppa e mostra eventi di cronaca rilevanti (omicidi, grandi furti/rapine, proteste e manifestazioni, incidenti gravi, operazioni antimafia…), raccolti automaticamente ogni ora da feed RSS di testate nazionali e locali.

Obiettivo: tenere d'occhio la cronaca di interesse nazionale e quella locale significativa, escludendo il micro-locale senza rilevanza.

Il piano completo è in `PIANO.md`.

## Stack

- **Frontend**: HTML + CSS + JavaScript vanilla, senza framework né build step. Leaflet + Leaflet.markercluster da CDN. Ospitato su GitHub Pages.
- **Database**: Supabase (piano **free**). Il frontend legge con la chiave pubblica; le policy RLS permettono solo lettura agli anonimi.
- **Raccolta**: script Node (`scripts/collect.mjs`) eseguito ogni ora da una **GitHub Action schedulata**. Scrive su Supabase con la service key salvata nei secrets del repo.
- **Niente n8n e niente servizi sul VPS.**

## Vincoli da rispettare sempre

- **Dati minimi**: per ogni evento si salvano solo data/ora, categoria, codice ISTAT del comune, coordinate, gravità, link alla notizia, testata, numero di fonti. Al massimo un titolo breve come etichetta.
- **Mai salvare il testo degli articoli** né riassunti lunghi: la descrizione è il link alla notizia originale.
- **Mai salvare nomi di persone** (vittime, indagati, testimoni).
- **Supabase free**: il progetto va in pausa dopo 7 giorni di inattività e pg_cron interno non lo tiene sveglio. Per questo lo scheduler è esterno (GitHub Actions). Non spostare la schedulazione su pg_cron.
- Geocodifica tramite tabella ISTAT dei comuni in locale (`data/comuni.json`), senza API esterne. Nominatim solo come fallback eventuale.
- Segreti (chiavi Supabase, chiave API dell'LLM) solo nei GitHub Secrets, mai nel codice.

## Modo di lavorare

- Prima di cambiamenti strutturali (schema del database, struttura cartelle, nuove dipendenze, cambio di approccio) descrivi cosa intendi fare e aspetta conferma.
- Procedi per fasi come indicato in `PIANO.md`; non anticipare le fasi successive.
- Comunicazione in italiano.

## Struttura prevista

```
index.html              pagina mappa
css/style.css
js/map.js               mappa, cluster, filtri, lettura da Supabase
js/config.js            URL e chiave pubblica Supabase
scripts/collect.mjs     raccolta feed → classificazione → geocodifica → dedup → scrittura
scripts/feeds.json      elenco feed RSS
data/comuni.json        comuni ISTAT con codice, nome, provincia, lat/lon
supabase/schema.sql     tabelle, indici, policy RLS
.github/workflows/collect.yml   esecuzione oraria
```
