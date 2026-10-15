---
name: react-query-framer
description: Frame TanStack Query `queryOptions` (GET endpoints) and mutation API functions (POST/PUT/PATCH/DELETE endpoints) from curl requests and their response schemas, using the project's ofetch clients and zod schemas. Accepts a tagged file, pasted curl text plus a file location, or a whole directory of curl files. Keeps the curl as a comment in the framed file for later cross-verification. Never builds the feature itself.
---

# React Query Framer

Turn curl requests into ready-to-use `*-options.ts` or `*-api.ts` files (plus their zod schemas), each carrying its source curl as a comment. Nothing more.

## Scope (strict)

- Only frame the options/api (and the response schema they need).
- NEVER build the feature: no components, hooks, forms, routes, `useQuery`/`useMutation` calls, or UI wiring. Another agent or the user does that using what you frame.
- Do not build or modify ofetch clients. They already exist (or the user builds them). You only pick and import the right one.

## Input

One of:

1. **A tagged file** containing the curl, with the response schema/shape below it.
2. **Direct text** (curl) plus the file location where the output should go.
3. **A directory** of curl files. Process every file in it (see "Directory mode").

Extract from each curl: HTTP method, full path, path params, query params, body, and whether it is authenticated (has cookies/auth headers) or public.
Extract from the section below the curl: the response shape.

## Directory mode

When given a directory, go through each file in it (recursively, unless told otherwise) and treat each file as its own input.

For each file, decide one of:

- **Ignore**: the file is empty, or has no curl request. Do nothing, do not mention it beyond the final summary.
- **Skip**: a framed file for this curl already exists (see "Existing check" below) and the user did NOT ask for a recheck.
- **Frame**: it has a curl and nothing has been framed for it yet.
- **Recheck**: the user explicitly asked to recheck/reframe (e.g. "the curl changed, reframe it", "verify the implementation against the curl"). See "Recheck".

A single file may contain more than one curl; handle each curl separately.

Finish directory mode with one short summary listing, per file: framed / skipped (already framed) / ignored (empty or no curl) / rechecked.

### Existing check

A curl counts as already framed when an options/api file in the project has a `@curl` comment (see Step 7) with the same method and path. Search for it (grep for the method + path in `@curl` blocks across the options/api locations) before creating anything. This also guards against creating a duplicate file under a different name.

### Recheck

Only when the user explicitly asks. Compare the curl in the input file with the `@curl` comment in the existing framed file, then check the implementation against the new curl: method category (options vs api), path, `Ctx` keys and types, body/query, client choice, and the zod schema.

- If something differs, update the framed file and the schema, and replace the `@curl` comment with the new curl.
- If everything matches, change nothing and say it is consistent.
- If the method changed between GET and non-GET, the file moves between `-options.ts` and `-api.ts`: create the right file, remove the old one, and tell the user.
- Report what changed in 2-4 lines (or per file in directory mode).

## Step 1: Categorize by method

- `GET` -> **options** (for `useQuery`), file `<prefix>-options.ts`
- Anything else (`POST`, `PUT`, `PATCH`, `DELETE`, ...) -> **api** (for `useMutation`), file `<prefix>-api.ts`

`<prefix>` is a short, descriptive kebab-case name derived from the endpoint and action, e.g. `GET /ops-admin/all` -> `get-all-operations-admin-users-options.ts`; `PUT /batch/:id` -> `update-batch-details-api.ts`.
Exported names mirror the file: `getAllNotesOptions`, `assignDeanInstituteApi`.

## Step 2: Pick the ofetch client

- The env base URL already covers everything up to the version (`/v1`, `/v2`, ...). Do not include the version in the path.
- The endpoint prefix after the version selects the client. Example: `/super-admin/...` -> the super-admin client; the path passed to ofetch drops that prefix (`/ops-admin/all`).
- Authenticated endpoint -> the normal client (e.g. `superAdminV1Api`). Unauthenticated endpoint -> the public client (e.g. `superAdminPublicV1Api`).
- Do not reason about `onResponseError` (it just redirects to login on 401).
- Look at the existing project to find the real client names and import path. Typical locations: `src/api/ofetch-client` (single client) or `src/features/<feature>/api` / `apps/*/src/api` (multiple clients by prefix). Reuse the import alias the project already uses (e.g. `@/api/ofetch-api`). Do not guess names if you can check.

