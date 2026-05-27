# The Compendium

A reference document for writing NVSE scripts in the style used by this mod. Covers data types, UDF conventions, array handling, string management, persistence, and project structure.

---

## Project

### File Layout

```
config/
    ModName.ini                   ; User-facing configuration (INI sections, key=value)
NVSE/
    plugins/
        scripts/
            ln_ScriptName.txt     ; Runs once on new game or game load (a script for gamemode use)
            gr_ScriptName.txt     ; Runs once per game start (a script for value changes, not for gamemode use)
    user_defined_functions/
        ModName/
            VerbNoun.gek  ; Each UDF is its own .gek file, subdirectories allowed for further hierarchy
```

---

## Language

### Data Types

Gamebryo/NVSE scripting has a small, fixed set of variable types.

| Type         | Meaning                              | Notes                                            |
|--------------|--------------------------------------|--------------------------------------------------|
| `short`      | 16-bit integer                       | Used for counts, flags, indices, booleans        |
| `float`      | 32-bit floating point                | Used for time values, chances, SPECIAL stats     |
| `ref`        | Reference to a game object or script | Used for actors, items, effects, and UDF handles |
| `string_var` | Heap-allocated string                | Must be manually freed with `Sv_Destruct`        |
| `array_var`  | Heap-allocated dynamic array or map  | Must be nulled with `Ar_Null` after use          |

> Booleans are `short` variables named with a `b` prefix, holding `0` (false) or `1` (true).

### Variable Assignment

There are two assignment forms:

| Form                | Use case                                 |
|---------------------|------------------------------------------|
| `type name = value` | Declaration + initialization (first use) |
| `let name := value` | Reassignment of an existing variable     |

```gek
short iCount = 0            ; declare
let iCount := iCount + 1    ; mutate
let iCount += 1             ; increment
let iCount *= 1             ; multiply
let aList[0] := value       ; list array assign
let aMap["key"] := value    ; map array assign
```

#### Eval

The `eval` keyword should be used whenever an evaluation requires implicit coercion, reads values from arrays, or performs anything more complex than a direct value comparison.

```gek
if eval bFlagsByType["master"]
    ; ...
elseif eval bFlagsByType["alternative"]
    ; ...
endif
```

It is also advised when a compound boolean expression is used as a condition:

```gek
if eval (iIndex == Ar_BadNumericIndex) || (Ar_HasKey aValues, iIndex) == 0
    ; ...
endif
```

Inline function calls and their parameters should always be wrapped in parentheses.

### Arrays

Arrays are typed as `array_var`. NVSE arrays are 0-indexed and dynamically sized.

#### Construction

```gek
let aArray := Ar_Construct "array"         ; numerically indexed array
let aMap   := Ar_Construct "stringmap"     ; string-keyed hash map
```

#### Common Operations

```gek
short iSize = Ar_Size aArray         ; -1 if uninitialized/null
Ar_Append aArray, someValue          ; push to end
short iIndex = Ar_Find value, aArray ; returns Ar_BadNumericIndex if not found
short bHas = Ar_HasKey aArray, iKey  ; 1 if key exists
```

#### Checking Initialization

`Ar_Size` returns `-1` for a null/uninitialized array.

```gek
short iSize = Ar_Size aArray
if iSize == -1
    let aArray := Ar_Construct "array"
endif
```

#### Foreach Loop

A foreach loop will *always* produce an `array_var aEntry`, a string map with keys "key" and "value".

```gek
array_var aEntry
foreach aEntry <- aRecoveryMessageQueue
	short iKey = aEntry["key"] ; if needed
    string_var sValue = aEntry["value"]
    ; ...
loop
```

For form lists:

```gek
ref rEffect
foreach rEffect <- TTWAddictionWithdrawalList
    ; ...
loop
```

#### Freeing Arrays

Arrays are heap-allocated. After you're done with one, assign `Ar_Null` to release it:

```gek
let aArray := Ar_Null
```

### Strings

Strings are heap-allocated. Always call `Sv_Destruct` on them when done.

```gek
string_var sName = GetName rEffect
; use sName ...
Sv_Destruct sName
```

