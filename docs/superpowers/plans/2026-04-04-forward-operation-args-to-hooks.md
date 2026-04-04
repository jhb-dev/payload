# Forward Operation Intent Args to Collection & Global Hooks

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make operation-level intent arguments (`draft`, `autosave`, `trash`) available in all collection and global hooks, so hook authors can distinguish request intent from document state.

**Architecture:** Add `autosave` and `trash` as optional properties to hook type definitions (collection + global), then pass them at every hook invocation site. `draft` was already added in a prior commit. Each arg is threaded individually (matching the existing pattern) rather than introducing a new bag object, to keep the API surface explicit and backwards-compatible. Tests use the established "module-level state capture" pattern to assert that hook args are received correctly.

**Tech Stack:** TypeScript, Vitest, Payload CMS core (`packages/payload`)

**Branch:** `claude/check-draft-arg-hooks-O7IFl` (continues from prior `draft` work)

---

## Scope

This plan covers three intent arguments:

| Arg        | Meaning                                                   | Where it matters                                                                                                                       |
| ---------- | --------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| `draft`    | Caller wants to save/query drafts                         | Already done (prior commit). Tests still needed.                                                                                       |
| `autosave` | Save triggered by autosave, not explicit user action      | `beforeValidate`, `beforeChange`, `afterChange` (write hooks)                                                                          |
| `trash`    | Caller is soft-deleting (trash) rather than hard-deleting | `beforeChange`, `afterChange` (update ops use trash), `beforeDelete`, `afterDelete`, `beforeRead`, `afterRead` (find ops accept trash) |

Out of scope: `overrideLock`, `publishAllLocales`, `publishSpecificLocale`, `unpublishAllLocales`, `depth`, `populate`, `select`, `sort`, `limit`, `page`, `pagination`, `joins`, `includeLockStatus`. These are query/presentation concerns, not intent that changes hook behavior. They can be added later if needed.

---

## File Map

### Type definitions (modify)

- `packages/payload/src/collections/config/types.ts` — Add `autosave?` and `trash?` to `BeforeValidateHook`, `BeforeChangeHook`, `AfterChangeHook`, `BeforeReadHook`, `AfterReadHook`, `BeforeDeleteHook`, `AfterDeleteHook`
- `packages/payload/src/globals/config/types.ts` — Add `autosave?` and `trash?` to `BeforeValidateHook`, `BeforeChangeHook`, `AfterChangeHook`, `BeforeReadHook`, `AfterReadHook`

### Collection operation files (modify) — pass args to hook invocations

- `packages/payload/src/collections/operations/create.ts` — `autosave` to beforeValidate, beforeChange, afterRead, afterChange
- `packages/payload/src/collections/operations/utilities/update.ts` — `autosave`, `trash` (received from caller) to beforeValidate, beforeChange, afterRead, afterChange
- `packages/payload/src/collections/operations/find.ts` — `trash` to beforeRead, afterRead
- `packages/payload/src/collections/operations/findByID.ts` — (no trash arg on findByID, skip)
- `packages/payload/src/collections/operations/deleteByID.ts` — `trash` to beforeDelete, afterDelete, afterRead
- `packages/payload/src/collections/operations/delete.ts` — `trash` to beforeDelete, afterDelete
- `packages/payload/src/collections/operations/restoreVersion.ts` — (no autosave/trash, skip)

### Global operation files (modify)

- `packages/payload/src/globals/operations/update.ts` — `autosave` to beforeValidate, beforeChange, afterRead, afterChange
- `packages/payload/src/globals/operations/findOne.ts` — (no autosave/trash args, skip)

### Thread `trash` into `updateDocument` (modify)

- `packages/payload/src/collections/operations/utilities/update.ts` — Add `trash` to `SharedUpdateDocumentArgs`, destructure it, pass to hooks
- `packages/payload/src/collections/operations/updateByID.ts` — Pass `trash` through to `updateDocument`
- `packages/payload/src/collections/operations/update.ts` — Pass `trash` through to `updateDocument`

