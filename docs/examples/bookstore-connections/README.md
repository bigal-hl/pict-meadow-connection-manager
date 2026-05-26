# Bookstore Connections — Named Meadow Connection Manager

<!-- docuserve:example-launch:start -->
> **[&#9654; Launch the live app](examples/bookstore-connections/index.html)** — runs in your browser, opens in a new tab.
<!-- docuserve:example-launch:end -->

A reference application that wires the full `pict-meadow-connection-manager`
stack into a themed shell: a left-hand list of saved named connections,
a center detail editor with Save / Test / Delete controls, and a
schema-driven per-provider field block supplied by
`pict-section-connection-form`. Three production-like fixture
connections — Bookstore MySQL, Bookstore PostgreSQL, and an in-memory
SQLite — preload from app settings, so the demo runs as a single static
bundle with no server.

This is the canonical example for adopting the manager. The application
file is *only* wiring: register the provider + the two manager-shell
views, register the shared connection-form view at a specific DOM slot
+ id namespace, register the theme provider with custom brand + slot
views, and inject a schema fixture. Everything else — the list, the
editor, the form, the Test button, the JSON config payload that goes
to the server — comes from the libraries.

## What it demonstrates

| Capability | Where you see it |
|------------|------------------|
| Pict provider for named-connection CRUD | `addProviderSingleton('MeadowConnectionManager', …)` — owns the list, the selected index, the saved blob |
| Manager-shell list view (left panel) | `addView('MCM-ConnectionList', …)` — clickable rows, Add, status badges |
| Manager-shell detail editor (center) | `addView('MCM-ConnectionDetail', …)` — Name + Type + Status fields, Save / Test / Cancel |
| Schema-driven form composition | `addView('PictSection-ConnectionForm', { ShowProviderSelect: false, … })` — host owns the type `<select>`, form owns the field block |
| Demo schema fixture instead of server fetch | `DEMO_SCHEMAS` hardcoded array → `provider.setSchemas(...)` |
| Settings-seeded saved connections | `pict_configuration.MeadowConnections` → `provider.onInitialize()` hydrates `AppData.MCM.Connections` |
| Status-aware TopBar slot view | `Bookstore-TopBar-Nav` reads `AppData.MCM.CurrentConnection.Name` and re-renders on every refresh |
| Refresh interception for cross-view sync | Wraps `provider.refreshViews` / `refreshDetailView` to also re-render the TopBar |
| Modal-shell three-panel layout | `tmpModal.shell()` + `addPanel({ Side: 'top' / 'left' / 'right' })` + `center()` — pure library |
| Settings panel as a hidden overlay | Right-side panel with `Hidden: true, Collapsed: true, Position: 'overlay'`; opens via gear button |
| Theme provider mounted into a slot panel | `Theme-Section`'s `Picker` / `ModeToggle` / `ScaleSelect` re-mount into the settings panel on every render |
| Test endpoint configurable per host | Provider option `TestConnectionEndpoint: '/test-connection'` is the entire wiring |
| Persisted shell layout | `PersistenceKey: 'bookstore-connections-shell'` — drag-resize the sidebar, reload, the width sticks |

## Key files

- `source/Pict-Application-Connections.js` — the entire application
  wiring. Reads top-to-bottom: modal section, manager provider, manager
  views, shared form view, host layout view, topbar slot views,
  settings panel view, theme provider. `onAfterInitializeAsync` renders
  the shell, monkey-patches the refresh paths to also redraw the
  topbar, injects the demo schemas, and renders the connection list.
- `source/Pict-Application-Connections-Configuration.json` — the
  `MeadowConnections` settings array that seeds the saved-connection
  list, plus the standard Pict application stanza.
- `source/views/PictView-Bookstore-Layout.js` — the layout view. One
  call to `tmpModal.shell()`, four `addPanel(...)` calls (topbar,
  sidebar, settings overlay, center), `PersistenceKey` scoped to this
  app.
- `source/views/PictView-Bookstore-TopBar-Nav.js` — the nav slot the
  theme topbar mounts. Reads `AppData.MCM.CurrentConnection.Name` and
  shows "Connections · <current name>".
- `source/views/PictView-Bookstore-TopBar-User.js` — the user slot.
  Houses the settings gear that toggles the right-side panel.
- `source/views/PictView-Bookstore-SettingsPanel.js` — the right-side
  overlay panel. Re-mounts the theme controls into a sub-div on every
  render.
- `source/BookstoreConnections-Brand.js` — reads `package.json`'s
  `retold.brand` block (name, palette, inline SVG icon, favicons) so
  the brand stays a single source of truth.

## The data model

The provider hangs everything off **one** AppData key, `MCM` (configurable
via the provider's `AppDataAddress` option):

- `AppData.MCM.Connections` — the saved-connection list, each
  `{ Name, Type, Config, Status }`. Hydrated at `onInitialize` from
  `settings.MeadowConnections` (which Pict pulls from the configuration
  JSON's `pict_configuration.MeadowConnections`).
- `AppData.MCM.SelectedIndex` — index into `Connections`, or `-1` when
  the user has clicked **Add** and the editor is on an unsaved draft.
- `AppData.MCM.CurrentConnection` — the active draft `{ Name, Type, Config, Status }`.
  The detail editor and the form view both bind to this address.
- `AppData.MCM.ConnectionTypes` — array of `{ Type, DisplayName }`
  built from the injected schemas; powers the type `<select>` in the
  detail editor.
- `AppData.MCM.Schemas` — full schema list (one per provider). The
  shared form view reads its field definitions from here.

The provider exposes a thin CRUD surface: `addConnection`,
`selectConnection`, `saveConnection`, `removeConnection`, plus
`testConnection` which POSTs `{ Type, Config }` to the configured
`TestConnectionEndpoint` and updates `Status`.

---

## Feature 1 — Registering the manager

The provider is a **singleton** because the list-view, the detail-view,
and the connection-form view all need to talk to the same state. A
fresh instance per view would mean three independent drafts.

```js
let tmpProviderConfig = Object.assign({},
    libMCM.PictProviderConnectionManager.default_configuration,
    {
        TestConnectionEndpoint: '/test-connection'
    });
this.pict.addProviderSingleton('MeadowConnectionManager',
    tmpProviderConfig,
    libMCM.PictProviderConnectionManager);
```

`TestConnectionEndpoint` is the entire production-wiring contract: any
endpoint that accepts a POST of `{ Type, Config }` and answers
`{ ok: true }` / `{ ok: false, message: '…' }` will light up the Test
button. The demo points it at `/test-connection` so the request shows
up in network panels — in a real deployment this is the
`meadow-connection-manager`-backed Orator route in your server.

The provider also defaults `AppDataAddress: 'MCM'`. Override at
registration time if you need two managers on the same page (one for
"source" connections, one for "target" connections, etc.).

---

## Feature 2 — Mounting the manager-shell views

The list and detail views ship inside `pict-meadow-connection-manager`
and self-register their CSS:

```js
this.pict.addView('MCM-ConnectionList',
    libMCM.PictViewConnectionList.default_configuration,
    libMCM.PictViewConnectionList);
this.pict.addView('MCM-ConnectionDetail',
    libMCM.PictViewConnectionDetail.default_configuration,
    libMCM.PictViewConnectionDetail);
```

Their default destinations are `#MCM-ConnectionList-Container` and
`#MCM-ConnectionDetail-Container` — the layout view exposes both as
panel `ContentDestinationId`s, which is the entire integration story.
Drop the manager into any layout by naming two slots with those ids;
the views handle their own list rendering, selection highlighting,
**+ Add** workflow, save / cancel / delete / test buttons, and theme
token wiring.

The detail view *does not* own per-provider field rendering. It carries
the Name input, the Type `<select>`, and the Status badge, plus a slot
at `#MCM-ConnectionConfig-Container` that the shared form view will
take over.

---

## Feature 3 — Composing the schema-driven form

`pict-section-connection-form` is re-exported by
`pict-meadow-connection-manager` as
`libMCM.PictSectionConnectionForm`, so one `require()` brings in the
whole stack. The registration overrides four host knobs:

```js
this.pict.addView('PictSection-ConnectionForm',
    Object.assign({}, libMCM.PictSectionConnectionForm.default_configuration,
        {
            ContainerSelector:         '#MCM-ConnectionConfig-Container',
            DefaultDestinationAddress: '#MCM-ConnectionConfig-Container',
            SchemasAddress:            'AppData.MCM.Schemas',
            ActiveAddress:             'AppData.MCM.CurrentConnection.Type',
            FieldIDPrefix:             'mcm-conn',
            ShowProviderSelect:        false       // detail view owns the type <select>
        }), libMCM.PictSectionConnectionForm);
```

- **`ContainerSelector`** — points at the slot the detail view exposes.
- **`SchemasAddress`** — the form mirrors schemas from `AppData.MCM.Schemas`,
  the same address the provider writes to in `setSchemas()`.
- **`ActiveAddress`** — the form reads the active provider from
  `AppData.MCM.CurrentConnection.Type`, which is the field the detail
  view's type `<select>` writes to. That's the wiring that makes the
  form swap provider blocks the moment the user picks a different type.
- **`FieldIDPrefix: 'mcm-conn'`** — every input's DOM id becomes
  `mcm-conn-<provider>-<field>`. The prefix exists so multiple
  connection forms can coexist on a single page without colliding.
- **`ShowProviderSelect: false`** — the detail view already renders the
  Type select; the form hides its own to avoid two selects side by
  side.

The form view is **pure presentation** — it does not fetch schemas
itself. The manager provider's `setSchemas()` is what populates them.

---

## Feature 4 — Schema fixture in place of a server fetch

In production, the schema list is the JSON response of
`GET /<app>/connection/schemas`, which is backed by
`meadow-connection-manager.getAllProviderFormSchemas()`. The demo
short-circuits that with a hardcoded fixture so the bundle runs as a
single static page:

```js
const DEMO_SCHEMAS =
[
    {
        Provider:    'SQLite',
        DisplayName: 'SQLite',
        Fields:
        [
            { Name: 'SQLiteFilePath', Label: 'Database File Path', Type: 'Path', Default: './data/database.db', Required: true, Placeholder: '/path/to/database.db' }
        ]
    },
    {
        Provider:    'MySQL',
        DisplayName: 'MySQL',
        Fields:
        [
            { Name: 'host',            Label: 'Server',           Type: 'String',   Default: '127.0.0.1', Required: true },
            { Name: 'port',            Label: 'Port',             Type: 'Number',   Default: 3306,        Required: true },
            { Name: 'user',            Label: 'User',             Type: 'String',   Default: 'root',      Required: true },
            { Name: 'password',        Label: 'Password',         Type: 'Password' },
            { Name: 'database',        Label: 'Database',         Type: 'String',   Default: 'meadow',    Required: true },
            { Name: 'connectionLimit', Label: 'Connection Limit', Type: 'Number',   Default: 20,          Group: 'Advanced' }
        ]
    },
    {
        Provider:    'PostgreSQL',
        DisplayName: 'PostgreSQL',
        Fields: [ /* host/port/user/password/database */ ]
    }
];
```

Then in `onAfterInitializeAsync`:

```js
this.pict.providers.MeadowConnectionManager.setSchemas(DEMO_SCHEMAS);
```

`setSchemas()` writes `AppData.MCM.Schemas` (the form's
`SchemasAddress`) and `AppData.MCM.ConnectionTypes` (the detail
editor's `<select>` source), then forces a refresh. **The same code
runs unchanged in production** — production just sources `DEMO_SCHEMAS`
from a `fetch()` first.

The fixture intentionally uses the same shape the
`Meadow-Connection-<Type>-FormSchema.js` files export
(`Name`/`Label`/`Type`/`Default`/`Required`/`Placeholder`/`Group`/…),
so the demo exercises the production schema contract verbatim.

---

## Feature 5 — Seeded saved connections via settings

The configuration JSON's `pict_configuration.MeadowConnections` array
seeds the saved-connection list. Pict surfaces this as
`fable.settings.MeadowConnections`, which the provider reads at
`onInitialize`:

```json
"pict_configuration":
{
    "Product": "MCM-Example",
    "MeadowConnections":
    [
        {
            "Name": "Bookstore MySQL",
            "Type": "MySQL",
            "Config": { "server": "127.0.0.1", "port": 23306, "user": "root", "password": "1234567890", "database": "bookstore", "connectionLimit": 5 },
            "Status": "unknown"
        },
        {
            "Name": "Bookstore PostgreSQL",
            "Type": "PostgreSQL",
            "Config": { "server": "127.0.0.1", "port": 25432, "user": "postgres", "password": "testpassword", "database": "testdb" },
            "Status": "unknown"
        },
        {
            "Name": "In-Memory SQLite",
            "Type": "SQLite",
            "Config": { "SQLiteFilePath": ":memory:" },
            "Status": "unknown"
        }
    ]
}
```

The shape matches `AppData.MCM.Connections` exactly. In a real app the
same array could be loaded from `GET /<app>/connections` and handed to
`provider.setConnections()`; the settings fixture just makes the demo
self-contained.

The MySQL and PostgreSQL ports (23306, 25432) align with the bundled
`docker-compose.yml` so `npm run docker-up` brings up *actual* MySQL +
PostgreSQL servers and the Test button hits them for real.

---

## Feature 6 — Cross-view sync via refresh interception

The TopBar's nav slot shows the active connection name — but the
manager-shell views don't know the topbar exists. Rather than couple
them, the application monkey-patches the provider's refresh methods at
boot:

```js
let tmpProvider = this.pict.providers.MeadowConnectionManager;
let tmpLayout   = this.pict.views['Bookstore-Layout'];
if (tmpProvider && tmpLayout && typeof tmpLayout.renderTopBar === 'function')
{
    let tmpOrigRefreshViews  = tmpProvider.refreshViews.bind(tmpProvider);
    let tmpOrigRefreshDetail = tmpProvider.refreshDetailView.bind(tmpProvider);
    tmpProvider.refreshViews = function ()
    {
        tmpOrigRefreshViews();
        tmpLayout.renderTopBar();
    };
    tmpProvider.refreshDetailView = function ()
    {
        tmpOrigRefreshDetail();
        tmpLayout.renderTopBar();
    };
}
```

`refreshDetailView` fires on selection change (which is what updates
`CurrentConnection.Name`); `refreshViews` covers add / remove / save /
schema injection. Hooking both ensures the topbar stays in sync no
matter which path triggered the change.

This is the manager's extension point: **the provider drives the
refresh; the host decides what else to refresh alongside it.**

---

## Feature 7 — Three-panel modal shell layout

The layout view delegates the entire chrome to `pict-section-modal`'s
shell API. One `tmpModal.shell()` call, then `addPanel({...})` per
edge, then `center({...})` for the main workspace:

```js
this._shell = tmpModal.shell(tmpMount, { PersistenceKey: 'bookstore-connections-shell' });

// Top — theme chrome (BrandMark + Nav + User slots).
this._shell.addPanel(
{
    Hash: 'topbar', Side: 'top', Mode: 'fixed', Size: 48,
    ContentDestinationId: 'Theme-TopBar', ContentView: 'Theme-TopBar'
});

// Left — connection list sidebar.
this._shell.addPanel(
{
    Hash: 'sidebar', Side: 'left', Mode: 'resizable',
    Size: 320, MinSize: 220, MaxSize: 520, Title: 'Connections',
    ContentDestinationId: 'MCM-ConnectionList-Container',
    ResponsiveDrawer: 900
});

// Right (overlay, hidden) — theme settings panel.
this._shell.addPanel(
{
    Hash: 'settings', Side: 'right', Mode: 'resizable', Position: 'overlay',
    Size: 360, MinSize: 280, MaxSize: 540,
    Hidden: true, Collapsed: true,
    ContentDestinationId: 'Bookstore-Settings-Panel',
    ContentView: 'Bookstore-SettingsPanel'
});

// Center — connection detail editor.
this._shell.center({ ContentDestinationId: 'MCM-ConnectionDetail-Container' });
```

Three things to notice:

1. **`PersistenceKey: 'bookstore-connections-shell'`** scopes the
   panel-size memory in `localStorage` to this app. Drag the sidebar's
   inner edge, reload — the sidebar reopens at the dragged width.
2. **`ResponsiveDrawer: 900`** on the sidebar flips it into a top
   drawer below 900px viewport width. Zero CSS in the application; the
   library handles the breakpoint.
3. **The settings panel is `Position: 'overlay'` + `Hidden: true`** —
   it floats above the workspace rather than displacing it, and starts
   invisible. The gear button in the topbar's user slot calls
   `layout.toggleSettingsPanel()` to open it.

---

## Feature 8 — Theme provider mounted last

The theme provider is the **last** thing registered, after every view
the topbar might reference. `Theme-Section`'s bootstrap looks up
`NavView` and `UserView` by hash and won't find them if they were
registered after it. Order matters:

```js
// Slot views first
this.pict.addView('Bookstore-TopBar-Nav', libViewTopBarNav.default_configuration, libViewTopBarNav);
this.pict.addView('Bookstore-TopBar-User', libViewTopBarUser.default_configuration, libViewTopBarUser);

// Theme provider last — picks up the slot views by hash
this.pict.addProvider('Theme-Section',
    {
        ApplyDefault: 'pict-default',
        DefaultMode:  'system',
        DefaultScale: 1.0,
        Brand:        libBrand,
        Views: ['Picker', 'ModeToggle', 'ScaleSelect', 'BrandMark', 'TopBar'],
        ViewOptions:
        {
            TopBar:
            {
                NavView:  'Bookstore-TopBar-Nav',
                UserView: 'Bookstore-TopBar-User',
                Height:   48
            }
        }
    },
    libPictSectionTheme);
```

`Views: [...]` is the theme provider's "what would you like me to
provide" array. We skip `Button` (the manager already supplies its
own) and `BottomBar` (no status bar here), but keep `Picker` /
`ModeToggle` / `ScaleSelect` for the settings panel and `BrandMark` /
`TopBar` for the chrome.

The `Brand` block comes from `BookstoreConnections-Brand.js`, which is
a one-liner that reads `package.json#retold.brand`. The brand block is
the single source of truth — `npm run brand` regenerates it
(name, palette, inline SVG, favicons) from the manifest.

---

## Feature 9 — Settings overlay with re-mounted theme controls

The settings panel's body template carries one empty div:

```html
<div class="bookstore-settings-section">
    <div class="bookstore-settings-label">Appearance</div>
    <div id="Bookstore-Settings-Theme"></div>
</div>
```

On every render, the panel view tells the theme provider to mount its
controls into that div:

```js
onAfterRender(pRenderable, pAddress, pRecord, pContent)
{
    if (this.pict && this.pict.CSSMap) { this.pict.CSSMap.injectCSS(); }

    let tmpTheme = this.pict.providers && this.pict.providers['Theme-Section'];
    if (tmpTheme && typeof tmpTheme.mount === 'function')
    {
        tmpTheme.mount(
        {
            Container: '#Bookstore-Settings-Theme',
            Views: ['Picker', 'ModeToggle', 'ScaleSelect']
        });
    }

    return super.onAfterRender(pRenderable, pAddress, pRecord, pContent);
}
```

Re-mounting on every render is intentional — the shell renders the
panel's destination div fresh on every expand, which wipes any
previously-mounted children. Idempotent mounting via `tmpTheme.mount()`
is how every theme-provided view stays alive across panel
collapse/expand cycles.

---

## Running the example

The example ships with a Docker compose stack that runs *real* MySQL
and PostgreSQL servers on the ports the fixture connections point at:

```bash
cd example_applications/bookstore-connections
npm install
npm run brand        # regenerates retold.brand from the manifest (only needed if changed)
npm run build        # quack build → dist/

# Option A: serve the static bundle (no real DBs, Test button will fail)
# Open dist/index.html in a browser

# Option B: full stack with docker
npm run docker-up    # brings up MySQL + PostgreSQL on the demo ports
npm start            # node server.js — Orator wrapping the dist folder + /test-connection
# Visit http://localhost:8080
```

`server.js` is a thin Orator server that serves `dist/` statically and
exposes `POST /test-connection` backed by
`meadow-connection-manager.testConnection()`. The settings JSON has
its ports aligned with `docker-compose.yml` so the Test button works
end-to-end against the live servers.

## Things to try in the running app

- **Click Bookstore MySQL** in the sidebar — the editor populates, the
  topbar title becomes "Connections · Bookstore MySQL", and the form
  block swaps to the MySQL field set.
- **Click + Add** — the editor switches to a new draft. Pick PostgreSQL
  from the Type select and the field block instantly swaps.
- **Click Test** with the Docker stack running — the request goes to
  `POST /test-connection`, the status badge updates to `connected` /
  `error`, and the topbar title re-renders.
- **Drag the sidebar's inner edge** — width persists via
  `PersistenceKey: 'bookstore-connections-shell'`.
- **Click the gear** in the topbar (right side) — the right-side
  overlay slides in with Picker / ModeToggle / ScaleSelect.
- **Resize below 900px** — the sidebar flips into a top drawer.
- **Edit a connection's Config and click Save** — the entry in the
  sidebar updates, the topbar reflects the new name, and the saved blob
  matches the canonical wire format the server expects.

## Takeaways

1. **The manager is two views and a provider.** Wire them at named DOM
   slots, hand them a schema list, and the entire add / edit / list /
   test workflow is yours — no per-provider subclasses, no bespoke
   form code.
2. **Schemas are the contract.** `setSchemas()` is the boundary between
   "what providers exist" (server-driven) and "how to render their
   forms" (pure-presentation view). The demo fixture has the same shape
   as the production endpoint response.
3. **Composition lives at registration time.** `ShowProviderSelect: false`
   + `ContainerSelector: '#MCM-ConnectionConfig-Container'` is how the
   detail view and the form view negotiate which one owns the type
   select.
4. **Refresh hooks are the host's extension point.** The provider
   centralises the refresh; wrap its methods to re-render anything else
   that needs to stay in sync (topbar, status indicators, custom side
   panels).
5. **The chrome comes from libraries.** The shell, the panels, the
   responsive drawer, the topbar, the theme picker, the persistence —
   all `pict-section-modal` + `pict-section-theme`. The application
   adds the routes and the slot views; nothing else.

## Related documentation

- [pict-meadow-connection-manager — module overview](https://fable-retold.github.io/pict-meadow-connection-manager/)
- [pict-section-connection-form — schema-driven form reference](https://fable-retold.github.io/pict-section-connection-form/)
- [pict-section-modal — shell + panels reference](https://fable-retold.github.io/pict-section-modal/)
- [pict-section-theme — brand + theme tokens reference](https://fable-retold.github.io/pict-section-theme/)
- [meadow-connection-manager — server-side schema aggregation](https://fable-retold.github.io/meadow-connection-manager/)