## Step 3: Define `Ctx`

`Ctx` holds up to three keys, mapped from the endpoint:

- `params`: path params
- `query`: query/search params
- `body`: request body (api files only; options never have a body)

Only include keys the endpoint actually uses. If the endpoint has none, drop `Ctx` entirely and take no argument. Mark optional fields with `?` and use literal unions for enums (`'ASC' | 'DESC'`).

## Step 4: Response schema (zod)

- Every response gets a zod schema AND an inferred type.
- Schema location: `src/api-schema/<feature>.ts` (or `apps/*/src/api-schema/`). Shared helpers (e.g. `jsonResponseSchema`, `permissionsSchema`) may live in a common package in a monorepo; import them rather than redefining.
- Build the schema from the response shape provided below the curl. If the file already exists, append; do not overwrite other schemas.
- Naming: `<name>Schema` for schemas, PascalCase for types, with the top-level one named after the action (`getAllOperationsAdminUsersSchema` / `GetAllOperationsAdminUsers`). Export schemas and inferred types.
- If the response shape is missing or ambiguous, say so and ask or flag it. Never invent fields.
- If the response is empty (no data payload), skip this step: no schema, use `EmptyJsonResponse` (see "Empty responses" in Step 5).

```ts
import { jsonResponseSchema } from "@scope/common/api-schema/json-response";
import { z } from "zod";

export const operationsAdminAccountSchema = z.object({
  id: z.string(),
  name: z.string(),
  is_active: z.boolean(),
});

export const getAllOperationsAdminUsersSchema = jsonResponseSchema(
  z.object({ accounts: z.array(operationsAdminAccountSchema) }),
);

export type OperationsAdminAccount = z.infer<
  typeof operationsAdminAccountSchema
>;
export type GetAllOperationsAdminUsers = z.infer<
  typeof getAllOperationsAdminUsersSchema
>;
```

## Step 5: Frame the file

Validation is done inside ofetch with `onResponse`, parsing only when `response.ok`, and the generic is the inferred type.

### Options (GET)

```ts
import { keepPreviousData, queryOptions } from "@tanstack/react-query";
import { superAdminV1Api } from "@/api/ofetch-api";
import { type GetAllNotes, getAllNotesSchema } from "@/api-schema/notes";

/**
 * @curl
 * curl --location 'https://<host>/v1/super-admin/department/{departmentId}/notes/all?search=abc&page_size=10' \
 *   --header 'Cookie: <redacted>'
 */

interface Ctx {
  params: { departmentId: string };
  query: {
    search?: string;
    page_size: number;
  };
}

const fn = async (ctx: Ctx) => {
  return await superAdminV1Api<GetAllNotes>(
    `/department/${ctx.params.departmentId}/notes/all`,
    {
      query: ctx.query,
      onResponse: ({ response }) => {
        if (response.ok) getAllNotesSchema.parse(response._data);
      },
    },
  );
};

export const getAllNotesOptions = (ctx: Ctx) => {
  return queryOptions({
    queryKey: ["super-admin", "notes", "get-all-notes", ctx],
    queryFn: () => fn(ctx),
    placeholderData: keepPreviousData, // paginated/filtered lists only
  });
};
```

Rules:

- Query key: `[<client scope>, <feature>, <action>, ctx]`. Include `ctx` whenever it exists so params/query changes refetch. With no ctx, the key is just the static segments.
- Use `placeholderData: keepPreviousData` only for paginated, searchable, or filtered lists.
- No body, ever.
- With no ctx: `export const getAllUsersOptions = () => queryOptions({ ..., queryFn: () => fn() })`.

### API (non-GET)

```ts
import { superAdminV1Api } from "@/api/ofetch-api";
import {
  type UpdateBatchDetails,
  updateBatchDetailsSchema,
} from "@/api-schema/batch";

/**
 * @curl
 * curl --location --request PUT 'https://<host>/v1/super-admin/batch/{batch_id}' \
 *   --header 'Content-Type: application/json' \
 *   --header 'Cookie: <redacted>' \
 *   --data '{ "name": "Batch A", "year": 2026 }'
 */

interface Ctx {
  params: { batch_id: string };
  body: { name?: string; year?: number };
}

export const updateBatchDetailsApi = async (ctx: Ctx) => {
  return await superAdminV1Api<UpdateBatchDetails>(
    `/batch/${ctx.params.batch_id}`,
    {
      method: "PUT",
      body: ctx.body,
      onResponse: ({ response }) => {
        if (response.ok) updateBatchDetailsSchema.parse(response._data);
      },
    },
  );
};
```