Multiple strings can be destroyed in one call:

```gek
Sv_Destruct sEffectName, sItemName
```

#### String Manipulation

```gek
Sv_Replace " Withdrawal|", sItemName    ; strip " Withdrawal" from sItemName (in-place)
string_var sSlice = sMessage[1:-2]      ; Python-style slice notation
```

#### String Interpolation

Prefix a `string_var` with `$` to interpolate it in a command that expects a plain string:

```gek
MessageBoxEx ($sMessage) sItemName
```

#### String Comparison

String comparison *must* be performed with `eval`.

```gek
if eval sLeft == sRight
    ; ...
endif
```

#### Format Specifiers in `PrintC` / `MessageBoxEx`

| Specifier | Meaning                               |
|-----------|---------------------------------------|
| `%z`      | `string_var` argument                 |
| `%g`      | numeric argument (`short` or `float`) |
| `%.2f`    | float with 2 decimal places           |

### UDFs

#### Declaration

Every UDF begins with `begin function { ... }` and ends with `end`. Parameters are declared inline in the header.

```gek
begin function { ref rEffect, float fDaysSinceLastActivation }
    ; body
end
```

A UDF with no parameters uses an empty block:

```gek
begin function {}
    ; body
end
```

#### Return Values

Use `SetFunctionValue` to return a value. A `return` statement exits early but does not return a value. Without a preceding `SetFunctionValue`, the caller gets 0 (the null value). Clean-up logic can be run after `SetFunctionValue` before a `return` statement.

```gek
SetFunctionValue fFinalChance
```

`SetFunctionValue` can return any type — `short`, `float`, `ref`, `string_var`, or `array_var`.

#### Importing a UDF

UDFs are loaded by `CompileScript` using the relative path from `user_defined_functions/`. The result is stored in a `ref`.

```gek
ref TickTargetEffectUDF = CompileScript "SaintsTransientAddictions\STAd_TickTargetEffect.gek"
```

This is always done at the top of the function, in a `; UDFs` section.

#### Calling a UDF

```gek
Call TickTargetEffectUDF rEffect
```

When the return value is needed, wrap in parentheses:

```gek
float fRecoveryChance = (Call GetEffectRecoveryChanceUDF fDaysSinceLastActivation)
```

Or use `let` for assignment after initialization:

```gek
let fTimeChanceFactor := (Call SomeUDF someArg)
```
#### Expressions

Condensed from the GECK NVSE Expressions operator table. ([geckwiki.com][1])

