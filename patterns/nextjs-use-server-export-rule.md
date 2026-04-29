---
title: Next.js "use server" — only async functions can be exported
tags: [patterns, nextjs, server-actions, gotcha]
created: 2026-04-29
updated: 2026-04-29
status: active
sources:
  - mytodo/src/app/actions.ts (the bug that found this)
related:
  - ../projects/mytodo.md
---

# Next.js `"use server"` — exports rule

A subtle but devastating Next.js gotcha: in any file with `"use server"` at the top, **only async functions can be exported**. Re-exporting a constant, class, type-as-value, or anything else throws at runtime.

## What breaks

```ts
// src/app/actions.ts
"use server";
import { revalidatePath } from "next/cache";
import { db, dbConfigured } from "@/lib/db";

export async function toggleTask(id: string) { ... }  // ✅ fine

export { dbConfigured };  // 💥 BREAKS EVERY ACTION IN THIS FILE
```

The `export { dbConfigured }` (re-exporting a boolean) makes Next.js throw when it loads the file for action execution. **Every** action in the file returns HTTP 500. The error is not surfaced clearly in client-side React — `startTransition`'s callback completes silently, the UI looks frozen.

## Why the build doesn't catch it

In Next.js 16 (and earlier?), `next build` does NOT validate this rule. The build succeeds, the page loads, the JS bundle is fine. The runtime check only fires when an action is actually invoked. So you don't notice in dev (if you don't click anything) or in CI.

## How to debug a "Server Action does nothing" symptom

1. Open browser dev tools, Network tab
2. Click the failing action (edit, delete, etc.)
3. Look for a POST to `/` (or whatever page) with `Next-Action` header
4. If it returns **500**, this is your bug — check what's exported from your `"use server"` file
5. Check Vercel function logs for stack traces

## The rule, stated precisely

In any file marked `"use server"`:
- ✅ `export async function foo() {}`
- ✅ `export async function bar(formData: FormData) {}`
- ⚠️ `export type Foo = ...` — TypeScript types ARE OK because they erase at compile time
- ❌ `export const FOO = "bar";`
- ❌ `export class MyClass {}`
- ❌ `export function notAsync() {}` (must be async)
- ❌ `export { something }` (re-exports — even if `something` IS async, the re-export form trips Next.js' check)

If you need to share a constant or non-function value with client/server components, put it in a separate non-`"use server"` file and import from there.

## Found via

mytodo, 2026-04-29 — every web action (edit, delete, toggle, bulk-*) silently failing with 500. Calvin reported "edit/delete failing", I assumed it was middleware/Clerk/hydration. Spent hours on red herrings. Eventual fix: removing one line.

The smoking gun was hooking `window.fetch` in the browser to capture the response status:

```js
const o = window.fetch;
window.fetch = async (...a) => {
  const r = await o.apply(this, a);
  console.log(r.status, a[0]);
  return r;
};
```

Then clicking the failing button. The 500 immediately gave it away.

## See also

- [mytodo project page](../projects/mytodo.md)
- Next.js Server Actions docs: https://nextjs.org/docs/app/building-your-application/data-fetching/server-actions-and-mutations
