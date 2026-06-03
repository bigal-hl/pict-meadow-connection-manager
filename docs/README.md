# Pict Meadow Connection Manager

A browser-safe Pict module for managing **named** meadow database connections.
It supplies a list view, a detail editor, and a state/CRUD provider - the
*manager shell* - and delegates the per-provider field block to
[`pict-section-connection-form`](https://fable-retold.github.io/pict-section-connection-form/).

This split matters: the manager owns everything that is the *same* across
every database engine (a saved-connection list, a name field, a type picker, a
status badge, Save / Test / Cancel buttons, and the CRUD state behind them),
while the form view owns everything that *differs* per engine (the host, port,
user, password, file-path, advanced-options fields). The per-engine field
definitions are not duplicated here - they come from the `meadow-connection-*`
modules and are aggregated server-side by
[`meadow-connection-manager`](https://fable-retold.github.io/meadow-connection-manager/).

## The three pieces

| Piece | Class | Responsibility |
|-------|-------|----------------|
| Provider | `PictProviderConnectionManager` | The saved-connection list, the selected/draft connection, the injected schema list, and connection-test dispatch. All state lives under one configurable `AppData` address (default `MCM`). |
| List view | `PictViewConnectionList` | Renders the saved connections as rows with Edit / Remove buttons, plus an Add Connection button. |
| Detail view | `PictViewConnectionDetail` | Edits one connection: a Name input, a Type `<select>`, a Status badge, Save / Test / Cancel buttons, and a slot the form view fills. |

A fourth export, `PictSectionConnectionForm`, is a convenience re-export of the
`pict-section-connection-form` module so hosts can pull the whole stack in with
a single `require()`.

## How it fits together

<!-- bespoke diagram: edit diagrams/how-it-fits-together.mmd or .hints.json, then: npx pict-renderer-graph build modules/pict/pict-meadow-connection-manager/docs -->
![How it fits together](diagrams/how-it-fits-together.svg)

The provider is the hub. Both views read and write its state through
`pict.AppData`, and the provider propagates the schema list into the form view
whenever the detail editor opens. The host application is responsible for
fetching the schema list and handing it to `setSchemas()` - this module never
talks to a server itself.

## Where to go next

- [Quickstart](quickstart.md) - register the provider and views, inject schemas, and wire the form slot.
- [Architecture](architecture.md) - the provider state model, the list/detail views, and the delegation to the form view.
- [Bookstore Connections example](examples/bookstore-connections/README.md) - the canonical reference application.

## Related Modules

- [pict](https://fable-retold.github.io/pict/) - the MVC framework this module is built on.
- [pict-provider](https://fable-retold.github.io/pict-provider/) - base class for `PictProviderConnectionManager`.
- [pict-view](https://fable-retold.github.io/pict-view/) - base class for the list and detail views.
- [pict-section-connection-form](https://fable-retold.github.io/pict-section-connection-form/) - the schema-driven form view this module delegates field rendering to.
- [meadow-connection-manager](https://fable-retold.github.io/meadow-connection-manager/) - the server-side counterpart that aggregates the provider form schemas.
