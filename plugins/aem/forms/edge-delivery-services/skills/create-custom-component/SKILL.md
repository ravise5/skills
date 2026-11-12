---
name: create-custom-component
description: "Create a custom form component for an AEM Forms Edge Delivery Services (Form block) project — scaffold blocks/form/components/<name>, extend an OOTB field type (text-input, drop-down, panel, etc.), define authoring properties in _<name>.json, wire the decorate function to the form model with subscribe(), register it in mappings.js, merge the authoring JSON into the built component-*.json files, and review the output. Use when a user asks to create, scaffold, build, or fix a custom form component, form field widget, or viewType in an aem-boilerplate-forms project."
license: Apache-2.0
metadata:
  version: "1.0.0"
---

# Create Custom Form Component

Build a custom component for the AEM Forms **Form block** on Edge Delivery Services end to end — scaffold, authoring schema, `decorate` logic, model subscriptions, registration, built authoring JSON, documentation — and **review** it before reporting done.

## When to use

- The project is a forms project derived from `aem-boilerplate-forms` (it has `blocks/form/`, `blocks/form/mappings.js`, `blocks/form/rules/index.js`).
- The user wants a new field widget/variant (slider, rating, OTP input, searchable dropdown, card choice, consent panel…) or wants to fix one under `blocks/form/components/`.

Not for: generic Edge Delivery Services blocks (use the `building-blocks` skill of the `aem-edge-delivery-services` plugin), AEM Sites HTL components (`create-component` in `aem-cloud-service`), or rule-editor logic.

## References (read as needed)

| File | Read when |
|------|-----------|
| `references/custom-form-components.md` | Always — architecture, folder layout, JSON schema, registration |
| `references/form-field-types.md` | Step 4 — the base HTML the `element` parameter already contains |
| `references/authoring-field-types.md` | Step 3 — widgets for the properties dialog (`text`, `select`, `container`, …) |
| `references/subscribe-api.md` | Step 4 — `subscribe()` patterns: own value, child fields, custom events |
| `references/build-json.md` | Steps 5 and 7 — merge the component into `component-models.json`, `component-definition.json` and `component-filters.json` yourself (what `npm run build:json` does), and count `xwalk/max-cells` |

Paths under `references/` are relative to this skill's directory.

---

## Workflow

### Step 1: Gather requirements

Run every command in this skill from the **repository root** — the directory containing `package.json` and `blocks/form/`. Do not `cd` into a subfolder (such as `code/`) first. Then confirm — ask only for what the request does not already answer:

| Question | Why |
|----------|-----|
| Component name | Folder, file names, and `fd:viewType` (lowercase kebab-case, e.g. `star-rating`) |
| Which OOTB field type does it extend? | Base HTML and AEM resource type |
| Which authorable properties does it need? | Fields in `_<name>.json` |
| What model state does it read/write, and when should `model.value` be committed? | Drives `subscribe` wiring and the README contract |
| Is it a panel that watches child fields, or does it react to custom events? | Selects the subscribe pattern |
| Expected behaviour (what the user does, what the model should hold, how often rules fire) | What the review in Step 8 checks, and what the user verifies in Step 9 |

Supported base types: `text-input`, `number-input`, `drop-down`, `button`, `checkbox`, `radio-group`, `checkbox-group`, `panel`, `date-input`, `multiline-input`, `file-input`, `email`, `telephone-input`.

Check the name is not already used: `blocks/form/components/<name>/` must not exist and `<name>` must not be in `mappings.js`.

### Step 2: Scaffold

If `package.json` has a `create:custom-component` script, run it from the repository root (install dependencies first if `node_modules` is missing):

```bash
npm run create:custom-component -- --name <name> --base <base-type>
```

It creates `blocks/form/components/<name>/{<name>.js,<name>.css,_<name>.json}` and usually also registers the component (mappings, `_form.json` filter, `models/_component-definition.json`). **Do not assume it did. Step 8 checks it.**

If the script does not exist, create the three files by hand: copy the base type's `blocks/form/models/form-components/_<base>.json`, set `definitions[0].id` and `models[0].id` to `<name>`, set `template["fd:viewType"]` to `<name>`, and fix `"..."` include paths so they resolve from the component folder (`../../models/form-common/...`).

