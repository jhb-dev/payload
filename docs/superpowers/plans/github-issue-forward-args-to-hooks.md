# GitHub Issue: Forward full operation args to collection and global hooks

**Title:** `feat: forward full operation args to collection and global hooks`
**Labels:** `enhancement`

---

## Summary

Collection and global hooks (`beforeValidate`, `beforeChange`, `afterChange`, `beforeRead`, `afterRead`, `beforeDelete`, `afterDelete`) currently receive a **hand-picked subset** of operation arguments. This means hook authors cannot access critical request-intent arguments like `draft`, `autosave`, or `trash` — even though these fundamentally change what the operation is doing.

`beforeOperation` and `afterOperation` already receive the **full `args` object** via a discriminated union. The other hooks don't, creating an inconsistency where users must use `beforeOperation` as a workaround just to capture context that should be available everywhere.

## Problem

### Request intent vs. document state

Arguments like `draft`, `autosave`, and `trash` represent **caller intent** — what the operation is trying to do. This is different from document state (e.g., `_status: 'draft'`), which reflects the current data.

Example: A `beforeChange` hook on an update where `draft: false` — the document's `_status` could still be `'draft'` from a previous save, but the current request intends to **publish** it. Without the `draft` arg, the hook can't distinguish this.

### Current state of affairs

| Hook              | Gets `draft`? | Gets `autosave`? | Gets `trash`? | Gets full `args`? |
| ----------------- | :-----------: | :--------------: | :-----------: | :---------------: |
| `beforeOperation` |  via `args`   |    via `args`    |  via `args`   |      **yes**      |
| `afterOperation`  |  via `args`   |    via `args`    |  via `args`   |      **yes**      |
| `beforeValidate`  |      no       |        no        |      no       |        no         |
| `beforeChange`    |      no       |        no        |      no       |        no         |
| `afterChange`     |      no       |        no        |      no       |        no         |
| `beforeRead`      |      no       |       n/a        |      no       |        no         |
| `afterRead`       |      no       |       n/a        |      no       |        no         |
| `beforeDelete`    |      n/a      |       n/a        |      no       |        no         |
| `afterDelete`     |      n/a      |       n/a        |      no       |        no         |
| Field `afterRead` |    **yes**    |        no        |      no       |        no         |

### The maintenance problem

Every time a new meaningful operation argument is added, someone must:

1. Add it to each hook type definition (collection + global = 2 files)
2. Thread it through every hook invocation site across all operations (10+ files, 20+ call sites)
3. Remember which hooks should get which args

This is error-prone. `draft` was already missed until now. `autosave` and `trash` are still missing.

## Proposed solution

Add an `args` property to every hook that provides the **full operation arguments**, matching what `beforeOperation`/`afterOperation` already do.

### Before (current)

```typescript
const myCollection: CollectionConfig = {
  hooks: {
    beforeChange: [
      ({ data, operation, req }) => {
        // Can't tell if this is a draft save, autosave, or trash operation
        // Must use beforeOperation as workaround to capture this info
      },
    ],
  },
}
```

### After (proposed)

```typescript
const myCollection: CollectionConfig = {
  hooks: {
    beforeChange: [
      ({ data, operation, req, args }) => {
        if (args.draft) {
          // This is a draft save — skip external sync
        }
        if (args.autosave) {
          // This is an autosave — skip notifications
        }
      },
    ],
  },
}
```

### Implementation approach

1. **Add `args` property to all hook type definitions** — typed using the same discriminated union pattern that `beforeOperation`/`afterOperation` already use
2. **Pass `args` at each invocation site** — a single property addition per call site
3. **Fully backwards compatible** — `args` is a new optional property; existing hooks are unaffected
4. **Future-proof** — new operation arguments are automatically available in all hooks without further plumbing

### What `args` contains (per operation)

The `args` object is the same one already defined in `packages/payload/src/collections/operations/utilities/types.ts`. For each operation variant it includes all operation-level arguments:

| Operation               | Key args available                                                                                            |
| ----------------------- | ------------------------------------------------------------------------------------------------------------- |
| `create`                | `data`, `draft`, `autosave`, `overrideAccess`, `depth`, `populate`, `select`, `locale`                        |
| `update` / `updateByID` | `data`, `draft`, `autosave`, `trash`, `overrideAccess`, `depth`, `populate`, `select`, `locale`, `where`/`id` |
| `find` / `findByID`     | `draft`, `trash`, `overrideAccess`, `depth`, `populate`, `select`, `where`/`id`, `sort`, `limit`, `page`      |
| `delete` / `deleteByID` | `trash`, `overrideAccess`, `depth`, `where`/`id`                                                              |

### Files involved

**Type definitions:**

- `packages/payload/src/collections/config/types.ts` — Add `args` to `BeforeValidateHook`, `BeforeChangeHook`, `AfterChangeHook`, `BeforeReadHook`, `AfterReadHook`, `BeforeDeleteHook`, `AfterDeleteHook`
- `packages/payload/src/globals/config/types.ts` — Same for global hook types

**Hook invocation sites (pass `args` through):**

- `packages/payload/src/collections/operations/create.ts`
- `packages/payload/src/collections/operations/utilities/update.ts`
- `packages/payload/src/collections/operations/find.ts`
- `packages/payload/src/collections/operations/findByID.ts`
- `packages/payload/src/collections/operations/deleteByID.ts`
- `packages/payload/src/collections/operations/delete.ts`
- `packages/payload/src/collections/operations/restoreVersion.ts`
- `packages/payload/src/globals/operations/findOne.ts`
- `packages/payload/src/globals/operations/update.ts`
- `packages/payload/src/globals/operations/restoreVersion.ts`

### Safety considerations

- The `args` object is **not mutated** during operation flow in core Payload code
- The local API already calls `deepCopyObjectSimple(data)` before passing to operations, protecting the original `data` from hook mutations
- The `args` reference is the same one passed to `beforeOperation`/`afterOperation`, so this is not a new exposure — just making it consistent

## Alternatives considered

1. **Add individual properties one by one** (`draft`, `autosave`, `trash` each threaded separately) — Works but doesn't scale; every new arg requires touching 10+ files and 20+ invocation sites. Still valuable as an interim step.

2. **Attach operation metadata to `req`** — Pollutes the request object with operation-specific state and creates ambiguity when hooks trigger nested operations.

3. **Use `context` for passing intent** — Already possible as a userland workaround via `beforeOperation`, but shouldn't be necessary for first-party operation arguments.
