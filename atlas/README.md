# Atlas (Apex)

A modern, design-forward dashboard for Home Assistant.

Atlas replaces Lovelace for the two surfaces that benefit most from a
tighter, more opinionated UI: a 1280×800 wall-mounted tablet (the Hub)
and a phone in portrait (Mobile). Both render from a single locked design
system called Hearth — Cobalt brand, seven swappable accents, light and
dark schemes. The voice is quiet and declarative; no exclamation marks,
no emoji.

The add-on packages the Go backend that talks to Home Assistant over
WebSocket and REST, with the React SPA embedded so a single binary serves
everything. Supervisor's ingress framing means HA's auth terminates the
request before it reaches Atlas; the HA long-lived token lives server-side
and never crosses the network edge.

## After install

The add-on appears in HA's sidebar as **Atlas**. Click the sidebar entry,
or open the add-on's detail page and click **Open Web UI**, to load the
dashboard inside an HA panel. Admin-only visibility is controlled by the
`panel_admin` flag in the manifest; HA admins see the panel, other users
do not.

In add-on mode, Atlas auto-detects Supervisor and skips the onboarding
form — the first thing you see is the Hub home screen with your live HA
areas, floors, and entities already populated. State changes from HA
stream into the dashboard in real time over the WebSocket proxy.

## Documentation

See [`DOCS.md`](DOCS.md) for the full add-on documentation: installation
walk-through, first-run setup, configuration options (currently none),
troubleshooting, updating, uninstalling, and reporting issues.

## Source

The Atlas source code lives in the same Git repository as this add-on
metadata. See the repository root's `README.md` for development setup,
the Hearth design system, and the Codev-driven development flow.

## License

MIT. See the repository root's `LICENSE` file.
