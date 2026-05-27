# Pip-Log Exchange JSON Examples

Pip-Log exports JSON files to the game directory under `Export/<Player Name>-<Export Type>.json`.
Imports read from the same path for the current player name.

## Journal Entries

Path example: `Export/Courier-JournalEntries.json`

```json
{
  "version": 1,
  "kind": "JournalEntries",
  "playerName": "Courier",
  "dateTimeGame": "2281-01-21 14:05:00",
  "dateTimeReal": "2026-05-25 18:42:10",
  "data": [
    {
      "id": "1392-24-8801-318",
      "flags": 0,
      "datetime": "2281-01-21 13:42:00",
      "location": "Goodsprings",
      "text": "Met Sunny Smiles and learned the basics."
    },
    {
      "id": "7120-40-1933-55",
      "flags": 0,
      "datetime": "2281-01-22 09:10:00",
      "location": "Primm",
      "text": "Primm needs a new sheriff."
    }
  ]
}
```

## Player Events

Path example: `Export/Courier-PlayerEvents.json`

```json
{
  "version": 1,
  "kind": "PlayerEvents",
  "playerName": "Courier",
  "dateTimeGame": "2281-01-21 14:05:00",
  "dateTimeReal": "2026-05-25 18:42:10",
  "data": [
    {
      "id": "8804-63-9012-441",
      "date": "2281-01-25",
      "name": "Meet with Johnson Nash",
      "flags": 2
    },
    {
      "id": "2362-99-1200-706",
      "date": "*-10-23",
      "name": "Great War memorial",
      "flags": 2
    },
    {
      "id": "4601-32-7704-902",
      "date": "*-*-01",
      "name": "Monthly supply check",
      "flags": 2
    }
  ]
}
```

Date formats accepted by the event system:

- `YYYY-MM-DD` for one specific day.
- `*-MM-DD` for yearly recurring events.
- `*-*-DD` for monthly recurring events.

## Player Stats

Path example: `Export/Courier-PlayerStats.json`

```json
{
  "version": 1,
  "kind": "PlayerStats",
  "playerName": "Courier",
  "dateTimeGame": "2281-01-21 14:05:00",
  "dateTimeReal": "2026-05-25 18:42:10",
  "data": {
    "2281-01-21": {
      "stats": {
        "0": 1,
        "1": 3,
        "2": 8
      },
      "quests": [
        "VMS01",
        "FFGoodsprings"
      ]
    },
    "2281-01-22": {
      "stats": {
        "4": 2,
        "6": 5
      },
      "quests": []
    }
  }
}
```

Stat keys are the numeric stat codes from `config/PipLog/stats/stats.ini`.

## Import Notes

Imports merge with existing saved Pip-Log data.

- Journal entries and player events with matching `id` values replace existing records.
- Journal entries and player events with missing or empty `id` values receive generated IDs.
- `flags` values are numeric in JSON.
- Datetimes use storage format: `yyyy-mm-dd hh:mm:ss`.
- Player stats merge by date. Matching stat codes are overwritten, unrelated local stat codes remain, and quest IDs are deduplicated.
- JSON should use expanded arrays and objects as shown above. Do not use the internal `@@@`, comma-separated quest lists, or `code:count` stat strings.
