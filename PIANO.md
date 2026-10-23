# Piano – Mappa Cronaca Italia

## 1. Raccolta

**Fonti iniziali** (10–20 feed per l'MVP, elenco in `scripts/feeds.json`):
- Feed ANSA regionali e sezione cronaca nazionale
- Feed RSS di quotidiani locali con buona copertura territoriale
- Google News RSS con query per regione/parola chiave (opzionale)

Ogni voce di `feeds.json`: `{ "url", "testata", "ambito": "nazionale" | "regionale" | "locale", "regione" }`. L'ambito serve al punteggio di rilevanza.

**Pre-filtro** a parole chiave per scartare sport, meteo, politica, spettacolo prima di passare all'LLM (riduce costi e rumore).

## 2. Elaborazione (`scripts/collect.mjs`)

1. Scarica tutti i feed e scarta gli URL già presenti in `articles`.
2. Pre-filtro parole chiave.
3. **Classificazione LLM** su titolo + descrizione del feed. Output JSON:
   ```json
   { "cronaca": true, "categoria": "omicidio", "comune": "Pesaro", "provincia": "PU", "gravita": 4, "titolo_breve": "..." }
   ```
   Categorie iniziali: `omicidio`, `furto_rapina`, `protesta_manifestazione`, `incidente_grave`, `criminalita_organizzata`, `altro`.
4. **Geocodifica**: match comune + provincia su `data/comuni.json` → codice ISTAT, lat, lon. Se non trovato: scartato e loggato.
5. **Deduplica**: stesso comune + stessa categoria entro 48 ore = stesso evento. L'articolo nuovo si collega all'evento esistente e incrementa `n_fonti`.
6. **Punteggio di rilevanza**, ricalcolato a ogni nuova fonte:
   - numero di testate distinte (peso principale)
   - bonus se almeno una fonte è `nazionale`
   - gravità
   Soglia da tarare dopo 2 settimane di dati reali.
7. Scrittura su Supabase.

## 3. Schema dati (bozza, `supabase/schema.sql`)

**events**
| campo | tipo |
|---|---|
| id | uuid |
| data_evento | timestamptz |
| categoria | text |
| istat | text |
| lat, lon | double precision |
| gravita | smallint (1–5) |
| titolo_breve | text, opzionale |
| n_fonti | int |
| rilevanza | real |
| creato_il | timestamptz |

**articles**
| campo | tipo |
|---|---|
| url | text, PK |
| testata | text |
| ambito | text |
| pubblicato_il | timestamptz |
| event_id | uuid → events.id |

RLS: lettura pubblica su entrambe, scrittura solo con service key.

## 4. Mappa (frontend)

- Leaflet + markercluster: punti raggruppati da lontano, separati con lo zoom.
- Icona/colore per categoria, layer attivabili.
- Filtri: periodo (24h / 7gg / 30gg), categorie, rilevanza minima (default: solo "nazionale").
- Popup: categoria, comune, data, titolo breve, link alle notizie.
- Pannello laterale con lista cronologica sincronizzata.

## 5. Scheduler

GitHub Action con `schedule: cron: '0 * * * *'` + `workflow_dispatch` per lanci manuali. Le chiamate esterne tengono attivo il progetto Supabase free.

Secrets: `SUPABASE_URL`, `SUPABASE_SERVICE_KEY`, `LLM_API_KEY`.

## Fasi

1. **MVP**: schema, `comuni.json`, 10–20 feed, raccolta con classificazione e geocodifica, mappa con cluster e filtri base. Deduplica semplice.
2. **Rilevanza**: punteggio, soglia, deduplica raffinata. Taratura su dati reali.
3. **Extra**: heatmap/coropletica per provincia, trend, eventuale digest giornaliero.

## Decisioni aperte

- Quale LLM e quale modello usare per la classificazione (costo per ~qualche centinaio di titoli al giorno).
- Progetto Supabase nuovo o schema dedicato in un progetto esistente (limite free: 2 progetti).
- Elenco definitivo dei feed iniziali.