Rules:

- Always set `method`. Pass `body: ctx.body` only if the endpoint has a body; pass `query: ctx.query` only if it has query params.
- Plain async function, no `queryOptions`, no query key, no `useMutation`.
- Empty responses (no `data` payload, e.g. assign/delete endpoints) use the shared `EmptyJsonResponse` type, never `jsonResponseSchema(z.null())` or any other per-endpoint null schema. See "Empty responses" below.

#### Empty responses

The common package exports `EmptyJsonResponse` (the envelope `{ status, message, data: undefined }`). Only the type is exported, there is no schema to import. So for an endpoint with an empty response:

- Use `EmptyJsonResponse` as the ofetch generic.
- Do not create a zod schema for it and do not add an `onResponse` parse.
- Import the type from the same shared json-response module the project already uses for `jsonResponseSchema` (do not redefine it).

```ts
import { superAdminV1Api } from "@/api/ofetch-api";
import type { EmptyJsonResponse } from "@scope/common/api-schema/json-response";

/**
 * @curl
 * curl --location --request POST 'https://<host>/v1/super-admin/institute/{institute_id}/faculty/{faculty_id}/assign' \
 *   --header 'Cookie: <redacted>'
 */

interface Ctx {
  params: { institute_id: string; faculty_id: string };
}

export const assignDeanInstituteApi = async (ctx: Ctx) => {
  return await superAdminV1Api<EmptyJsonResponse>(
    `/institute/${ctx.params.institute_id}/faculty/${ctx.params.faculty_id}/assign`,
    {
      method: "POST",
    },
  );
};
```

This applies to options too if a GET ever returns an empty envelope.

## Step 6: Keep the curl as a comment (`@curl`)

Every framed options/api file carries the source curl as a block comment so the user can later paste an updated curl and cross-verify the implementation against it.

- Place it right after the imports and before `interface Ctx` (or before the exported function if there is no `Ctx`), as shown in the examples above.
- Format: a `/** ... */` block whose first line is `@curl`, followed by the curl exactly as given (same method, URL, params, headers that matter, body), one `\` continuation per line, indented with `*`.
- Keep the real method and path intact; the Existing check greps for them, so do not paraphrase, reorder, or abbreviate the URL.
- Redact secrets: replace tokens, cookies, API keys and passwords in headers/body with `<redacted>`. Replace the host with `<host>` only if the original was a real/internal host the user would not want committed; otherwise keep it.
- Do not put the response schema or sample response in the comment. The zod schema is the source of truth for that.
- On Recheck, replace the whole `@curl` block with the new curl.

## Step 7: Output

1. Create the options/api file (with its `@curl` comment) in the right directory (see Step 2) and the schema (see Step 4).
2. Reply in 2-4 lines: files created, method category (options/api), client used (and why: auth vs public), and anything flagged (missing schema fields, guessed client name, assumptions). In directory mode, give the per-file summary described above instead.
3. Stop. Do not wire it into any feature.

## Checklist

- [ ] Input handled: tagged file, text + location, or directory (empty/no-curl files ignored, already-framed curls skipped unless recheck requested)
- [ ] Method -> options or api correct; filename matches `<prefix>-options.ts` / `<prefix>-api.ts`
- [ ] Right ofetch client (prefix + auth/public); path excludes version and client prefix
- [ ] `Ctx` has only the keys the endpoint uses; no `body` in options
- [ ] zod schema + inferred type exist; `onResponse` parses only when `response.ok`
- [ ] Empty responses use `EmptyJsonResponse` (no schema, no `onResponse` parse, never `jsonResponseSchema(z.null())`)
- [ ] Query key (options) includes `ctx` when present
- [ ] `@curl` comment present, secrets redacted, method + path verbatim
- [ ] No feature code written
