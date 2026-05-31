# Architecture

`pict-meadow-connection-manager` is three Pict pieces plus a re-export:

- **`PictProviderConnectionManager`** — the state hub (CRUD + schemas + testing).
- **`PictViewConnectionList`** — the saved-connection list.
- **`PictViewConnectionDetail`** — the single-connection editor that exposes the form slot.
- **`PictSectionConnectionForm`** — a convenience re-export of `pict-section-connection-form`.

```
┌────────────────────────────────────────────────────────────┐
│  pict-meadow-connection-manager                              │
│   ├─ PictProviderConnectionManager   state + CRUD + tests    │
│   ├─ PictViewConnectionList          list of saved conns     │
│   └─ PictViewConnectionDetail        editor (name + slot)    │
│                                              │               │
│                                              ▼ slot          │
│                              pict-section-connection-form    │
│                                schema-driven form            │
└────────────────────────────────────────────────────────────┘
```

The provider is the single source of truth. Both views read and write its
state through `pict.AppData`, and the provider hands the schema list to the
form view. Everything is browser-only — there is no server code in this module.

## The provider

`PictProviderConnectionManager` extends
[`pict-provider`](https://fable-retold.github.io/pict-provider/). Its default
configuration:

```javascript
{
	ProviderIdentifier:    'MeadowConnectionManager',
	AutoInitialize:        true,
	AutoInitializeOrdinal: 0,
	AppDataAddress:        'MCM',
	TestConnectionEndpoint: false
}
```

| Option | Default | Purpose |
|--------|---------|---------|
| `AppDataAddress` | `'MCM'` | The `AppData` key under which all manager state lives. Override it to run two managers on one page (e.g. one for "source", one for "target" connections). |
| `TestConnectionEndpoint` | `false` | URL that `testConnection()` POSTs to. When falsy, `testConnection()` returns a graceful "No test endpoint configured" result. |

The provider is normally registered with `addProviderSingleton`, because the
list view, the detail view, and the form view must all read the same draft.

### State shape

The provider lazily scaffolds its state on first access (`_getState()`). With
the default address it lives at `AppData.MCM`:

```javascript
AppData.MCM =
{
	Connections:       [],     // saved connections, each { Name, Type, Config, Status }
	SelectedIndex:     -1,     // index into Connections, or -1 for an unsaved draft
	CurrentConnection: { Name: '', Type: '', Config: {}, Status: 'new' },
	ConnectionTypes:   [],     // [{ TypeName, DisplayName }], rebuilt by setSchemas()
	Schemas:           []      // full schema list, mirrored by setSchemas()
};
```

- **`Connections`** — the persisted list. Hydrated at `onInitialize` from `fable.settings.MeadowConnections` (a deep copy) when that array is present.
- **`SelectedIndex`** — which row is being edited, or `-1` when the editor holds an unsaved draft.
- **`CurrentConnection`** — the active draft. `selectConnection()` deep-copies the chosen row into it, so edits do not mutate the saved list until Save.
- **`ConnectionTypes`** — `{ TypeName, DisplayName }` entries built from the schemas; this is the source for the detail view's Type `<select>`.
- **`Schemas`** — the full schema list; the form view reads its field definitions from this address.

### Schema injection

The host fetches the schema list and calls `setSchemas()`. This is the boundary
between "which providers exist" (server-driven) and "how to render their forms"
(the pure-presentation form view).

```javascript
setSchemas(pSchemas)
```

`setSchemas()` performs five steps:

1. Stores the array on the provider (`_Schemas`) and mirrors it into `AppData.<addr>.Schemas`.
2. Rebuilds `AppData.<addr>.ConnectionTypes` as `{ TypeName: Provider, DisplayName: DisplayName || Provider }` per schema.
3. If no type is chosen yet for an in-progress new connection, defaults `CurrentConnection.Type` to the first schema and seeds `CurrentConnection.Config` from `buildDefaultConfig()`.
4. Propagates the schemas to the form view (`pict.views['PictSection-ConnectionForm'].setSchemas(...)`) if it is already mounted.
5. Calls `refreshViews()`.

Schema lookups:

```javascript
getSchemas()                  // → object[]  the full list
getSchema(pTypeName)          // → object|null  matched by Provider
getAvailableTypes()           // → string[]  the Provider id of every schema
```

### Schema-driven helpers

Three helpers derive behaviour from a schema's `Fields` array. They honor
`Multiplier` (numeric unit conversion) and `MapTo` (one field targeting one or
more dotted config paths), the same field contract
[`pict-section-connection-form`](https://fable-retold.github.io/pict-section-connection-form/)
uses.

```javascript
buildDefaultConfig(pTypeName)        // → object
```

Walks the schema's fields and builds a config blob from each field's `Default`.
Numeric defaults are multiplied by `Multiplier` when present (e.g. a 120-second
default with `Multiplier: 1000` stores `120000`). Fields with no `Default` are
omitted. Values are written to each path in `MapTo` (or to `Name` when `MapTo`
is absent).

```javascript
validateConfig(pTypeName, pConfig)   // → { valid: boolean, errors: string[] }
```

Currently checks only `Required: true` fields. For each required field it reads
`MapTo[0]` (or `Name`) from the config; an undefined, null, or empty value adds
`"<Label> is required"`. An unknown type yields
`{ valid: false, errors: [ 'Unknown connection type: <type>' ] }`.

```javascript
maskConfig(pTypeName, pConfig)       // → object
```

Returns a deep copy with every `Type: 'Password'` field's non-empty value
replaced by `'***'`. Empty values are left alone. Useful before logging or
listing a config.

> These helpers replaced an earlier inline `ConnectionTypeRegistry`. The
> deprecated registry and the per-type config view subclasses are **not**
> exported — the test suite asserts they are `undefined` so hosts that still
> reference them fail loudly.

### CRUD and selection

```javascript
getConnections()             // → the Connections array
getConnection(pIndex)        // → a connection, or null when out of range

addConnection(pConnection)   // append (or a defaulted draft) then select; → new index
updateConnection(pIndex, pConnection)   // replace a row in place
removeConnection(pIndex)     // splice, then fix up SelectedIndex

selectConnection(pIndex)     // deep-copy a row into CurrentConnection; refresh detail
deselectConnection()         // clear selection + reset CurrentConnection to a blank draft
saveCurrentConnection()      // write CurrentConnection back to the selected row (or append)

setConnections(pArray)       // replace the whole list; clear selection
```

Notable behaviours, verified in source and tests:

- `addConnection()` with no argument builds a draft defaulted from the **first** schema (`buildDefaultConfig(firstProvider)`), then selects it.
- `removeConnection()` clears the selection when the removed row was selected, or decrements `SelectedIndex` when an earlier row was removed.
- `selectConnection()` deep-copies into `CurrentConnection`, so editing the draft does not mutate the saved row until `saveCurrentConnection()`.
- `saveCurrentConnection()` writes back to `SelectedIndex` if it is valid, otherwise appends and points `SelectedIndex` at the new row.

### Connection testing

```javascript
testConnection(pConfig, fCallback)
```

`pConfig` is the draft (`{ Type, Config, ... }`). When `TestConnectionEndpoint`
is falsy, the callback gets `(null, { Success: false, Error: 'No test endpoint configured' })`.
Otherwise it POSTs `{ Type, Config }` as JSON:

- via the global `fetch` when available (response parsed as JSON and passed to the callback), or
- via `fable.RestClient.postJSON(...)` when there is no `fetch`, or
- failing both, calls back with `{ Success: false, Error: 'No HTTP client available' }`.

The detail view interprets the result: a `Success: true` response sets the
draft `Status` to `OK`, a non-success response sets it to `Failed`, and a
transport error sets it to `Error`.

### View refresh

The provider drives re-rendering through two methods that reference the
manager-shell views by their hashes:

```javascript
refreshViews()         // re-render MCM-ConnectionList, then refreshDetailView()
refreshDetailView()    // re-render MCM-ConnectionDetail when a row is selected;
                       // otherwise blank #MCM-ConnectionDetail-Container
```

These are the host's extension point: wrap them to also re-render anything else
that should stay in sync (a status bar, a custom side panel). The Bookstore
example monkey-patches both to also redraw a topbar slot.

## The list view

`PictViewConnectionList` extends
[`pict-view`](https://fable-retold.github.io/pict-view/) with
`ViewIdentifier: 'MCM-ConnectionList'` and a default destination of
`#MCM-ConnectionList-Container`. Its template record address is `AppData.MCM`,
so it reads `Connections` directly.

- The container template renders a header (with an **Add Connection** button calling `providers.MeadowConnectionManager.addConnection()`) and iterates `Record.Connections` with `{~TS:MCM-ConnectionList-Row:Record.Connections~}`.
- Each row shows Name, Type, and a Status badge (`data-status` styles `OK` / `Failed` / `Error` / `new` distinctly), plus **Edit** (`selectConnection(index)`) and **Remove** (`removeConnection(index)`) buttons.
- `onBeforeRender()` stamps each connection's array index onto the record as `Index` so the row buttons can reference it.
- `onAfterRender()` injects the view's CSS and calls `super`.

The view's CSS is theme-token driven (`var(--theme-color-*, …)` with hardcoded
fallbacks) and is registered automatically by the `pict-view` base class.

## The detail view

`PictViewConnectionDetail` (`ViewIdentifier: 'MCM-ConnectionDetail'`, default
destination `#MCM-ConnectionDetail-Container`) edits one connection. Its
template record address is `AppData.MCM.CurrentConnection`. It carries
`AutoRender: false` — the provider renders it on demand via `refreshDetailView()`.

The container template renders:

- a **Connection Name** input wired to `onNameChange(this.value)`;
- a **Connection Type** `<select>` populated by `{~TS:MCM-ConnectionDetail-TypeOption:AppData.MCM.ConnectionTypes~}` (each option is `{ TypeName, DisplayName }`) and wired to `onTypeChange(this.value)`;
- a **Status** badge bound to `Record.Status`;
- the form slot `<section id="MCM-ConnectionConfig-Container"></section>`;
- a footer with **Save** (`onSave()`), **Test Connection** (`onTest()`), and **Cancel** (`onCancel()`) buttons.

### Delegation to the form view

The detail view does **not** render any per-provider fields. That is the entire
point of the split: the `#MCM-ConnectionConfig-Container` slot is where the
shared `pict-section-connection-form` view (registered under the hash
`PictSection-ConnectionForm`) mounts.

The handoff happens in `onAfterRender()`:

1. Inject CSS.
2. Resolve the active type from `CurrentConnection.Type` (falling back to the first available type).
3. Sync the Type `<select>`'s DOM value to the active type.
4. Call `tmpFormView.setSchemas(provider.getSchemas())` so the form has the current schema list.
5. Call `tmpFormView.setValues(activeType, CurrentConnection.Config)` so the form shows the saved values.

This is belt-and-suspenders with the provider's own `setSchemas()` handoff — the
detail view re-pushes schemas each time it renders, so ordering between
provider injection and detail rendering does not matter.

### Action handlers

- **`onNameChange(pNewName)`** — writes `CurrentConnection.Name` directly.
- **`onTypeChange(pNewType)`** — sets `CurrentConnection.Type`, reseeds `CurrentConnection.Config` from `provider.buildDefaultConfig(pNewType)` (so old values from the previous type cannot leak), then calls the form view's `setActiveProvider(pNewType)` and re-applies the fresh defaults via `setValues(...)`.
- **`onSave()`** — pulls live form values via `formView.getProviderConfig()` into `CurrentConnection`, runs `provider.validateConfig(...)`, logs a warning and returns on failure, otherwise calls `provider.saveCurrentConnection()`.
- **`onTest()`** — collects form values the same way, then calls `provider.testConnection(...)` and sets `Status` to `OK` / `Failed` / `Error` based on the result before re-rendering.
- **`onCancel()`** — calls `provider.deselectConnection()`.

### Why the form view, not inline fields

The per-provider field definitions live in exactly one place — the
`Meadow-Connection-<Type>-FormSchema.js` file each `meadow-connection-*` module
exports. The server-side
[`meadow-connection-manager`](https://fable-retold.github.io/meadow-connection-manager/)
aggregates them with `getAllProviderFormSchemas()`, and
[`pict-section-connection-form`](https://fable-retold.github.io/pict-section-connection-form/)
renders whichever one is active. Putting field markup in this module would
duplicate that contract and drift from it. Instead this module owns only the
manager shell and slots the canonical form view in.

The form view's `getProviderConfig()` is also what understands the schema's
`Multiplier`, `MapTo`, and `OmitIfFalsy` semantics when reading values back out
of the DOM into the canonical wire format. The manager's provider applies the
same `Multiplier` / `MapTo` rules in `buildDefaultConfig()` so defaults and
collected values agree.

## Data flow at a glance

```
host fetch ──► provider.setSchemas(schemas)
                    │
                    ├─► AppData.MCM.Schemas        ──► form view field definitions
                    ├─► AppData.MCM.ConnectionTypes ──► detail view <select> options
                    └─► refreshViews()

user clicks Edit ──► provider.selectConnection(i)
                    └─► CurrentConnection (deep copy) ──► detail.render()
                            └─► onAfterRender: form.setSchemas + form.setValues

user edits + Save ──► detail.onSave()
                    ├─► form.getProviderConfig() ──► CurrentConnection.Config
                    ├─► provider.validateConfig()
                    └─► provider.saveCurrentConnection() ──► Connections[i]
```

## Related Modules

- [pict](https://fable-retold.github.io/pict/) — the MVC framework this module is built on.
- [pict-provider](https://fable-retold.github.io/pict-provider/) — base class for `PictProviderConnectionManager`.
- [pict-view](https://fable-retold.github.io/pict-view/) — base class for the list and detail views.
- [pict-section-connection-form](https://fable-retold.github.io/pict-section-connection-form/) — the schema-driven form view this module delegates field rendering to.
- [meadow-connection-manager](https://fable-retold.github.io/meadow-connection-manager/) — the server-side counterpart that aggregates the provider form schemas.
