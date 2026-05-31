# Quickstart

This guide wires the manager into a Pict application: register the provider and
the views, expose the DOM slots they need, inject a schema list, and read the
saved connections back out.

## 1. Install

```bash
npm install pict-meadow-connection-manager
```

`pict-section-connection-form` comes along as a dependency and is re-exported
as `libMCM.PictSectionConnectionForm`, so a single `require()` brings in the
whole stack.

## 2. Register the provider

The provider is a **singleton** — the list view, the detail view, and the form
view all read the same state, so a fresh instance per view would mean three
independent drafts.

```javascript
const libMCM = require('pict-meadow-connection-manager');

this.pict.addProviderSingleton('MeadowConnectionManager',
	libMCM.PictProviderConnectionManager.default_configuration,
	libMCM.PictProviderConnectionManager);
```

To point the Test button at a server endpoint, override
`TestConnectionEndpoint` at registration:

```javascript
let tmpProviderConfiguration = Object.assign({},
	libMCM.PictProviderConnectionManager.default_configuration,
	{
		TestConnectionEndpoint: '/test-connection'
	});
this.pict.addProviderSingleton('MeadowConnectionManager',
	tmpProviderConfiguration,
	libMCM.PictProviderConnectionManager);
```

## 3. Register the manager-shell views

```javascript
this.pict.addView('MCM-ConnectionList',
	libMCM.PictViewConnectionList.default_configuration,
	libMCM.PictViewConnectionList);
this.pict.addView('MCM-ConnectionDetail',
	libMCM.PictViewConnectionDetail.default_configuration,
	libMCM.PictViewConnectionDetail);
```

Both views ship their own CSS (theme-token driven) and self-register it. Their
default destinations are `#MCM-ConnectionList-Container` and
`#MCM-ConnectionDetail-Container` — your layout must expose both elements.

## 4. Register the shared form view into the detail slot

The detail view renders a `<section id="MCM-ConnectionConfig-Container">`
placeholder. Register `pict-section-connection-form` so it mounts there:

```javascript
this.pict.addView('PictSection-ConnectionForm',
	Object.assign({}, libMCM.PictSectionConnectionForm.default_configuration,
		{
			ContainerSelector:         '#MCM-ConnectionConfig-Container',
			DefaultDestinationAddress: '#MCM-ConnectionConfig-Container',
			SchemasAddress:            'AppData.MCM.Schemas',
			ActiveAddress:             'AppData.MCM.CurrentConnection.Type',
			FieldIDPrefix:             'mcm-conn',
			ShowProviderSelect:        false
		}), libMCM.PictSectionConnectionForm);
```

The overrides:

- **`ContainerSelector` / `DefaultDestinationAddress`** — the slot the detail view exposes.
- **`SchemasAddress: 'AppData.MCM.Schemas'`** — the same address the provider writes to in `setSchemas()`.
- **`ActiveAddress: 'AppData.MCM.CurrentConnection.Type'`** — the field the detail view's Type `<select>` writes to, so the form swaps provider blocks when the user picks a different type.
- **`FieldIDPrefix: 'mcm-conn'`** — each input's DOM id becomes `mcm-conn-<provider>-<field>`, so multiple connection forms can coexist on one page.
- **`ShowProviderSelect: false`** — the detail view already renders the Type select, so the form hides its own.

