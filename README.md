# pict-meadow-connection-manager

Browser-safe Pict provider plus list/detail views for managing **named**
meadow database connections. This module owns the *manager-shell* concerns -
a list of saved connections, a detail editor (Name + Type + Status + Save /
Test / Cancel), and a Pict provider for state and CRUD.

Per-provider field rendering is **not** in this module. It is delegated to
[`pict-section-connection-form`](https://fable-retold.github.io/pict-section-connection-form/),
which renders the field block for whichever provider type is active. The
per-provider field schemas come from that one canonical source - the
`meadow-connection-*` modules export them and the server-side
[`meadow-connection-manager`](https://fable-retold.github.io/meadow-connection-manager/)
aggregates them via `getAllProviderFormSchemas()`.

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

This module is **browser-only**. It has no knowledge of the
`meadow-connection-*` modules and does not fetch schemas itself. The host
application owns the fetch (typically `GET /<app>/connection/schemas`) and
hands the result to the provider via `setSchemas()`.

## Installation

```bash
npm install pict-meadow-connection-manager
```

Dependencies (from `package.json`):

- `pict-provider` ^1.0.13
- `pict-view` ^1.0.68
- `pict-section-connection-form` ^1.0.0
- `fable-serviceproviderbase` ^3.0.19

`pict-section-connection-form` is a hard dependency: the manager re-exports it
(`libMCM.PictSectionConnectionForm`) so hosts can register the whole stack from
one `require()`.

## Exports

The package `main` (`source/Pict-Meadow-Connection-Manager.js`) exports four
things:

| Export | Kind | Purpose |
|--------|------|---------|
| `PictProviderConnectionManager` | provider class | State + CRUD + schema injection + test dispatch |
| `PictViewConnectionList` | view class | List of saved connections |
| `PictViewConnectionDetail` | view class | Single-connection editor + the form slot |
| `PictSectionConnectionForm` | re-export | The `pict-section-connection-form` module, for one-`require()` wiring |

Each class also carries a `.default_configuration` property.

## Quick start

Register the singleton provider, the two manager-shell views, and the shared
form view; then inject schemas once they arrive.

```javascript
const libMCM = require('pict-meadow-connection-manager');

// Provider - singleton so the list view, detail view, and form view
// all read the same state.
this.pict.addProviderSingleton('MeadowConnectionManager',
	libMCM.PictProviderConnectionManager.default_configuration,
	libMCM.PictProviderConnectionManager);

// Manager-shell views.
this.pict.addView('MCM-ConnectionList',
	libMCM.PictViewConnectionList.default_configuration,
	libMCM.PictViewConnectionList);
this.pict.addView('MCM-ConnectionDetail',
	libMCM.PictViewConnectionDetail.default_configuration,
	libMCM.PictViewConnectionDetail);

// Shared schema-driven form - renders into the slot the detail view exposes.
this.pict.addView('PictSection-ConnectionForm',
	Object.assign({}, libMCM.PictSectionConnectionForm.default_configuration,
		{
			ContainerSelector:         '#MCM-ConnectionConfig-Container',
			DefaultDestinationAddress: '#MCM-ConnectionConfig-Container',
			SchemasAddress:            'AppData.MCM.Schemas',
			ActiveAddress:             'AppData.MCM.CurrentConnection.Type',
			FieldIDPrefix:             'mcm-conn',
			ShowProviderSelect:        false   // detail view owns the type <select>
		}), libMCM.PictSectionConnectionForm);
```

The host must provide two destination elements somewhere in its layout:
`#MCM-ConnectionList-Container` and `#MCM-ConnectionDetail-Container`. The
detail view itself renders the third slot, `#MCM-ConnectionConfig-Container`,
where the form view mounts.

Once schemas are available (typically from a server endpoint backed by
`meadow-connection-manager.getAllProviderFormSchemas()`), hand them to the
provider:

```javascript
fetch('/myapp/connection/schemas').then((pResponse) => pResponse.json()).then((pPayload) =>
{
	this.pict.providers.MeadowConnectionManager.setSchemas(pPayload.Schemas || []);
	this.pict.views['MCM-ConnectionList'].render();
});
```

From there, the list/detail/form views drive the entire add / edit / list /
test workflow.

## Documentation

- [Overview](docs/README.md)
- [Quickstart](docs/quickstart.md)
- [Architecture](docs/architecture.md)

## Related Modules

- [pict](https://fable-retold.github.io/pict/) - the MVC framework this module is built on.
- [pict-provider](https://fable-retold.github.io/pict-provider/) - base class for `PictProviderConnectionManager`.
- [pict-view](https://fable-retold.github.io/pict-view/) - base class for the list and detail views.
- [pict-section-connection-form](https://fable-retold.github.io/pict-section-connection-form/) - the schema-driven form view this module delegates field rendering to.
- [meadow-connection-manager](https://fable-retold.github.io/meadow-connection-manager/) - the server-side counterpart that aggregates the provider form schemas.

## Running the tests

```bash
npm install
npm test
```

The test suite (`test/PictMeadowConnectionManager_tests.js`) uses an inline
jsdom shim to mount a real Pict instance plus a synthetic DOM, so the
provider state/CRUD paths and the full list + detail + shared-form render
paths are exercised end-to-end without a browser.

## Running the example

A reference application lives under
`example_applications/bookstore-connections`. See the
[Bookstore Connections walkthrough](docs/examples/bookstore-connections/README.md).

```bash
npm run example
```

## License

MIT - Steven Velozo
