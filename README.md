# dprint-plugin-panache

A [dprint](https://dprint.dev) Wasm plugin that wraps the
[Panache](https://panache.bz) formatter for Quarto (`.qmd`), Pandoc, and
Markdown (`.md`, `.Rmd`).

It is released independently of the main Panache CLI. The plugin lives in its
own repository so that its `panache.wasm` release asset does not interfere with
the Panache CLI's GitHub release stream (the Zed extension resolves its download
via `latest_github_release(require_assets: true)`, which would otherwise pick up
this plugin's asset).

## Usage

Add the plugin to your `dprint.json`:

```jsonc
{
  "panache": {},
  "plugins": [
    "https://github.com/jolars/dprint-plugin-panache/releases/latest/download/panache.wasm"
  ]
}
```

Then format:

```bash
dprint fmt
```

## Configuration

Configure under the `panache` key in `dprint.json`. Supported keys:

| Key                  | Description                                              |
| -------------------- | -------------------------------------------------------- |
| `flavor`             | Pandoc, Quarto, RMarkdown, GFM, or CommonMark            |
| `wrap`               | `reflow`, `preserve`, `sentence`, or `semantic`          |
| `blankLines`         | Maximum consecutive blank lines                          |
| `mathIndent`         | Indentation for display math content                     |
| `mathDelimiterStyle` | Display-math delimiter style                             |
| `tabStops`           | Tab stop width                                           |
| `lineEnding`         | Line-ending style                                        |
| `lineWidth`          | Maximum line width (dprint global, default 80)           |
| `tabWidth`           | Indentation width (dprint global)                        |

## Building

The plugin compiles only for `wasm32-unknown-unknown` (it relies on
`dprint_core::generate_plugin_code!`):

```bash
cargo build --release --target wasm32-unknown-unknown
```

The resulting `target/wasm32-unknown-unknown/release/dprint_plugin_panache.wasm`
is published as `panache.wasm` on each GitHub release.

## License

MIT