### Step 3: Define authoring properties (`_<name>.json`)

Read `references/authoring-field-types.md` to choose the widget for each property. Add custom fields to `models[0].fields`; reuse shared field groups instead of redefining them:

```json
{ "...": "../../models/form-common/_basic-validation-fields.json#/fields" }
```

Rules: every field has `component` and a unique `name`; `select`/`multiselect`/`radio-group`/`checkbox-group` have `options`; keep `fieldType`, `resourceType`, and `fd:viewType` from the scaffold.

### Step 4: Implement `decorate`

Read `references/form-field-types.md` (base HTML) and `references/subscribe-api.md`, then pick the pattern:

| The component… | Use |
|----------------|-----|
| Reacts to its own value / enum / visible changes | `subscribe(fieldDiv, formId, cb, { listenChanges: true })` — handle `'register'` and `'change'` |
| Watches child fields (panel) | Same on the parent; inside `'register'`, `subscribe()` each child wrapper found with `[data-id="${child.id}"]` |
| Reacts to custom events (`custom:<event>`) | `fieldModel.subscribe(cb, '<event>')` inside `'register'` — never `fieldDiv.addEventListener` |

Implementation rules:

- `export default function decorate(fieldDiv, fieldJson, container, formId)` and `return fieldDiv`.
- Import `subscribe` from `'../../rules/index.js'`; relative imports always include `.js`.
- Read authored properties from `fieldJson.properties.<prop>`.
- Enhance the existing OOTB markup in `fieldDiv`; scope DOM queries to it.
- When writing the value from custom UI, either set `model.value` or dispatch `new Event('change', { bubbles: true })` on the underlying input.
- Guard against loops: never set `model.value` in a value-change handler without comparing first.
- Know when the value is committed. Out of the box, the form block writes `model.value` only from the input's native `change` event (`blocks/form/rules/index.js`), which text-like inputs fire on blur/commit, not on each keystroke. So rules on the field's change event run once per edit, when the user leaves the field. If the component must commit on every keystroke (live counters, search-as-you-type, values derived while typing), handle the `input` event in the component and set `model.value` there, comparing with the current value first. When reformatting the display while typing (masks, grouping), don't dispatch `change` on each keystroke, or you turn the blur commit into a per-keystroke one.
- Prefix CSS classes with the component name in `<name>.css`.

### Step 5: Register

1. `blocks/form/mappings.js` — add `<name>` to `customComponents` (or to `OOTBComponentDecorators` for OOTB-style decorators). Must exactly equal `fd:viewType`.
2. If the component has no stylesheet, add it to `NO_CSS_COMPONENTS` when the project defines that set; otherwise keep `<name>.css`.
3. `blocks/form/_form.json` — make sure `<name>` is in the `"form"` filter `components` so authors can insert it.
4. `models/_component-definition.json` — make sure the custom-components group includes `../blocks/form/components/<name>/_<name>.json#/definitions` (unless the project includes components by glob).
5. Repository lint limits: check the project's ESLint config (`.eslintrc.cjs`, `eslint.config.*`, …) for per-model limits. The xwalk config usually sets `'xwalk/max-cells'`: it limits each model in the built `component-models.json` to 4 cells unless the model id has its own entry. A form component inherits its base type's fields, so it needs one. Count the cells of the resolved model (Step 7) as described in `references/build-json.md` §6, and add `'<name>': <cells>` next to the existing entries. Follow any other per-model lint options the repository defines the same way.

### Step 6: Document

Create `blocks/form/components/<name>/README.md` (use an existing component README as a model):

| Section | Required | Content |
|---------|----------|---------|
| `# <Component Name>` | Yes | One-line summary |
| `## How to Use` / `## Authoring Properties` | Yes | How to configure it; table of custom properties |
| `## Model Subscriptions` | If applicable | Model changes / custom events it listens to |
| `## Events Dispatched` | If applicable | Custom events it fires |
| `## Model Properties Set` | If applicable | What it writes and **when `model.value` is committed**: on blur/`change` (the OOTB default), per keystroke (component-handled `input`), or external only. Also how often the field's change rules fire |
| `## Usage Notes` | Yes | Base type, `fd:viewType`, key behaviour |