### Test files (create + modify)

- `test/hooks/collections/DraftAndIntentHooks/index.ts` — New collection config for testing draft/autosave/trash in hooks
- `test/hooks/config.ts` — Register the new collection
- `test/hooks/int.spec.ts` — New test cases for draft, autosave, and trash args in hooks

---

## Task 1: Add `autosave` to collection hook types

**Files:**

- Modify: `packages/payload/src/collections/config/types.ts`

- [ ] **Step 1: Add `autosave` to write hook types**

In `packages/payload/src/collections/config/types.ts`, add `autosave?: boolean` to `BeforeValidateHook`, `BeforeChangeHook`, and `AfterChangeHook`. These are write-only hooks — `autosave` is not relevant for read or delete hooks.

```typescript
// BeforeValidateHook — add after `data` property:
  /**
   * Whether this operation was triggered by autosave.
   */
  autosave?: boolean

// BeforeChangeHook — add after `data` property:
  /**
   * Whether this operation was triggered by autosave.
   */
  autosave?: boolean

// AfterChangeHook — add after `doc` property:
  /**
   * Whether this operation was triggered by autosave.
   */
  autosave?: boolean
```

- [ ] **Step 2: Verify no type errors**

Run: `cd packages/payload && npx tsc --noEmit 2>&1 | head -20`

Adding optional properties to object types is always backwards-compatible, so this should produce no new errors.

- [ ] **Step 3: Commit**

```bash
git add packages/payload/src/collections/config/types.ts
git commit -m "chore: add autosave to collection hook types"
```

---

## Task 2: Add `trash` to collection hook types

**Files:**

- Modify: `packages/payload/src/collections/config/types.ts`

- [ ] **Step 1: Add `trash` to relevant hook types**

Add `trash?: boolean` to `BeforeChangeHook`, `AfterChangeHook` (update operations can be trash operations), `BeforeReadHook`, `AfterReadHook` (find accepts trash), `BeforeDeleteHook`, and `AfterDeleteHook`.

Do NOT add `trash` to `BeforeValidateHook` — trash operations skip validation, so this hook may not run during trash. Adding it would be misleading.

```typescript
// BeforeChangeHook — add after `draft` property:
  /**
   * Whether this operation is a soft-delete (trash) operation.
   */
  trash?: boolean

// AfterChangeHook — add after `draft` property:
  /**
   * Whether this operation is a soft-delete (trash) operation.
   */
  trash?: boolean

// BeforeReadHook — add after `draft` property:
  /**
   * Whether trashed documents are included in this query.
   */
  trash?: boolean

// AfterReadHook — add after `draft` property:
  /**
   * Whether trashed documents are included in this query.
   */
  trash?: boolean

// BeforeDeleteHook — add after `id` property:
  /**
   * Whether this is a soft-delete (trash) rather than a permanent delete.
   */
  trash?: boolean

// AfterDeleteHook — add after `id` property:
  /**
   * Whether this was a soft-delete (trash) rather than a permanent delete.
   */
  trash?: boolean
```

- [ ] **Step 2: Commit**

```bash
git add packages/payload/src/collections/config/types.ts
git commit -m "chore: add trash to collection hook types"
```

---

## Task 3: Add `autosave` and `trash` to global hook types

**Files:**

- Modify: `packages/payload/src/globals/config/types.ts`

- [ ] **Step 1: Add `autosave` to global write hook types**

Add `autosave?: boolean` to `BeforeValidateHook`, `BeforeChangeHook`, and `AfterChangeHook` in the globals config types.

```typescript
// BeforeValidateHook — add after `data` property:
  /**
   * Whether this operation was triggered by autosave.
   */
  autosave?: boolean

// BeforeChangeHook — add after `data` property:
  /**
   * Whether this operation was triggered by autosave.
   */
  autosave?: boolean

// AfterChangeHook — add after `doc` property:
  /**
   * Whether this operation was triggered by autosave.
   */
  autosave?: boolean
```