| Symbol    | Precedence | Function                               | Description |
|-----------|-----------|----------------------------------------|--------------|
| `:=`      |          0 | Assignment                             | Assigns right expression to the left variable/array element; right-associative and supports chained assignment.     |
| `\|\|`    |          1 | Logical Or                             | True if either side is true; in assignment context returns the first useful numeric value rather than only boolean. |
| `&&`      |          2 | Logical And                            | True if both sides are true; in assignment context returns `0` if left is false, otherwise the right numeric value. |
| `+=`      |          2 | Add and Assign                         | Adds right expression into the left variable/array element.                                                         |
| `-=`      |          2 | Subtract and Assign                    | Subtracts right expression from the left variable/array element.                                                    |
| `*=`      |          2 | Multiply and Assign                    | Multiplies the left variable/array element by the right expression.                                                 |
| `/=`      |          2 | Divide and Assign                      | Divides the left variable/array element by the right expression.                                                    |
| `^=`      |          2 | Exponent and Assign                    | Raises the left variable/array element to the right expression’s power.                                             |
| `\|=`     |          2 | Bitwise Or and Assign                  | Integer-demotes operands, bitwise-ORs them, and assigns the result left.                                            |
| `&=`      |          2 | Bitwise And and Assign                 | Integer-demotes operands, bitwise-ANDs them, and assigns the result left.                                           |
| `%=`      |          2 | Modulo and Assign                      | Computes integer remainder and assigns it left.                                                                     |
| `:`       |          3 | Slice/Range                            | Selects an inclusive string/array range; strings support negative indices.                                          |
| `::`      |          3 | Make Pair                              | Creates a key-value pair.                                                                                           |
| `==`      |          4 | Equality                               | True if comparable operands are equal; arrays compare shallow surface contents in modern xNVSE.                     |
| `!=`      |          4 | Inequality                             | True if operands are not equal.                                                                                     |
| `>`       |          5 | Greater Than                           | True if left is greater than right; string comparison is case-insensitive.                                          |
| `<`       |          5 | Less Than                              | True if left is less than right; string comparison is case-insensitive.                                             |
| `>=`      |          5 | Greater or Equal                       | True if left is greater than or equal to right; string comparison is case-insensitive.                              |
| `<=`      |          5 | Less or Equal                          | True if left is less than or equal to right; string comparison is case-insensitive.                                 |
| `\|`      |          6 | Bitwise Or                             | Integer-demotes operands and performs bitwise OR.                                                                   |
| `&`       |          7 | Bitwise And                            | Integer-demotes operands and performs bitwise AND.                                                                  |
| `<<`      |          8 | Binary Left Shift                      | Integer-demotes operands and shifts left by the right operand’s bit count.                                          |
| `>>`      |          8 | Binary Right Shift                     | Integer-demotes operands and shifts right by the right operand’s bit count.                                         |
| `+`       |          9 | Addition/Concatenation                 | Adds numbers or concatenates strings.                                                                               |
| `-`       |          9 | Subtraction                            | Subtracts right operand from left.                                                                                  |
| `*`       |         10 | Multiplication                         | Multiplies operands.                                                                                                |
| `/`       |         10 | Division                               | Divides left operand by right.                                                                                      |
| `%`       |         10 | Modulo                                 | Returns integer division remainder.                                                                                 |
| `^`       |         11 | Exponentiation                         | Raises left operand to the right operand’s power.                                                                   |
| unary `-` |         12 | Negation                               | Returns the numeric opposite.                                                                                       |
| `$`       |         12 | Stringize                              | Converts expression to string, shorthand for `ToString`.                                                            |
| `#`       |         12 | Numericize                             | Converts string to number, shorthand for `ToNumber`.                                                                |
| unary `*` |         12 | Dereference/Unbox                      | Unboxes an array, preferring StringMap `"value"` or otherwise the first element.                                    |
| unary `&` |         12 | Box                                    | Wraps any value in a one-element array.                                                                             |
| `!`       |         13 | Logical Not                            | Returns the boolean inverse.                                                                                        |
| `( )`     |         14 | Parentheses                            | Groups expressions and overrides precedence.                                                                        |
| `[ ]`     |         15 | Subscript                              | Accesses an array key or single string character; bracket contents are treated as parenthesized.                    |
| `->`      |         15 | Member Access                          | Reads a StringMap value by key; `dict->key` equals `dict["key"]`.                                                   |
| `.`       |         15 | Quest Variable Access / Reference Call | Calls a function on a reference or accesses a quest script variable; supports chained dot syntax.                   |
| `{ }`     |         16 | Function Args                          | Declares accepted argument variables for a UDF or lambda.                                                           |


#### Limitations

A single line cannot exceed a maximum of 512 characters. Inline functions (like lambdas and callbacks) are squashed by a precompile step and count as a single line. It is advised to split logic into smaller chunks.


## APIs

### Auxiliary Variables

Aux vars are the primary persistence mechanism for ESPless mods. They are stored on an actor (typically `Player`) and can be defined to either store their data in the game save or be transient.

#### Scalar Values

```gek
; Read
short iValue = Player.AuxiliaryVariableGetFloat "KeyName"

; Write
Player.AuxiliaryVariableSetFloat "KeyName" iValue

; Erase
Player.AuxiliaryVariableErase "KeyName"
```

#### Arrays

Aux vars store arrays only in list form. `AuxiliaryVariableSetFromArray` will only use values from its supplied array, not the keys.

```gek
; Read
array_var aArray = Player.AuxiliaryVariableGetAsArray "KeyName"

; Write
Player.AuxiliaryVariableSetFromArray "KeyName" aArray
```