If the project keeps a component catalog (for example a `component-registry.md`), add a one-line row for the component too.

### Step 7: Build the authoring JSON (merge it yourself)

Do **not** run `npm run build:json`. Do what it does yourself, following `references/build-json.md`:

1. Resolve the component's model from `_<name>.json`: replace every `"..."` include with the fields it points to (paths are relative to the file that contains the include), until no `"..."` remains.
2. Insert or replace that model in `component-models.json`, at the position the build would use.
3. Insert or replace the component's definition in the right group of `component-definition.json` (usually `custom-components`).
4. Add `<name>` to the `form` filter in `component-filters.json`.
5. Count the `xwalk/max-cells` cells of the resolved model and update the ESLint config (Step 5.5).

Without this, new properties never reach the Universal Editor. Treat the boilerplate's `rating` component (`blocks/form/components/rating/_rating.json` and its entries in the three generated files) as the reference example. Redo this step whenever `_<name>.json` or a file it includes changes.

### Step 8: Review (mandatory)

Do not report the component as done until every check below passes. Fix any failure and check again.

**A. Files and naming**
- `blocks/form/components/<name>/` contains `<name>.js`, `_<name>.json`, `README.md`, and `<name>.css` (unless it's in `NO_CSS_COMPONENTS`).
- Folder name = JS file name = `definitions[0].id` = `models[0].id` = `fd:viewType` = `<name>`, all lowercase kebab-case.
- `_<name>.json` is valid JSON. `fieldType` and `resourceType` match the base type. Every `"..."` include resolves to an existing file and pointer. Field `name`s are unique after resolution.

**B. JavaScript**
- ES module with `export default function decorate(...)` that returns `fieldDiv`.
- `subscribe` is imported from `'../../rules/index.js'` and called with `{ listenChanges: true }`. Every relative import ends in `.js`.
- None of these legacy patterns: `fieldModel.subscribe(cb, 'change')`, `new Event('change')` without `{ bubbles: true }`, child fields selected by `#id` (use `[data-id="…"]`), custom events handled with `addEventListener`.
- Every value write is guarded against loops, and the commit timing matches the requirement (blur vs. per keystroke).

**C. Registration**: `<name>` in `customComponents` (or `OOTBComponentDecorators`) in `mappings.js`, in the `form` filter of `_form.json`, and in `models/_component-definition.json`, each exactly once.

**D. Build output**: the checks in `references/build-json.md` §5 pass. The three generated files are valid JSON, contain no `"..."`, have exactly one entry for `<name>`, and `git diff` shows no other changes.

**E. Lint**: if `package.json` has a `lint` script and `node_modules` is installed, run `npm run lint` and fix every issue in the new or changed files. This includes `xwalk/max-cells`, which lints the generated `component-models.json`. If lint can't run, say so. Don't claim it passed.

**F. README**: the sections required in Step 6 are present.

### Step 9: User verification and feedback

You can't confirm runtime behaviour from code review alone, so give the user concrete steps to verify it:

1. Tell them how to see the component. Either preview a form that uses it locally (for example, `aem up` and a form page with a field whose `:type`/`fd:viewType` is `<name>`), or add it in the Universal Editor after the change is pushed.
2. List exactly what to try: each interaction, each authored property, model ↔ UI sync (for example, a rule that sets the value), how often change rules should fire, and keyboard use.
3. Ask whether it looks and behaves as expected and what should change. Use the agent's question tool if it has one.
4. For each piece of feedback, change the component, redo Step 7 if the JSON changed, re-run Step 8, and ask again. Repeat until the user approves.

If the user can't be reached (non-interactive run), skip the questions and include the verification steps in the report.

### Step 10: Report

Finish with: the files created or changed (including the three generated JSON files and the ESLint config), the results of each Step 8 check, the lint result (or why it couldn't run), the `xwalk/max-cells` count, the feedback rounds and changes made (or the browser verification steps if the user wasn't reachable), and anything left open.

---

## Quick reference: file locations

| File | Purpose |
|------|---------|
| `blocks/form/components/<name>/<name>.js` | Component logic (`decorate`) |
| `blocks/form/components/<name>/<name>.css` | Component styles |
| `blocks/form/components/<name>/_<name>.json` | Definition + authoring model |
| `blocks/form/components/<name>/README.md` | Component documentation |
| `blocks/form/mappings.js` | Registration (`customComponents`, `OOTBComponentDecorators`) |
| `blocks/form/_form.json` | `"form"` filter — which components authors can insert |
| `models/_component-definition.json` | Component palette (custom-components group) |
| `component-models.json`, `component-definition.json`, `component-filters.json` | Generated (built) authoring JSON that the Universal Editor reads. Updated by hand in Step 7 |
| `blocks/form/rules/index.js` | `subscribe` source |

## Common mistakes

| Mistake | Fix |
|---------|-----|
| Folder name ≠ JS file name ≠ `fd:viewType` | All three must be identical |
| Not in `customComponents` | JS/CSS never load (Step 8 C) |
| `subscribe` without `{ listenChanges: true }` or legacy `fieldModel.subscribe(cb, 'change')` | Use the standard pattern in `references/subscribe-api.md` |
| Child field selected with `#id` | Use `[data-id="${child.id}"]` (the wrapper) |
| Custom event handled via `addEventListener` | `fieldModel.subscribe(cb, '<event>')` in `'register'` |
| `new Event('change')` without `{ bubbles: true }` | Change never reaches the form model |
| Setting `model.value` inside its own change handler | Compare before assigning |
| Assuming the model updates while the user types | OOTB commits on blur (`change`); handle `input` in the component if rules need each keystroke, and document it in `## Model Properties Set` |
| Dispatching `change` on every keystroke while formatting | Turns the blur commit into a per-keystroke one; let the native `change` commit, or commit deliberately |
| Generated JSON not updated (Step 7) | Properties missing in the Universal Editor. Merge the model, definition and filter into the three root `component-*.json` files |
| `"..."` left in the generated JSON, or include path resolved from the repo root | Resolve each include relative to the file that contains it, recursively, until no `"..."` remains |
| Hand-patching single fields in `component-models.json` | Re-resolve and replace the whole model entry so it matches the source |
| No `xwalk/max-cells` entry for the new model | `npm run lint` fails with "Avoid using more than 4 cells"; add `'<name>': <cells>` to the ESLint config, counted as in `references/build-json.md` §6 |
| Reporting done without running Step 8 | Always run every Step 8 check, and report any check that could not run |
| Trusting `data-component-status="loaded"` alone | It is set even when `decorate()` throws (`mappings.js` swallows the error). Ask the user to check the browser console too |

## Example

**User:** "Create a star rating component based on number-input with a configurable maximum number of stars."

1. Scaffold `npm run create:custom-component -- --name star-rating --base number-input`.
2. `_star-rating.json`: add `{ "component": "number", "name": "maxStars", "label": "Maximum stars", "valueType": "number" }`.
3. `star-rating.js`: hide the number input, render `maxStars` buttons, set `model.value` on click, sync the selected stars on `'change'` of `value`, using `subscribe(..., { listenChanges: true })`.
4. Register in `mappings.js` / `_form.json` / `models/_component-definition.json`, and write `README.md`.
5. Build by hand: resolve `_star-rating.json`'s includes into a flat `star-rating` model in `component-models.json` (next to `rating`), add its definition to the `custom-components` group in `component-definition.json`, and add `star-rating` to the `form` filter in `component-filters.json`. Count the resolved model's cells (the base `number-input` fields plus `maxStars`) and add `'star-rating': <count>` to `xwalk/max-cells`.
6. Review (Step 8): all checks pass, and `npm run lint` is clean.
7. Ask the user to verify: the stars render, `maxStars: 3` renders 3 stars, a rule that sets the value to 4 selects 4 stars, clicking star 3 sets the value to `3` and fires change rules once, and Enter on a star works from the keyboard.
