## View Model

`iViewMode`						; 0 = None/vanilla views, 1=calendar view, 2 = events view, 3 = journal view

View mode is set from mode buttons (e.g. the Log, Notes, Events, Journal buttons). The selected view mode determines which buttons to display or hide or what labels to set them to, as well as what sub-parts of the UI control loop to run.

`iCalendarYear`					; Year the calendar is set to view
`iCalendarMonth`				; Month the calendar is set to view

`sHoveredCalendarDate`			; Date string of the hovered calendar tile (precedence over selected, unset when not hovering)
`sSelectedCalendarDate`			; Date string of the clicked calendar tile (auto-set to today's date string)

`sCalendarDateSummary`			; Text to be displayed in `CM_DataTextRect/CM_DataText/string`, set to placeholder tutorial text when no date is hovered, otherwise generated day summary

The calendar summary string is generated when a calendar tile is clicked and will automatically be picked up by the next UI control loop. It must not be regenerated every frame in a menu mode loop.

Some additional flags can gate against constant redrawing in the UI control loop.

`bIsCalendarViewModified`		; Dirty flag for calendar tiles
`bIsCalendarSummaryModified`	; Dirty flag for selected date summary text
`bIsCalendarSelectionModified`	; Dirty flag for an individual tile's visual selection

## Events

Assuming an event is a structure:

```
	structure Event
		string_var id			; UUID
		string_var date			; 
		string_var name			; Global or player-provided event name
```

We can use `Sv_Join` and `Sv_Split` to establish basic encoding. This makes our complex string map array event data type into an easy to handle string.

```gek
array_var aEvent = Ar_Construct "stringmap"

let aEvent["id"] := GetRandomId
let aEvent["date"] := "2077-12-01"
let aEvent["name"] := "Festival of Suns"
```

```gek
ref EncodeEventToString = (begin function { array_var aEvent } 
	array_var aComponents = (Ar_List aEvent["id"], aEvent["name"], aEvent["date"])
	string_var sEncodedEvent = (Sv_Join aComponents, "@@@")

	SetFunctionValue sEncodedEvent
	let aComponents := Ar_Null
end)

ref DecodeEventFromString = (begin function { string_var sData } 
	array_var aComponents = (Sv_Split sData, "@@@")
	array_var aEvent = Ar_Construct "stringmap"

	; TODO: Safeguard with `Ar_Size` check and `(TypeOf aComponents[0]) == "String"`, etc.

	let aEvent["id"] := aComponents[0]
	let aEvent["date"] := aComponents[1]
	let aEvent["name"] := aComponents[2]

	SetFunctionValue aEvent
	let aComponents := Ar_Null
end)
```

We can then decide to store global events by id in encoded form and have "events by time"-type arrays store a time -> event id relationship. Event ids are predictable values that can be easily encoded as comma-separated values, e.g. "id,id,id".

```
array_var aGlobalEventsById; string map, event id -> encoded event
array_var aPlayerEventsById; string map, event id -> encoded event

array_var aGlobalEventsByDate; string map, iso date string -> encoded event id list
array_var aPlayerEventsByDate; string map, iso date string -> encoded event id list
```

If we then wanted to get all events for, e.g. a month, we could query `Ar_Keys` and filter them.


## Journal Entries

Assuming a journal entry is a structure:

```
	structure JournalEntry
		string_var id			; UUID
		string_var flags 		; Bit mask, parses to `short`, reserved
		string_var datetime 	; String-encoded date
		string_var location 	; Textual representation of player location when written
		string_var text 		; Player journal text
```

We can employ the same encoding/decoding approach as we do for events. Journal entries are stored as:

```
array_var aPlayerJournal; array list of encoded journal entries
```