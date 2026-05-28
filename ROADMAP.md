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
- [x] Fix computed events not being added
- [x] Implement journal entry import from JSON file (invoked via event handler)
- [x] Implement player stats import from JSON file (invoked via event handler)
- [x] Add player stat change tracking (see reference, uses `ShowOff:OnPCMiscStatChange`, increment per call)
- [x] Use same textual date for journal entries (a la 12th September, 2277)
- [x] Check whether tracked player stats are stored/read correctly
- [x] Fix selected date text view not being scrollable (likely an XML layout constraints issue)
- [ ] Fix date format import for journal entries
- [ ] Implement journal entry sorting (button functionality)

- [x] Display messages to the player with custom icon for daily events
- [x] Add indicator to date tiles that have events
- [x] Event handler API to allow adding events and journal entries
- [x] Push vanilla content rect for notes text further down
- [x] Add YSI icon support for all mod buttons
- [ ] Restyle selected event details to use same font size as calendar view
- [ ] Enhance event listing (include system/player, improve formatting)
- [ ] Add content placeholders ("No events." and "No journal entries.")