If you use a non-default provider `AppDataAddress`, point `SchemasAddress` and
`ActiveAddress` at the matching prefix. See
[pict-section-connection-form](https://fable-retold.github.io/pict-section-connection-form/)
for the full list of form-view options.

## 5. Lay out the slots

The views render into plain elements. Place them however your layout dictates —
the only contract is the element ids:

```html
<div id="MCM-ConnectionList-Container"></div>
<div id="MCM-ConnectionDetail-Container"></div>
```

`#MCM-ConnectionConfig-Container` is rendered *inside* the detail view's
template, so you do not place it yourself.

## 6. Inject schemas

This module never fetches schemas. Your application fetches them — typically
from a server endpoint backed by
`meadow-connection-manager.getAllProviderFormSchemas()` — and hands the array
to the provider:

```javascript
fetch('/myapp/connection/schemas').then((pResponse) => pResponse.json()).then((pPayload) =>
{
	this.pict.providers.MeadowConnectionManager.setSchemas(pPayload.Schemas || []);
	this.pict.views['MCM-ConnectionList'].render();
});
```

`setSchemas()` mirrors the array into `AppData.<addr>.Schemas`, rebuilds the
type `<select>` source `AppData.<addr>.ConnectionTypes`, hands the schemas to
the form view if it is already mounted, defaults the in-progress connection's
type to the first schema, and refreshes the views.

Each schema follows the same `{ Provider, DisplayName, Fields }` contract the
`Meadow-Connection-<Type>-FormSchema.js` files export. A minimal pair:

```javascript
let tmpSchemas =
[
	{
		Provider:    'SQLite',
		DisplayName: 'SQLite',
		Fields:
		[
			{ Name: 'SQLiteFilePath', Label: 'Database File Path', Type: 'Path', Default: ':memory:', Required: true }
		]
	},
	{
		Provider:    'MySQL',
		DisplayName: 'MySQL',
		Fields:
		[
			{ Name: 'host',     Label: 'Server',   Type: 'String',   Default: '127.0.0.1', Required: true },
			{ Name: 'port',     Label: 'Port',     Type: 'Number',   Default: 3306,        Required: true },
			{ Name: 'user',     Label: 'User',     Type: 'String',   Default: 'root',      Required: true },
			{ Name: 'password', Label: 'Password', Type: 'Password' },
			{ Name: 'database', Label: 'Database', Type: 'String',   Default: 'meadow',    Required: true }
		]
	}
];
this.pict.providers.MeadowConnectionManager.setSchemas(tmpSchemas);
```

## 7. Seed saved connections (optional)

The provider hydrates its list from `fable.settings.MeadowConnections` at
`onInitialize` if that array is present. In a Pict application, the array comes
from the configuration JSON's `pict_configuration.MeadowConnections`:

```json
"pict_configuration":
{
	"Product": "MyApp",
	"MeadowConnections":
	[
		{
			"Name": "Bookstore MySQL",
			"Type": "MySQL",
			"Config": { "host": "127.0.0.1", "port": 3306, "user": "root", "database": "bookstore" },
			"Status": "unknown"
		}
	]
}
```

Each entry is a `{ Name, Type, Config, Status }` record — the same shape the
provider stores in `AppData.<addr>.Connections`. To load connections from a
server instead, call `provider.setConnections(pArray)`.

## 8. Read connections back out

The provider exposes the saved list through its CRUD surface:

```javascript
let tmpProvider = this.pict.providers.MeadowConnectionManager;

let tmpAll = tmpProvider.getConnections();        // the full array
let tmpOne = tmpProvider.getConnection(0);        // by index, or null

// Mask passwords before logging or display:
let tmpSafe = tmpProvider.maskConfig(tmpOne.Type, tmpOne.Config);
```

To POST the entire list somewhere on save, read `getConnections()` and send it;
the manager does not persist for you.

## What you get

With the provider and three views registered and a schema list injected, the
following workflow is wired with no further code:

- **Add Connection** — appends a new draft (defaulted from the first schema) and opens the editor.
- **Edit** — loads a saved connection into the editor; the form block populates from the saved config.
- **Type select** — switches the active provider; the form block swaps and reseeds to the new type's defaults.
- **Save** — pulls the live form values, validates required fields, and writes back into the saved list.
- **Test Connection** — POSTs `{ Type, Config }` to `TestConnectionEndpoint` and updates the status badge.
- **Remove** — deletes a connection and fixes up the selection.

See [Architecture](architecture.md) for the full provider API and the
view-to-provider wiring.