#### String Map Arrays

A `"stringmap"` array can be stored and retrieved using the following undocumented functions:

```gek
; Read
array_var aStringMap = Player.AuxStringMapArrayGetAsArray "KeyName"

; Write
Player.AuxStringMapArraySetFromArray "KeyName" aStringMap
```

For completeness, the following undocumented string map functions exist.

```
AuxStringMapArrayGetSize (AuxStringMapSize)
AuxStringMapArrayGetValueType (AuxStringMapGetValueType)
AuxStringMapArrayGetFloat (AuxStringMapGetFlt)
AuxStringMapArrayGetRef (AuxStringMapGetRef)
AuxStringMapArrayGetString (AuxStringMapGetStr)
AuxStringMapArrayGetFirst (AuxStringMapFirst)
AuxStringMapArrayGetNext (AuxStringMapNext)
AuxStringMapArrayGetKeys (AuxStringMapKeys)
AuxStringMapArrayGetAll (AuxStringMapGetAll)
AuxStringMapArrayGetAsArray (AuxStringMapGetAsArr)
AuxStringMapArraySetFromArray (AuxStringMapSetFromArr)
AuxStringMapArraySetFloat (AuxStringMapSetFlt)
AuxStringMapArraySetRef (AuxStringMapSetRef)
AuxStringMapArraySetString (AuxStringMapSetStr)
AuxStringMapArrayEraseKey (AuxStringMapEraseKey)
AuxStringMapArrayValidateValues (AuxStringMapValidateVals)
AuxStringMapArrayDestroy (AuxStringMapDestroy)
```

#### Reference Map

A RefMap is a special type of array variable, where the keys are refs. In contrast to regular aux vars, a ref map is not owned by a reference like `Player`, it is defined as-is. This type may be used to store values indexed by game objects (like storing data indexed by placed statics, e.g. stock for placed vending machines).

The following functions exist:

```gek
RefMapArrayDestroy arrName:string 
RefMapArrayErase arrName:string baseForm:ref
RefMapArrayGetSize arrName:string
RefMapArrayGetKeys arrName:string 
RefMapArrayGetRef arrName:string baseForm:ref
RefMapArraySetRef arrName:string value:ref baseForm:ref
RefMapArrayGetFloat arrName:string baseForm:ref 
RefMapArraySetFloat arrName:string value:float baseForm:ref
RefMapArrayGetString arrName:string baseForm:ref
RefMapArraySetString arrName:string value:string baseForm:ref
```

It can also be probed with `(type:int) reference.RefMapArrayGetType arrName:string baseForm:ref` where the returned `short iType` is 0 = None, 1 = Float, 2 = Form, 4 = String.

#### Key Naming

- Every mod typically defines a disambiguation prefix from its initials.
- A mod "Saint's Transient Addictions" may choose "STAd" from its name.
- An example key may be named in the format: `"STAd_KeyName"`

A prefix character may define whether the aux var is private/public and temporary/persistent. It's best practice to define all aux vars as mod-private (not public).

| Prefix | Example      | Persistent | Public |
|--------|--------------|------------|--------|
| (None) | `STAd_Key`   | Yes        | No     |
| `*`    | `*STAd_Key`  | No         | No     |
| `_`    | `_STAd_Key`  | Yes        | Yes    |
| `*_`   | `*_STAd_Key` | No         | Yes    |

Persistent aux values are stored with the game save, temporary values are not. Nothing should be made persistent if not necessary for the functionality of the mod.

### INI Configuration

#### Reading Values

```gek
float fThreshold = GetINIFloat_Cached "Section:fKey" "ModName.ini"
short iInterval = GetINIFloat_Cached "General:iTickInterval" "ModName.ini" 15
string_var sSetting = GetINIString_Cached "General:sSetting" "ModName.ini" "Fallback"
```

The third argument is the default value if the key is missing. The fallback is optional and can be omitted.

> **Note:** Despite the function being named `GetINIFloat_Cached`, it is used for both `float` and `short` values. The value is coerced on assignment.

#### Reading Entire Sections

