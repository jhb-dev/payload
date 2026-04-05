# Bug Report: Operation intent args missing from collection and global hooks

> For submitting at https://github.com/payloadcms/payload/issues using the "Functionality Bug" template.

---

### Describe the Bug

Collection and global hooks (`beforeValidate`, `beforeChange`, `afterChange`, `beforeRead`, `afterRead`, `beforeDelete`, `afterDelete`) do not receive operation-level intent arguments like `draft`, `autosave`, or `trash`. These arguments represent **caller intent** — what the operation is trying to do — and are distinct from document state (e.g., `_status: 'draft'`).

This is inconsistent: `beforeOperation` and `afterOperation` hooks already receive the full `args` object containing all operation arguments. Field-level `afterRead` hooks also receive `draft`. But the most commonly used hooks — `beforeChange`, `afterChange`, `beforeRead`, `afterRead` — do not.

**Why this is a bug, not a feature request:**

1. **Inconsistency in the hook API.** `beforeOperation`/`afterOperation` get full args, all other hooks get an incomplete hand-picked subset. There's no documented reason for the difference.

2. **The `context` workaround is unsafe.** The suggested workaround is to stash values in `context` via `beforeOperation`, then read them in other hooks. But `context` is a shared mutable object — if a hook triggers a nested operation (e.g., `payload.update()` inside `afterChange`), that nested operation's `beforeOperation` hook overwrites the same `context` keys, corrupting the values for the outer operation. This is a race condition by design.

3. **Intent vs. state divergence causes incorrect behavior.** For example, a `beforeChange` hook cannot distinguish between:
   - An update that **publishes** a draft (`draft: false`, but `_status` is still `'draft'` on the incoming doc)
   - An update that **saves a new draft** (`draft: true`)
   - An **autosave** (`autosave: true`) where you'd want to skip side effects like sending notifications
   - A **soft-delete/trash** operation (`trash: true`) where the update is just setting `deletedAt`

**Current hook arg availability:**

| Hook              |  `draft`   | `autosave` |  `trash`   | Full `args` |
| ----------------- | :--------: | :--------: | :--------: | :---------: |
| `beforeOperation` | via `args` | via `args` | via `args` |   **yes**   |
| `afterOperation`  | via `args` | via `args` | via `args` |   **yes**   |
| `beforeValidate`  |     no     |     no     |     no     |     no      |
| `beforeChange`    |     no     |     no     |     no     |     no      |
| `afterChange`     |     no     |     no     |     no     |     no      |
| `beforeRead`      |     no     |    n/a     |     no     |     no      |
| `afterRead`       |     no     |    n/a     |     no     |     no      |
| `beforeDelete`    |    n/a     |    n/a     |     no     |     no      |
| `afterDelete`     |    n/a     |    n/a     |     no     |     no      |
| Field `afterRead` |  **yes**   |     no     |     no     |     no      |

**Race condition example with `context` workaround:**

```typescript
const myCollection: CollectionConfig = {
  hooks: {
    beforeOperation: [
      ({ args, context }) => {
        context.draft = args.draft // stash for later hooks
      },
    ],
    afterChange: [
      async ({ doc, req, context }) => {
        // context.draft is correct here... unless:
        await req.payload.update({
          collection: 'other-collection',
          id: doc.relatedId,
          data: { synced: true },
          // This triggers OTHER collection's beforeOperation,
          // which may overwrite context.draft
        })
        // context.draft may now reflect the INNER operation's value, not ours
      },
    ],
  },
}
```

**Proposed fix:** Add `args` as a property to all hook types — the same full operation args object that `beforeOperation`/`afterOperation` already receive. This is backwards compatible (new optional property) and future-proof (new operation args are automatically available without threading them individually).

```typescript
hooks: {
  beforeChange: [({ data, operation, req, args }) => {
    if (args.draft) { /* draft save — skip external sync */ }
    if (args.autosave) { /* autosave — skip notifications */ }
  }],
}
```

### Link to the code that reproduces this issue

No external reproduction needed — this is visible in the source code:

- Hook type definitions: `packages/payload/src/collections/config/types.ts` (lines 178-274)
- Hook invocation in create: `packages/payload/src/collections/operations/create.ts` (lines 196-429)
- Compare with beforeOperation which gets full args: `packages/payload/src/collections/operations/utilities/buildBeforeOperation.ts` (line 94)

### Reproduction Steps

1. Create a collection with `versions: { drafts: true }` and a `beforeChange` hook
2. In the hook, try to access the `draft` argument: `({ draft }) => { console.log(draft) }`
3. Call `payload.create({ collection: 'my-collection', data: { title: 'test' }, draft: true })`
4. Observe that `draft` is `undefined` in the hook — the argument is not forwarded
5. Same issue with `autosave` in write hooks and `trash` in delete/update hooks

### Which area(s) are affected?

- `area: core`