Globals do not have `trash` — they cannot be deleted, so no `trash` property is needed.

- [ ] **Step 2: Commit**

```bash
git add packages/payload/src/globals/config/types.ts
git commit -m "chore: add autosave to global hook types"
```

---

## Task 4: Pass `autosave` to hook invocations in `create.ts`

**Files:**

- Modify: `packages/payload/src/collections/operations/create.ts`

The `autosave` variable is available in the destructured args (around line 120). It needs to be passed to `beforeValidate`, `beforeChange`, `afterRead` (collection), and `afterChange` hook invocations.

- [ ] **Step 1: Add `autosave` to all 4 hook invocations**

Find each hook invocation block and add `autosave,` to the args object:

**beforeValidate** (~line 199):

```typescript
;(await hook({
  autosave,
  collection: collectionConfig,
  context: req.context,
  data,
  draft,
  operation: 'create',
  originalDoc: duplicatedFromDoc,
  req,
})) || data
```

**beforeChange** (~line 218):

```typescript
;(await hook({
  autosave,
  collection: collectionConfig,
  context: req.context,
  data,
  draft,
  operation: 'create',
  originalDoc: duplicatedFromDoc,
  req,
})) || data
```

**afterRead** (~line 387):

```typescript
;(await hook({
  collection: collectionConfig,
  context: req.context,
  doc: result,
  draft,
  overrideAccess,
  req,
})) || result
```

