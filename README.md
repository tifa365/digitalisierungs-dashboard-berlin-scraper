# Berlin Digitalization Dashboard API

Unofficial API documentation for the [Berlin Digitalization Dashboard](https://digitalisierungs-dashboard.berlin.de/) (Digitalisierungs-Dashboard Berlin).

The dashboard tracks the digitalization status of public services in Berlin under Germany's OZG (Onlinezugangsgesetz - Online Access Act).

## API Overview

The dashboard uses Next.js Server Actions. The API returns data about 6,800+ public services (Leistungen) and their digitalization status.

### Endpoint

```
POST https://digitalisierungs-dashboard.berlin.de/
```

### Required Headers

| Header | Description |
|--------|-------------|
| `Content-Type` | `application/json` |
| `next-action` | Server Action hash (see below) |
| `next-router-state-tree` | URL-encoded router state |

Current values (may change between deployments):
```
next-action: 7f9824d5ef028cde6ac686e0a4854e6f1cd8da35cf
next-router-state-tree: %5B%22%22%2C%7B%22children%22%3A%5B%22__PAGE__%22%2C%7B%7D%2Cnull%2Cnull%5D%7D%2Cnull%2Cnull%2Ctrue%5D
```

### Request Body

JSON array with a single filter object:

```json
[{
  "page": 1,
  "search": "",
  "ozg_reifegrad": [],
  "ressort": [],
  "politikfeld": [],
  "efa_umsetzung": [],
  "ozg_prioritaet": [],
  "digitalisierungs_potenzial": [],
  "sdg_relevant": []
}]
```

All fields are required. Use empty arrays `[]` or empty string `""` for no filter.

### Response Format

RSC (React Server Components) streaming format with newline-separated chunks:

```
0:{"a":"$@1","f":"","b":"pB8gw_6GyPTngUxELaipA"}
1:{"page":1,"data":[...],"count":6842,"hasMore":true}
```

Parse by extracting the JSON after `1:`.

## Filter Options

### Ressort (Department)

| Code | Department |
|------|------------|
| `SenASGIVA` | Arbeit, Soziales, Gleichstellung, Integration, Vielfalt und Antidiskriminierung |
| `SenBJF` | Bildung, Jugend und Familie |
| `SenFin` | Finanzen |
| `SenInnSport` | Inneres und Sport |
| `SenJustV` | Justiz und Verbraucherschutz |
| `SenMVKU` | Mobilität, Verkehr, Klimaschutz und Umwelt |
| `SenStadt` | Stadtentwicklung, Bauen und Wohnen |
| `SenWGP` | Wissenschaft, Gesundheit und Pflege |
| `SenWiEnBe` | Wirtschaft, Energie und Betriebe |

### Politikfeld (Policy Area)

| Code | Area |
|------|------|
| `ENG` | Engagement & Hobby |
| `FAM` | Familie & Jugend |
| `GES` | Gesundheit |
| `INN` | Inneres |
| `JSZ` | Justiz |
| `MOB` | Mobilität |
| `SOZ` | Soziales |
| `STE` | Steuern |
| `STW` | Stadtentwicklung & Wohnen |
| `UWN` | Umwelt & Natur |
| `VBS` | Verbraucherschutz |
| `WIR` | Wirtschaft |
| `WIS` | Wissenschaft & Forschung |

### OZG Reifegrad (Maturity Level)

| Level | Description |
|-------|-------------|
| 0 | Not digitalized |
| 1 | Information available online |
| 2 | Forms downloadable |
| 3 | Forms submittable online |
| 4 | Fully digital with authentication |
| 5 | Fully automated |

### Other Filters

| Filter | Values |
|--------|--------|
| `efa_umsetzung` | `["Ja"]`, `["Nein"]` |
| `ozg_prioritaet` | `["1"]`, `["2"]`, `["3"]`, `["4"]` |
| `sdg_relevant` | `["ja"]`, `["nein"]` |

## Response Schema

Each service (Leistung) in the response contains:

| Field | Description |
|-------|-------------|
| `id` | Unique identifier |
| `leistungsbezeichnung` | Service name |
| `leistungsschluessel` | LeiKa service key |
| `ozg_reifegrad` | Maturity level (0-5) |
| `ressort` / `ressort_kurz` | Responsible department |
| `politikfeld` / `politikfeld_kurz` | Policy area |
| `link` | Link to service.berlin.de |
| `ozg_umsetzung` | Implementation approach |
| `ozg_themenfeld` | OZG topic area |
| `ozg_leistung` | OZG service category |
| `federfuehrendes_bundesland` | Lead federal state (EfA) |
| `efa_umsetzung` / `efa_faehig` | EfA implementation status |
| `sdg_relevant` | SDG relevance |
| `priorisierung` | Priority assessment |
| `technische_komplexitaet` | Technical complexity |
| `digitalisierungs_potential` | Digitalization potential |

## Testing

### Schemathesis (Fuzzing)

```bash
uvx schemathesis run openapi.yaml \
  --url https://digitalisierungs-dashboard.berlin.de \
  --mode positive \
  --exclude-checks positive_data_acceptance
```

### Dredd (Contract Testing)

```bash
npx dredd openapi.yaml https://digitalisierungs-dashboard.berlin.de \
  --hookfiles=dredd-hooks.cjs
```

<details>
<summary>dredd-hooks.cjs</summary>

```javascript
const hooks = require('hooks');

hooks.beforeEach((transaction, done) => {
  transaction.request.headers['next-action'] = '7f9824d5ef028cde6ac686e0a4854e6f1cd8da35cf';
  transaction.request.headers['next-router-state-tree'] = '%5B%22%22%2C%7B%22children%22%3A%5B%22__PAGE__%22%2C%7B%7D%2Cnull%2Cnull%5D%7D%2Cnull%2Cnull%2Ctrue%5D';
  done();
});

['400 > text/plain', '405 > text/plain', '500 > text/x-component', '200 > text/html'].forEach(status => {
  hooks.before(`/ > Search and filter public services > ${status}`, (t, done) => { t.skip = true; done(); });
});

hooks.beforeValidation('/ > Search and filter public services > 200 > text/x-component', (transaction, done) => {
  const body = transaction.real.body;
  if (body && body.includes('0:{') && body.includes('1:{')) {
    transaction.expected.body = body;
  }
  done();
});
```
</details>

## Files

| File | Description |
|------|-------------|
| `openapi.yaml` | OpenAPI 3.0 specification |
| `CLAUDE.md` | AI assistant guidance |

## Glossary

| Term | Description |
|------|-------------|
| **OZG** | Onlinezugangsgesetz - German law requiring public services to be available online |
| **EfA** | "Einer für Alle" - Federal cooperation model where one state develops a service for all |
| **SDG** | Single Digital Gateway - EU regulation for cross-border access to public services |
| **LeiKa** | Leistungskatalog - German public service catalog with unique identifiers |
| **RSC** | React Server Components - Next.js streaming response format |

## License

This is an unofficial documentation project. The API and data belong to the State of Berlin.