```gek
array_var aMessageMap = GetINISection "Messages" "ModName.ini"

foreach array_var aEntry <- aMessageMap
    string_var sKey = aEntry["key"]
    string_var sValue = aEntry["value"]
    ; or
    string_var sValue = *aEntry
loop
```

### Event Handlers and Tick Loops

Registered in a game load/new game script (`ln_*.txt`):

```gek
; Event-driven — fires when the player activates an inventory item
SetEventHandlerAlt "ShowOff:OnPreActivateInventoryItem" OnItemActivationUDF

; Game main loop — fires every iTickFrames frames
SetGameMainLoopCallback TickAllPlayerEffectsUDF 1 iTickFrames iModeFlag

; Game main loop interval
; 1 = every frame
; 30 = twice per second at 60fps
; 60 = once per second at 60fps

; Game main loop mode flag
; 1 = Game mode (not paused)
; 2 = Menu mode (paused or in Pip-Boy)
; 3 = Game mode and menu mode (default)
```

Game loops set up in an `ln_` entrypoint script are the primary way to port `begin gamemode` and `begin menumode` type scripts to pure JIP scripts. In the vanilla plugin system, a quest is created with an associated quest script that may then include either a `gamemode` or a `menumode` block that is run once per frame by default. Vanilla scripts with a `begin function` block are intended to be called by their `scn ...` name from another script. 

It should be noted that imports (e.g. the `CompileScript` block) are not used in vanilla plugin scripts -- all scripts are precompiled and globally available. In the JIP context, they require an import, once per file, to get an active `ref ...` reference to the compiled function.

### Game Time

```gek
; Days elapsed since a given real-world date (used as in-game epoch)
float fDaysPassed = GetGameDaysPassed 2277 08 17
```

The date `2277 08 17` is the canonical in-game epoch for the Fallout universe.

---

### Porting

A vanilla plugin script would define state as variables on a quest script -- these become properties that are called on a persistent reference. For example, a quest `QSTTrackDrinks` may have an associated script `TrackDrinksSCRIPT`. The script would define persistent values are follows:

```gek
scn TrackDrinksSCRIPT

short iNumDrinks = 0

begin gamemode {}
    ; Logic
end
```

Because the script is directly associated with the quest, other scripts may now access these values like properties on the quest ref.

```gek
short iNumDrinks = QSTTrackDrinks.iNumDrinks
```

```gek
; Player has used drink item, increment tracker.
let QSTTrackDrinks.iNumDrinks += 1
```

The equivalent in a JIP script would be to use auxiliary variables instead. No quests are needed, however, when porting from vanilla plugin scripts to JIP, it needs to be clear which scripts used to belong to which quest in order to translate a prior property access to an aux var.

```gek
; Vanilla:
short iNumDrinks = QSTTrackDrinks.iNumDrinks

; JIP:
short iNumDrinks = Player.AuxiliaryVariableGetFloat "ModPrefix_iNumDrinks"
```

```gek
; Vanilla:
let iNumDrinks += QSTTrackDrinks.iNumDrinks

; JIP:
short iNumDrinks = Player.AuxiliaryVariableGetFloat "ModPrefix_iNumDrinks"
Player.AuxiliaryVariableSetFloat "ModPrefix_iNumDrinks" (iNumDrinks + 1)
```

---

## Style

### Code

Use descriptive variable names, refrain from using abbreviations. A variable name or aux var key should broadly be self-explanatory. When reading aux vars into variables, use the same name for the variable as was used for the key after the prefix.

Exclusively use tabs. Do not use spaces to indent for horizontal alignment, e.g. consecutive lines of variable declarations.

There should be empty lines between major blocks of logic for legibility. There should always be an empty line before and after conditionals and similar blocks, including return statements. At the same time, orphan lines within a block should also be avoided, i.e. no empty line between two singular statements in a block. There should be no empty line orphans, i.e. right after an opening `if` or right before a closing `endif`. There should always be an empty line inside the outermost `begin function` block in UDFs.

### Naming Conventions