(No `autosave` here — afterRead type doesn't have it.)

**afterChange** (~line 419):

```typescript
;(await hook({
  autosave,
  collection: collectionConfig,
  context: req.context,
  data,
  doc: result,
  draft,
  operation: 'create',
  overrideAccess,
  previousDoc: {},
  req: args.req,
})) || result
```

- [ ] **Step 2: Commit**

```bash
git add packages/payload/src/collections/operations/create.ts
git commit -m "chore: pass autosave to hooks in create operation"
```

---

## Task 5: Thread `trash` into `updateDocument` and pass `autosave`/`trash` to hooks

**Files:**

- Modify: `packages/payload/src/collections/operations/utilities/update.ts`
- Modify: `packages/payload/src/collections/operations/updateByID.ts`
- Modify: `packages/payload/src/collections/operations/update.ts`

The `updateDocument` function in `utilities/update.ts` doesn't currently receive `trash`. It needs to be added to `SharedUpdateDocumentArgs` and then passed through from the callers (`updateByID.ts` and `update.ts`).

- [ ] **Step 1: Add `trash` to `SharedUpdateDocumentArgs`**

In `packages/payload/src/collections/operations/utilities/update.ts`, add to the `SharedUpdateDocumentArgs` type (~line 41):

```typescript
export type SharedUpdateDocumentArgs<TSlug extends CollectionSlug> = {
  autosave: boolean
  collectionConfig: SanitizedCollectionConfig
  config: SanitizedConfig
  data: DeepPartial<DataFromCollectionSlug<TSlug>>
  depth: number
  docWithLocales: JsonObject & TypeWithID
  draftArg: boolean
  fallbackLocale: TypedFallbackLocale
  filesToUpload: FileToSave[]
  id: number | string
  locale: string
  overrideAccess: boolean
  overrideLock: boolean
  payload: Payload
  // ... existing properties ...
  trash?: boolean // <-- ADD THIS
}
```

- [ ] **Step 2: Destructure `trash` in `updateDocument`**

In the `updateDocument` function, destructure `trash` from args alongside `autosave` and `draftArg`.

- [ ] **Step 3: Pass `autosave` and `trash` to all 4 hook invocations in `updateDocument`**

**beforeValidate** (~line 204):

```typescript
;(await hook({
  autosave,
  collection: collectionConfig,
  context: req.context,
  data,
  draft: draftArg,
  operation: 'update',
  originalDoc,
  req,
})) || data
```

**beforeChange** (~line 231):

```typescript
;(await hook({
  autosave,
  collection: collectionConfig,
  context: req.context,
  data,
  draft: draftArg,
  operation: 'update',
  originalDoc,
  req,
  trash,
})) || data
```

**afterRead** (~line 419):

```typescript
;(await hook({
  collection: collectionConfig,
  context: req.context,
  doc: result,
  draft: draftArg,
  overrideAccess,
  req,
})) || result
```

**afterChange** (~line 451):

```typescript
;(await hook({
  autosave,
  collection: collectionConfig,
  context: req.context,
  data,
  doc: result,
  draft: draftArg,
  operation: 'update',
  overrideAccess,
  previousDoc: originalDoc,
  req,
  trash,
})) || result
```

- [ ] **Step 4: Pass `trash` from `updateByID.ts` to `updateDocument`**

In `packages/payload/src/collections/operations/updateByID.ts`, find where `updateDocument` is called (~line 200) and ensure `trash` is passed through. The `trash` variable is available in the destructured args (~line 89). Look for the call like:

```typescript
return updateDocument({
  // ... existing args ...
  trash, // <-- ADD THIS
})
```

- [ ] **Step 5: Pass `trash` from `update.ts` (bulk) to `updateDocument`**

In `packages/payload/src/collections/operations/update.ts`, find where `updateDocument` is called in the loop (~line 250) and pass `trash`. The variable `shouldSaveDraft` exists but `trash` comes from the operation args. Check the destructured args for `trash` (~line 97).

Look for the pattern and add `trash`:

```typescript
return updateDocument({
  // ... existing args ...
  trash, // <-- ADD THIS
})
```

Note: The bulk `update.ts` may not destructure `trash` by default. Check the `Arguments` type — if `trash` is not there, it might be computed from the `where` clause. Read the file carefully before editing.

- [ ] **Step 6: Commit**

```bash
git add packages/payload/src/collections/operations/utilities/update.ts packages/payload/src/collections/operations/updateByID.ts packages/payload/src/collections/operations/update.ts
git commit -m "chore: pass autosave and trash to hooks in update operations"
```

---

## Task 6: Pass `trash` to hook invocations in `find.ts`

**Files:**

- Modify: `packages/payload/src/collections/operations/find.ts`

The `find` operation accepts a `trash` argument (check the `Arguments` type ~line 37). It needs to be passed to `beforeRead` and `afterRead` hook invocations.

- [ ] **Step 1: Verify `trash` is destructured**

Read `find.ts` around line 85 to confirm `trash` is destructured from args. If not, add it.

- [ ] **Step 2: Pass `trash` to `beforeRead` hook invocation (~line 299)**

```typescript
;(await hook({
  collection: collectionConfig,
  context: req.context,
  doc: docRef,
  draft: draftsEnabled,
  overrideAccess: overrideAccess!,
  query: fullWhere,
  req,
  trash,
})) || docRef
```

- [ ] **Step 3: Pass `trash` to `afterRead` hook invocation (~line 354)**

```typescript
;(await hook({
  collection: collectionConfig,
  context: req.context,
  doc: docRef,
  draft: draftsEnabled,
  findMany: true,
  overrideAccess: overrideAccess!,
  query: fullWhere,
  req,
  trash,
})) || docRef
```

- [ ] **Step 4: Commit**

```bash
git add packages/payload/src/collections/operations/find.ts
git commit -m "chore: pass trash to hooks in find operation"
```

---

## Task 7: Pass `trash` to hook invocations in delete operations

**Files:**

- Modify: `packages/payload/src/collections/operations/deleteByID.ts`
- Modify: `packages/payload/src/collections/operations/delete.ts`

- [ ] **Step 1: Read `deleteByID.ts` to find `trash` variable and hook invocations**

Read the file to find:

1. Where `trash` is destructured from args
2. The `beforeDelete` hook invocation
3. The `afterDelete` hook invocation
4. The `afterRead` hook invocation

- [ ] **Step 2: Pass `trash` to `beforeDelete`, `afterDelete`, and `afterRead` hooks in `deleteByID.ts`**

Add `trash,` to each hook invocation's args object.

- [ ] **Step 3: Read `delete.ts` (bulk delete) to find hook invocations**

Read the file to find `beforeDelete` and `afterDelete` hook invocations and where `trash` is available.

- [ ] **Step 4: Pass `trash` to hooks in `delete.ts`**

Add `trash,` to each hook invocation's args object.

- [ ] **Step 5: Commit**

```bash
git add packages/payload/src/collections/operations/deleteByID.ts packages/payload/src/collections/operations/delete.ts
git commit -m "chore: pass trash to hooks in delete operations"
```

---

## Task 8: Pass `autosave` to hook invocations in global `update.ts`

**Files:**

- Modify: `packages/payload/src/globals/operations/update.ts`

The global update operation has `autosave` in its args (~line 72). Pass it to `beforeValidate`, `beforeChange`, `afterRead`, and `afterChange` hook invocations.

- [ ] **Step 1: Add `autosave` to all 4 hook invocations**

**beforeValidate** (~line 214):

```typescript
;(await hook({
  autosave,
  context: req.context,
  data,
  draft: draftArg,
  global: globalConfig,
  originalDoc,
  overrideAccess,
  req,
})) || data
```

**beforeChange** (~line 233):

```typescript
;(await hook({
  autosave,
  context: req.context,
  data,
  draft: draftArg,
  global: globalConfig,
  originalDoc,
  overrideAccess,
  req,
})) || data
```

**afterRead** (~line 431):

```typescript
;(await hook({
  context: req.context,
  doc: result,
  draft: draftArg,
  global: globalConfig,
  overrideAccess,
  req,
})) || result
```

(No `autosave` on afterRead — not a write hook type for globals.)

**afterChange** (~line 463):

```typescript
;(await hook({
  autosave,
  context: req.context,
  data,
  doc: result,
  draft: draftArg,
  global: globalConfig,
  overrideAccess,
  previousDoc: originalDoc,
  req,
})) || result
```

- [ ] **Step 2: Commit**

```bash
git add packages/payload/src/globals/operations/update.ts
git commit -m "chore: pass autosave to hooks in global update operation"
```

---

## Task 9: Create test collection for intent args in hooks

**Files:**

- Create: `test/hooks/collections/DraftAndIntentHooks/index.ts`

- [ ] **Step 1: Create the test collection**

This collection uses the "module-level state capture" pattern (same as `BeforeOperation/index.ts`) to record which args each hook received.

```typescript
import type { CollectionConfig } from 'payload'

export const draftAndIntentHooksSlug = 'draft-and-intent-hooks'

type CapturedHookArgs = {
  autosave?: boolean
  draft?: boolean
  trash?: boolean
}

type CapturedHooks = {
  afterChange?: CapturedHookArgs
  afterRead?: CapturedHookArgs
  beforeChange?: CapturedHookArgs
  beforeDelete?: CapturedHookArgs
  afterDelete?: CapturedHookArgs
  beforeRead?: CapturedHookArgs
  beforeValidate?: CapturedHookArgs
}

let capturedArgs: CapturedHooks = {}

export const getCapturedArgs = () => capturedArgs
export const clearCapturedArgs = () => {
  capturedArgs = {}
}

export const DraftAndIntentHooksCollection: CollectionConfig = {
  slug: draftAndIntentHooksSlug,
  versions: {
    drafts: true,
  },
  access: {
    read: () => true,
    create: () => true,
    update: () => true,
    delete: () => true,
  },
  hooks: {
    beforeValidate: [
      ({ data, draft, autosave }) => {
        capturedArgs.beforeValidate = { draft, autosave }
        return data
      },
    ],
    beforeChange: [
      ({ data, draft, autosave, trash }) => {
        capturedArgs.beforeChange = { draft, autosave, trash }
        return data
      },
    ],
    afterChange: [
      ({ doc, draft, autosave, trash }) => {
        capturedArgs.afterChange = { draft, autosave, trash }
        return doc
      },
    ],
    beforeRead: [
      ({ doc, draft, trash }) => {
        capturedArgs.beforeRead = { draft, trash }
        return doc
      },
    ],
    afterRead: [
      ({ doc, draft, trash }) => {
        capturedArgs.afterRead = { draft, trash }
        return doc
      },
    ],
    beforeDelete: [
      ({ trash }) => {
        capturedArgs.beforeDelete = { trash }
      },
    ],
    afterDelete: [
      ({ trash }) => {
        capturedArgs.afterDelete = { trash }
      },
    ],
  },
  fields: [
    {
      name: 'title',
      type: 'text',
    },
  ],
}
```

- [ ] **Step 2: Commit**

```bash
git add test/hooks/collections/DraftAndIntentHooks/index.ts
git commit -m "chore: add test collection for draft/autosave/trash hook args"
```

---

## Task 10: Register test collection and write integration tests

**Files:**

- Modify: `test/hooks/config.ts`
- Modify: `test/hooks/int.spec.ts`

- [ ] **Step 1: Register the new collection in config.ts**

Add the import and include it in the collections array:

```typescript
// Add import at top:
import {
  DraftAndIntentHooksCollection,
} from './collections/DraftAndIntentHooks/index.js'

// Add to collections array:
collections: [
  // ... existing collections ...
  DraftAndIntentHooksCollection,
],
```

- [ ] **Step 2: Write integration tests in int.spec.ts**

Add a new `describe` block at the end of the test file. Import the helpers:

```typescript
import {
  draftAndIntentHooksSlug,
  getCapturedArgs,
  clearCapturedArgs,
} from './collections/DraftAndIntentHooks/index.js'
```

Then add tests:

```typescript
describe('operation intent args in hooks', () => {
  const createdIDs: (number | string)[] = []

  afterEach(async () => {
    clearCapturedArgs()
    for (const id of createdIDs) {
      try {
        await payload.delete({ collection: draftAndIntentHooksSlug, id })
      } catch {
        // may already be deleted
      }
    }
    createdIDs.length = 0
  })

  describe('draft arg', () => {
    it('should pass draft=true to beforeValidate, beforeChange, afterChange on create', async () => {
      const doc = await payload.create({
        collection: draftAndIntentHooksSlug,
        data: { title: 'Draft Test' },
        draft: true,
      })
      createdIDs.push(doc.id)

      const captured = getCapturedArgs()
      expect(captured.beforeValidate?.draft).toBe(true)
      expect(captured.beforeChange?.draft).toBe(true)
      expect(captured.afterChange?.draft).toBe(true)
    })

    it('should pass draft=false/undefined to hooks when not drafting', async () => {
      const doc = await payload.create({
        collection: draftAndIntentHooksSlug,
        data: { title: 'Published Test' },
        draft: false,
      })
      createdIDs.push(doc.id)

      const captured = getCapturedArgs()
      expect(captured.beforeValidate?.draft).toBeFalsy()
      expect(captured.beforeChange?.draft).toBeFalsy()
      expect(captured.afterChange?.draft).toBeFalsy()
    })

    it('should pass draft to beforeRead and afterRead on find', async () => {
      const doc = await payload.create({
        collection: draftAndIntentHooksSlug,
        data: { title: 'Read Draft Test' },
        draft: true,
      })
      createdIDs.push(doc.id)

      clearCapturedArgs()

      await payload.find({
        collection: draftAndIntentHooksSlug,
        draft: true,
      })

      const captured = getCapturedArgs()
      expect(captured.beforeRead?.draft).toBe(true)
      expect(captured.afterRead?.draft).toBe(true)
    })

    it('should pass draft to hooks on update', async () => {
      const doc = await payload.create({
        collection: draftAndIntentHooksSlug,
        data: { title: 'Update Draft Test' },
      })
      createdIDs.push(doc.id)

      clearCapturedArgs()

      await payload.update({
        collection: draftAndIntentHooksSlug,
        id: doc.id,
        data: { title: 'Updated' },
        draft: true,
      })

      const captured = getCapturedArgs()
      expect(captured.beforeValidate?.draft).toBe(true)
      expect(captured.beforeChange?.draft).toBe(true)
      expect(captured.afterChange?.draft).toBe(true)
    })
  })

  describe('autosave arg', () => {
    it('should pass autosave=true to write hooks on create', async () => {
      const doc = await payload.create({
        collection: draftAndIntentHooksSlug,
        data: { title: 'Autosave Test' },
        draft: true,
        autosave: true,
      })
      createdIDs.push(doc.id)

      const captured = getCapturedArgs()
      expect(captured.beforeValidate?.autosave).toBe(true)
      expect(captured.beforeChange?.autosave).toBe(true)
      expect(captured.afterChange?.autosave).toBe(true)
    })

    it('should pass autosave=false/undefined when not autosaving', async () => {
      const doc = await payload.create({
        collection: draftAndIntentHooksSlug,
        data: { title: 'Normal Save Test' },
      })
      createdIDs.push(doc.id)

      const captured = getCapturedArgs()
      expect(captured.beforeValidate?.autosave).toBeFalsy()
      expect(captured.beforeChange?.autosave).toBeFalsy()
      expect(captured.afterChange?.autosave).toBeFalsy()
    })

    it('should pass autosave=true to write hooks on update', async () => {
      const doc = await payload.create({
        collection: draftAndIntentHooksSlug,
        data: { title: 'Autosave Update' },
      })
      createdIDs.push(doc.id)

      clearCapturedArgs()

      await payload.update({
        collection: draftAndIntentHooksSlug,
        id: doc.id,
        data: { title: 'Updated Autosave' },
        draft: true,
        autosave: true,
      })

      const captured = getCapturedArgs()
      expect(captured.beforeValidate?.autosave).toBe(true)
      expect(captured.beforeChange?.autosave).toBe(true)
      expect(captured.afterChange?.autosave).toBe(true)
    })
  })

  describe('trash arg', () => {
    it('should pass trash=true to beforeDelete and afterDelete hooks', async () => {
      const doc = await payload.create({
        collection: draftAndIntentHooksSlug,
        data: { title: 'Trash Delete Test' },
      })

      clearCapturedArgs()

      await payload.delete({
        collection: draftAndIntentHooksSlug,
        id: doc.id,
      })

      const captured = getCapturedArgs()
      // trash config defaults - check what value is passed
      expect(captured.beforeDelete).toBeDefined()
      expect(captured.afterDelete).toBeDefined()
    })

    it('should pass trash to beforeRead and afterRead on find with trash=true', async () => {
      const doc = await payload.create({
        collection: draftAndIntentHooksSlug,
        data: { title: 'Trash Find Test' },
      })
      createdIDs.push(doc.id)

      clearCapturedArgs()

      await payload.find({
        collection: draftAndIntentHooksSlug,
        where: { title: { equals: 'Trash Find Test' } },
      })

      const captured = getCapturedArgs()
      expect(captured.beforeRead).toBeDefined()
      expect(captured.afterRead).toBeDefined()
    })
  })
})
```

- [ ] **Step 3: Run the tests**

Run: `pnpm run test:int hooks`
Expected: All new tests pass, existing tests still pass.

- [ ] **Step 4: Commit**

```bash
git add test/hooks/config.ts test/hooks/int.spec.ts
git commit -m "test: add integration tests for draft, autosave, and trash args in hooks"
```

---

## Task 11: Build and final verification

**Files:** None (verification only)

- [ ] **Step 1: Build the payload package**

Run: `pnpm turbo build --filter payload`
Expected: Build succeeds with no errors.

- [ ] **Step 2: Run the full hooks test suite**

Run: `pnpm run test:int hooks`
Expected: All tests pass.

- [ ] **Step 3: Push**

```bash
git push -u origin claude/check-draft-arg-hooks-O7IFl
```
