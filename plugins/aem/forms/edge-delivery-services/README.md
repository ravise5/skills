# AEM Forms on Edge Delivery Services

Skills for building forms with **AEM Forms on Edge Delivery Services** — the Form block from [`aem-boilerplate-forms`](https://github.com/adobe-rnd/aem-boilerplate-forms).

```bash
/plugin install aem-forms-edge-delivery-services@adobe-skills
```

## Skills

| Skill | Description |
|-------|-------------|
| [`create-custom-component`](skills/create-custom-component/SKILL.md) | Create a custom form component that extends an OOTB field type: component files (created by the agent, no scaffolder), authoring schema (`_<name>.json`), `decorate` + `subscribe()` model wiring, registration in `mappings.js`, README, built authoring JSON (merged by the agent), and review |

## Built authoring JSON and review

`create-custom-component` doesn't run a scaffolder, `build:json` or browser tests. The agent writes every file itself and checks the result with one dependency-free validator:

| Step | What the agent does |
|------|---------------------|
| Build | Instead of running `npm run build:json`, it does what [`merge-json-cli`](https://github.com/buuhuu/merge-json-cli) does by hand. It resolves the `"..."` includes in `_<name>.json` and merges the model, definition and filter entry into the root `component-models.json`, `component-definition.json` and `component-filters.json`. It also computes the `xwalk/max-cells` entry for the new model. See [`references/build-json.md`](skills/create-custom-component/references/build-json.md), which uses the boilerplate's `rating` component as the worked example |
| Review | `node skills/create-custom-component/scripts/validate-component.mjs <name> [--root <project>] [--json]` checks naming, JSON schema, `fd:viewType`, `"..."` includes, JS conventions, registration, the hand-merged build output (including leftover `"..."`), and the `xwalk/max-cells` limit, and exits `1` on errors. A checklist covers what it can't check (loop guards, value commit timing, `component-filters.json`, unrelated diffs). Runs `npm run lint` when the project can run it |
| User verification | Gives you concrete steps to try the component in a browser, asks for feedback, and iterates until you approve |

The validator has unit tests:

```bash
node --test plugins/aem/forms/edge-delivery-services/tests/*.test.mjs
```

## Evals

Tessl eval scenarios live in [`evals/`](evals) (`task.md` + `criteria.json` weighted checklists) and run on PRs via a commit starting with `eval:` (see [CONTRIBUTING](../../../../CONTRIBUTING.md#requesting-evals)).

| Scenario | What it measures |
|----------|------------------|
| `range-slider-from-number-input` | End-to-end component: files, `fd:viewType`, authoring model, `subscribe` with `listenChanges`, registration, README, honest validation report |
| `panel-watches-child-checkboxes` | Panel component subscribing to child fields via `[data-id]` wrappers |
| `custom-event-verification-badge` | Custom events through `fieldModel.subscribe(cb, '<event>')`, not DOM listeners |
| `commit-value-per-keystroke` | OOTB commits on blur; the component commits on each keystroke (formatted display, raw value) and documents the contract |
| `repair-unregistered-star-rating` | Diagnose and fix a broken component (naming, viewType, registration, includes, legacy patterns) and report verification |
| `searchable-dropdown-authoring-schema` | Universal Editor widget choice, shared-field reuse, runtime `enum` updates |