| Thing                   | Convention                   | Example                                       |
|-------------------------|------------------------------|-----------------------------------------------|
| UDF files               | `VerbNoun.gek`               | `TickTargetEffect.gek`                        |
| Game load scripts       | `ln_ModName.txt`             | `ln_SaintsTransientAddictions.txt`            |
| Game start scripts      | `gr_PatchName.txt`           | `gr_HitDrugsAddictionWithdrawalListPatch.txt` |
| UDF `ref` variables     | PascalCase + `UDF` suffix    | `TickTargetEffectUDF`                         |
| Persistent aux var keys | `"Prefix_KeyName"`           | `"STAd_LastActivationMapKeys"`                |
| Transient aux var keys  | `"*Prefix_KeyName"`          | `"*STAd_RecoveryMessageQueue"`                |
| Variables               | Hungarian prefix + camelCase | `fDaysSinceLastActivation`                    |

When reading aux vars into a variable, it is recommended to maintain name parity, like `float fSpecialValue = AuxiliaryVariableGetFloat "*MOD_fSpecialValue"`.

### Hungarian Notation

All variable names use a consistent Hungarian-style prefix:

| Prefix | Type              | Example                              |
|--------|-------------------|--------------------------------------|
| `i`    | `short`           | `iNumMessages`, `iIndex`             |
| `b`    | `short` (boolean) | `bDidRunSinceLastGameLoad`           |
| `f`    | `float`           | `fDaysPassed`, `fRecoveryChance`     |
| `r`    | `ref`             | `rEffect`, `rBaseItem`               |
| `s`    | `string_var`      | `sEffectName`, `sItemName`           |
| `a`    | `array_var`       | `aMessages`, `aRecoveryMessageQueue` |

Note that all `array_var` values use the `a` prefix, including string maps.

### Section Structure

Every UDF is divided into labeled sections using inline comments. Standard order:

```gek
begin function { ... }

    ; UDFs

    ref SomeHelperUDF = CompileScript "..."

    ; Config

    float fSomeThreshold = GetINIFloat_Cached "Section:fKey" "Mod.ini"

    ; State

    short bSomeFlag = SomeActor.AuxiliaryVariableGetFloat "FlagKey"

    ; Main

    ; ... logic ...

    ; Clean-Up

    Sv_Destruct sVar1, sVar2
    let aArray := Ar_Null

end
```

Not all sections appear in every UDF — only include what is relevant.

### Encapsulation

Try to follow the principles of functional programming. Logic that could naturally be displaced into its own function can be put into a UDF, in its own file. 
Reused logic that would otherwise have to be duplicated may either be put into a UDF file *or* alternatively into a local closure.

```gek
ref LocalClosureUDF = (begin function {} 
	; Custom logic
	SetFunctionValue 1
	return
end)

short bLocalResult = Call LocalClosureUDF
```

### Debugging

All debug output uses `PrintC` (formatted) or `Print` (plain). Debug lines are **commented out in production** with a leading `;`:

```gek
;PrintC "Player recovered from %z Addiction. Rolled %.2f/%.2f." sItemName fRandom fRecoveryChance
```

Active (uncommented) `PrintC`/`Print` calls are used only for genuine warnings or critical errors:

```gek
PrintC "Critical integrity error in key/value map (size mismatch, %g vs. %g). Resetting." iKeySize iValueSize
```

---

## Reference

### Example UDF

```gek
begin function { ref rSomeRef, float fSomeFloat }

    ; UDFs

    ref SomeHelperUDF = CompileScript "ModName\HelperName.gek"

    ; Config

    float fThreshold = GetINIFloat_Cached "General:fThreshold" "ModName.ini" 1.0

    ; State

    array_var aData = Player.AuxiliaryVariableGetAsArray "Prefix_DataKey"
    short iSize = Ar_Size aData

    if iSize == -1
        let aData := Ar_Construct "array"
    endif

    ; Main

    string_var sName = GetName rSomeRef
    float fResult = (Call SomeHelperUDF sName fSomeFloat)

    ;PrintC "Result for '%z': %.2f" sName fResult

    SetFunctionValue fResult

    ; Clean-Up

    Sv_Destruct sName
    let aData := Ar_Null

end
```
