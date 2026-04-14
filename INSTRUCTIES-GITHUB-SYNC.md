# GitHub sync instellen in Make.com

Het dashboard leest `data.json` van `https://bramovergaag.github.io/bg800-dashboard/data.json`. Om wijzigingen uit WhatsApp naar iedereen te delen moet Make.com dit bestand updaten na elke nieuwe Data Store record.

## Stap 1 — GitHub token aanmaken

1. Ga naar https://github.com/settings/tokens/new
2. Note: `Make.com BG800 sync`
3. Expiration: `No expiration` (of 1 jaar)
4. Scopes: alleen `repo` aanvinken
5. Klik **Generate token** — kopieer het token (`ghp_xxxxxx`), je ziet het maar 1x

## Stap 2 — Data Store GET scenario (handmatig testen)

Voeg aan het bestaande "BG800 WhatsApp Barrierplanning" scenario TWEE modules toe, ná module 3 (Data Store: Add Record):

### Module 5: Data Store — List records
- Datastore: `113253` (BG800 planning)
- Limit: `1000`

### Module 6: Array aggregator
- Source module: `5` (List records)
- Target structure type: `Custom`
- Fields:
  - `project` — text — `{{5.project}}`
  - `barriers` — number — `{{5.barriers}}`
  - `startWeek` — number — `{{5.startWeek}}`
  - `startYear` — number — `{{5.startYear}}`
  - `endWeek` — number — `{{5.endWeek}}`
  - `endYear` — number — `{{5.endYear}}`

### Module 7: HTTP — Make a request
- URL: `https://api.github.com/repos/bramovergaag/bg800-dashboard/contents/data.json`
- Method: `PUT`
- Headers:
  - `Authorization`: `Bearer GHP_JOUW_TOKEN_HIER`
  - `Accept`: `application/vnd.github+json`
  - `User-Agent`: `make-bg800-sync`
- Body type: `Raw`
- Content type: `JSON (application/json)`
- Request content:
```json
{
  "message": "Update barrierplanning via WhatsApp",
  "content": "{{base64(toString(6.array))}}",
  "sha": "{{7.sha}}"
}
```

> **Let op:** de `sha` is de hash van het bestaande bestand. Voeg hiervoor een extra **Module 7a — HTTP GET** toe vóór de PUT:
> - URL: `https://api.github.com/repos/bramovergaag/bg800-dashboard/contents/data.json`
> - Method: `GET`
> - Headers: `Authorization: Bearer GHP_…`
>
> Gebruik dan `{{7a.sha}}` in de PUT body.

## Stap 3 — Test

1. Stuur een WhatsApp bericht in de BG800 groep: `Grebbedijk 60 barriers week 10 tot 20 2025`
2. Wacht tot Make.com het scenario draait
3. Check https://github.com/bramovergaag/bg800-dashboard/commits/main — er moet een nieuwe commit zijn
4. Open https://bramovergaag.github.io/bg800-dashboard/ — binnen 1 min zie je de wijziging voor iedereen

## Flow overzicht

```
WhatsApp bericht
  └─→ Make.com regex
       └─→ Data Store (add record)
            └─→ Data Store (list all)
                 └─→ Array aggregator
                      └─→ GitHub API PUT data.json
                           └─→ GitHub Pages serveert nieuwe data.json
                                └─→ Dashboard refresh (elke 2 min) toont update
```

## Beveiliging

- Het GitHub token heeft alleen `repo` scope (geen admin)
- Het token staat alleen in Make.com, niet in de publieke code
- Bij lekken: regenereer via https://github.com/settings/tokens
