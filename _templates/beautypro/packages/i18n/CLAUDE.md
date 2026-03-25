# CLAUDE.md — packages/i18n

i18next setup and all locale files. Serbian (`sr`) is the default and primary locale.

## Structure

```
src/
  index.ts          → i18next init, exports t(), useTranslation
  locales/
    sr.json         → srpski (DEFAULT — always up to date)
    en.json         → engleski
    hr.json         → hrvatski
    bs.json         → bosanski
```

## Adding New Strings

1. Add to `sr.json` first (primary locale)
2. Add placeholder in `en.json`, `hr.json`, `bs.json` (can use Serbian as fallback during dev)
3. Never hardcode strings directly in components — always `t('namespace.key')`

## Namespace Convention

```json
// sr.json
{
  "common": { "save": "Sačuvaj", "cancel": "Otkaži", "delete": "Obriši" },
  "auth": { "login": "Prijava", "signup": "Registracija" },
  "clients": { "title": "Klijenti", "add": "Dodaj klijenta" },
  "appointments": { "book": "Zakaži termin", "cancel": "Otkaži termin" },
  "finance": { "income": "Prihodi", "expenses": "Troškovi" },
  "settings": { "theme": "Tema", "language": "Jezik" },
  "professions": { "lash": "Lash Artist", "nails": "Nail Artist" }
}
```

## Usage in Components

```typescript
import { useTranslation } from '@beautypro/i18n'

const { t } = useTranslation()
// t('clients.title') → "Klijenti"
```

## Currency & Date Formatting

Do NOT use i18n for currency/dates — use `Intl` directly:
```typescript
// From packages/shared/utils/format.ts
formatPrice(amount, currency, locale)  // Intl.NumberFormat
formatDate(date, locale)               // Intl.DateTimeFormat
```
