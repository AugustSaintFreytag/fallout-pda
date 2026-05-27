# Roadmap

Goal: Port the Pip-Log mod from a plugin to a pluginless JIP script mod.
Original script files have been dumped to `reference/` directory.

- Port of the original Pip-Log mod
	- No scripted special events like discounts or casino days
	- No support for weekly recurring events
	- No support for sunrise/sunset time calculations
	- No support for daylight savings time

- [x] Port calendar view, including button features
- [x] Port event list
- [x] Port journal list
- [x] Test journal layouting with various line numbers
- [x] Ensure output formatted dates are dd.mm.yyyy
- [x] Establish interactivity for player events
- [ ] Fix computed events not being added
- [ ] Implement journal entry import from JSON file (invoked via event handler)
- [ ] Implement player stats import from JSON file (invoked via event handler)
- [x] Add player stat change tracking (see reference, uses `ShowOff:OnPCMiscStatChange`, increment per call)

- [ ] Event handler API to allow adding events and journal entries
- [x] Push vanilla content rect for notes text further down
- [ ] Enhance event listing (include system/player, improve formatting)
- [x] Add YSI icon support for all mod buttons
- [ ] Add content placeholders ("No events." and "No journal entries.")
