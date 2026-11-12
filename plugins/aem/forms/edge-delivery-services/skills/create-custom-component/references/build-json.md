# Updating the built component JSON by hand (`build:json`)

The Universal Editor does not read the `_*.json` source files. It reads three **generated** files at the repository root:

| Generated file | Source | Shape |
|----------------|--------|-------|
| `component-models.json` | `models/_component-models.json` | Array of models: `[{ "id", "fields": [...] }, ...]` |
| `component-definition.json` | `models/_component-definition.json` | `{ "groups": [{ "title", "id", "components": [definition, ...] }, ...] }` |
| `component-filters.json` | `models/_component-filters.json` | Array of filters: `[{ "id", "components": ["<id>", ...] }, ...]` |

In `aem-boilerplate-forms`, `npm run build:json` produces them with [`merge-json-cli`](https://github.com/buuhuu/merge-json-cli):

```json
"build:json:models": "merge-json-cli -i \"models/_component-models.json\" -o \"component-models.json\"",
"build:json:definitions": "merge-json-cli -i \"models/_component-definition.json\" -o \"component-definition.json\"",
"build:json:filters": "merge-json-cli -i \"models/_component-filters.json\" -o \"component-filters.json\""
```

In this skill, **you do this merge yourself** instead of running the script: you edit the three generated files so they contain exactly what `merge-json-cli` would write for the new component. Update the source files first (Step 5 of `SKILL.md`), then the generated files, so the two never disagree. If someone runs `npm run build:json` later, it must produce no diff.

## 1. How `merge-json-cli` resolves `"..."`

The `"..."` key works like the JavaScript spread operator. Its value is `"<file>#<pointer>"`.

- **`<file>`** is resolved **relative to the file that contains the `"..."`**, not relative to the repository root. A reference inside `blocks/form/components/rating/_rating.json` to `../../models/form-common/_help-container.json` points to `blocks/form/models/form-common/_help-container.json`.
- **`#<pointer>`** selects a part of that file, like `#/fields`, `#/models` or `#/definitions`. Segments are object keys. When a segment meets an **array**, it selects the item whose `id` (or `name`) equals the segment. With no pointer, the whole file is used. With no file (`"#/x"`), the pointer is read from the current file.
- **Glob patterns** (`../blocks/*/_*.json`, `./components/*/_*.json`) are allowed only inside arrays. Matching files are processed in **alphabetical order of their resolved paths**.
- **Inside an array**, an item `{ "...": ref }` with **no other keys** is replaced by the referenced value. If that value is an array, its items are spliced in at that position (flattened). Otherwise, the object itself is inserted.
- **Inside an object** (or an array item that has other keys besides `"..."`), the referenced object's keys are inserted where `"..."` stands. If a key appears both in the referenced object and in the source object, the one that comes **later** wins. So keys written after `"..."` override referenced keys, and keys written before it are overridden.
- Resolution is **recursive**: a referenced file can contain its own `"..."` entries, and those resolve relative to **that** file.
- The output is `JSON.stringify(result, null, 2)`: two-space indentation, no trailing newline. Keep that format so a later real build produces no diff.

The resolved output **never contains a `"..."` key**. If one remains, the merge is incomplete.

## 2. Find where the component is included

Follow the chain from the root source files to the component's `_<name>.json`. In the boilerplate:

| Output | Chain | Where the new entry lands |
|--------|-------|---------------------------|
| `component-models.json` | `models/_component-models.json` → `{ "...": "../blocks/*/_*.json#/models" }` → `blocks/form/_form.json` `models` → `{ "...": "./components/*/_*.json#/models" }` | Picked up by the glob, no source change needed. It goes among the other `blocks/form/components/*` models, in **alphabetical order of the folder name** (in the boilerplate: `accordion`, `modal`, `password`, `range`, `rating`, `tnc`, `wizard`) |
| `component-definition.json` | `models/_component-definition.json` → group `custom-components` → `{ "...": "../blocks/form/components/<name>/_<name>.json#/definitions" }` (added in Step 5) | Appended to the `components` of the `custom-components` group, at the same position as its reference |
| `component-filters.json` | `models/_component-filters.json` → `{ "...": "../blocks/*/_*.json#/filters" }` → `blocks/form/_form.json` `filters` | `"<name>"` in the `components` of the filter with `"id": "form"`, at the same position as in `_form.json` |

Projects differ. Always read the actual source files and follow their chain. Glob patterns, group ids and ordering can change, and a project may list components explicitly instead of by glob. For example, the boilerplate's OOTB `rating` definition is listed explicitly in `blocks/form/_form.json` `definitions`, so it lands in the `blocks` group. The custom `range` component is listed in the `custom-components` group of `models/_component-definition.json`. Include the definition **in only one place**, or it appears twice in the palette.

## 3. Resolve the component's model and definition

Start from `blocks/form/components/<name>/_<name>.json`:

1. Copy `models[0]` (`{ "id": "<name>", "fields": [...] }`).
2. Replace each `{ "...": "<file>#/fields" }` item in `fields` with the items of the referenced `fields` array, in place and in order. Resolve the path relative to the component folder. Repeat until no `"..."` remains, including inside `container` fields and inside the included files.
3. Copy `definitions[0]` and resolve any `"..."` the same way. Definitions usually have none.

### Worked example: `rating` from the boilerplate

Source `blocks/form/components/rating/_rating.json` (model):

```json
{
  "id": "rating",
  "fields": [
    { "component": "tab", "label": "Basic", "name": "basic" },
    { "...": "../../models/form-common/_basic-input-fields.json#/fields" },
    { "component": "tab", "label": "Validation", "name": "validation" },
    { "...": "../../models/form-common/_number-validation-fields.json#/fields" },
    { "...": "../../models/form-common/_help-container.json#/fields" }
  ]
}
```

`../../models/form-common/_help-container.json` resolves to `blocks/form/models/form-common/_help-container.json`, which contains:

```json
{
  "fields": [
    { "component": "tab", "label": "Help Content", "name": "help" },
    { "component": "richtext", "name": "description", "label": "Help text", "valueType": "string" },
    { "component": "richtext", "name": "tooltip", "label": "Short description", "valueType": "string" }
  ]
}
```

so the last `"..."` item is replaced by those three fields. After the other two includes are expanded the same way, the built `rating` model in `component-models.json` is a flat `fields` array with 21 entries (3 tabs + 18 fields: `name`, `jcr:title`, `hideTitle`, `dataRef`, `unboundFormElement`, `visible`, `enabled`, `readOnly`, `colspan`, `required`, `mandatoryMessage`, `validateExpMessage`, `minimum`, `minimumMessage`, `maximum`, `maximumMessage`, `description`, `tooltip`). Its definition is copied unchanged:

```json
{
  "title": "Rating",
  "id": "rating",
  "plugins": {
    "xwalk": {
      "page": {
        "resourceType": "core/fd/components/form/numberinput/v1/numberinput",
        "template": {
          "jcr:title": "Rating",
          "fieldType": "number-input",
          "fd:viewType": "rating"
        }
      }
    }
  }
}
```

and `"rating"` appears in the `form` filter's `components` in `component-filters.json`.

## 4. Edit the generated files

1. **`component-models.json`**: insert the resolved model at the position described in section 2. If a model with the same `id` already exists (re-run or fix), replace it instead of adding a second one.
2. **`component-definition.json`**: insert the resolved definition into the right group's `components`. Replace an existing entry with the same `id`.
3. **`component-filters.json`**: add `"<name>"` to the `form` filter's `components`, in the same position as in `_form.json`. Don't add it twice.
4. When you change a component later (new property, renamed field, different include), redo the resolution and **replace** the whole model entry. Don't patch single fields by hand. Patched fields drift from the source.

If a generated file doesn't exist yet but its `models/_component-*.json` source does, create it by resolving the **whole** source file with the rules in section 1, not just the new component. If the `models/_component-*.json` source doesn't exist, the project doesn't use this build. Edit only the source files, and say so in the report.

## 5. Check the result

- Each generated file is valid JSON. For example: `node -e "JSON.parse(require('fs').readFileSync('component-models.json','utf8'))"`.
- No `"..."` key remains anywhere in the three generated files.
- `component-models.json` has exactly one model with `"id": "<name>"`, and its field `name`s are unique. Every authored property from `_<name>.json` is there, along with every field from the included shared groups.
- `component-definition.json` has exactly one definition with `"id": "<name>"`, with `fd:viewType` equal to `<name>`.
- `component-filters.json`: the `form` filter lists `<name>` once.
- No other entries changed. Compare with `git diff` on the three files: only the new component's entries should show up.

## 6. `xwalk/max-cells` cell count

The xwalk ESLint config (`.eslintrc.cjs` or similar) usually sets `'xwalk/max-cells'`. It lints the **built** `component-models.json` and allows each model 4 cells, unless the model id has its own entry. Count the cells of the resolved model the way `eslint-plugin-xwalk` does:

1. Collect the `name` of every field. Skip `tab` fields. For `container` fields, collect their nested `fields` instead of the container itself.
2. Drop a name if it ends with `MimeType`, `Text`, `Title`, `Type` or `Alt` **and** the name without that suffix is also in the list (for example, `imageAlt` when `image` exists).
3. Strip everything from the first `_` on (`foo_bar` → `foo`), then count the **distinct** names.

For `rating`, this gives 18, which matches `rating: 18` in the boilerplate's `.eslintrc.cjs`. Add `'<name>': <count>` next to the existing entries.
