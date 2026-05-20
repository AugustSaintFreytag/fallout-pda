# Roadmap

Goal: Port the Pip-Log mod from a plugin to a pluginless JIP script mod.
Original script files have been dumped to `reference/` directory.

- Port of the original Pip-Log mod
	- No scripted special events like discounts or casino days
	- No support for weekly recurring events
	- No support for sunrise/sunset time calculations
	- No support for daylight savings time

- Ensure output formatted dates are dd.mm.yyyy
- Event handler API to allow adding events and journal entries
- JSON-backing of journals for backup/restore purposes