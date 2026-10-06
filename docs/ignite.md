# @ignite/web integration

The site renders Xcel Energy's **@ignite/web** design-system components
(Lit web components) inside Edge Delivery Services blocks. This page covers how
the integration is wired and how to build a block on it.

## What's in the repo

| Path | Purpose |
|---|---|
| `scripts/ignite/bundle/` | The prebuilt Ignite components: `primitives/` (button, hyperlink, icon button, …), `compositions/` (footer, banner, hero, …), `themes/tokens.css` (design tokens). Generated — don't edit by hand. |
| `scripts/components/ignite.js` | Loads an Ignite bundle on demand (see below). |
| `scripts/components/xe-icon.js` + `icons.js` | The local `<xe-icon>` and the site's icon set, used by every Ignite component. |
| `scripts/components/fontawesome-shim.js` | Stand-in for the Font Awesome Pro packages the bundles import, mapped by the import map in `head.html`. |
| `scripts/components/primitives.js` | Shared helpers for **primitive blocks** (blocks that render one Ignite primitive). |
| `blocks/xe-button/` | The first primitive block: the Ignite `<xe-button>`. |

## How loading works

Each bundle under `scripts/ignite/bundle` is self-contained: it inlines Lit and
the primitives it uses, and registers them with `customElements.define()`.
Loading two bundles would normally throw on the second registration of a shared
element (`xe-icon`, `xe-hyperlink`, …). `scripts/components/ignite.js` handles
that before any bundle loads:

- it registers the local `<xe-icon>` first, so every icon — including those
  Ignite renders inside its shadow roots — comes from the site's icon set;
- it makes `customElements.define()` skip names already registered (first
  registration wins) instead of throwing;
- it loads the design tokens (`themes/tokens.css`: the `--xe-*` custom
  properties, without the theme's global element resets).

```js
import loadIgnite from '../../scripts/components/ignite.js';

loadIgnite(() => import('../../scripts/ignite/bundle/compositions/footer/xe-footer.js'));
```

`loadIgnite()` resolves `null` if the bundle fails to load, so a block keeps
its fallback content rather than breaking the page.

The bundles keep bare imports such as `@fortawesome/pro-regular-svg-icons`; the
import map in `head.html` resolves them to `fontawesome-shim.js`. It carries the
`aem` nonce the Content Security Policy requires and must stay before the
module scripts.

## Primitive blocks

A primitive block renders a single Ignite primitive and exports a builder, so
the same component — with the same props — is used whether an author adds the
block or another block builds it:

```js
import { buildButton, loadButton } from '../xe-button/xe-button.js';

const button = buildButton({
  label: 'Explore Programs',
  href: '/programs',
  variant: 'primary',
  treatment: 'outlined',
  size: 'sm',
  trailingIcon: 'faArrowRight',
});
button.setAttribute('slot', 'action');
loadButton(); // loads and registers the Ignite button, once per page
```

Each primitive declares its props once, as a schema, and the helpers in
`scripts/components/primitives.js` do the rest:

- `applyProps(element, props, schema)` / `createPrimitive(tag, props, schema)`
  — set props as attributes, validated against the schema (unknown props and
  values are ignored, so the component's default applies);
- `propsFromClasses(block, schema)` — read authored options from the block's
  classes (the model's `classes_*` select fields);
- `readRows(block, names)` / `readLinkContent(block)` — a model's rows by
  field name, and the link + accessible label rows link primitives share;
- `renderPrimitive(block, element, { source, placeholder })` — render the
  primitive (moving the Universal Editor instrumentation onto it), or an
  editor-only placeholder while the block is empty;
- `ICON_OPTIONS` — option classes for every icon in `icons.js`
  (`faArrowRight` → `arrow-right`).

### XE Button

`BUTTON_PROPS` (`blocks/xe-button/xe-button.js`) covers every prop of the
Ignite button:

| Prop | Values | Authorable as |
|---|---|---|
| `variant` | primary (default), secondary, tertiary, accent, neutral, static-dark, static-light | Variant |
| `treatment` | filled (default), outlined (`outline` accepted), text | Treatment |
| `size` / `sizeMobile` | xxs, xs, sm, md (default), lg — mobile below 768px | Size / Mobile Size |
| `href` / `target` | a URL / `_blank`, `_self`, `_parent`, `_top` | Link URL / Open In |
| `leadingIcon` / `trailingIcon` | an icon name from `icons.js` | Leading Icon / Trailing Icon |
| `expand` / `disabled` | boolean | Options › Full Width / Disabled |
| `ariaLabel` | text | Accessible Label |
| `type`, `ariaHaspopup`, `ariaExpanded` | for buttons code controls (forms, menus) | code only |

Use `tertiary` (gold) and `static-light` only on dark backgrounds; on light
ones their text fails WCAG AA contrast, and filled `tertiary` fails on any.

To add another primitive (for example `xe-hyperlink`), follow `xe-button`:
declare its props schema, export `build…()` and `load…()`, and decorate the
block with `readLinkContent()`, `propsFromClasses()` and `renderPrimitive()`.

## Updating the Ignite bundle

`@ignite/web` is published to Xcel's private registry (see `.npmrc`), so it
isn't a regular dependency — `npm install` and CI work without access to it.
With registry access:

```sh
npm install --no-save @ignite/web@<version>
npm run build:ignite
```

`build:ignite` copies the package's `dist`, bundles every component with
esbuild (keeping the Font Awesome imports external), and derives
`themes/tokens.css` from the theme with `tools/ignite/extract-tokens.mjs`.
Commit the regenerated `scripts/ignite/bundle`.
