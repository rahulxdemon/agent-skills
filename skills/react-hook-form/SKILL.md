---
name: react-hook-form
description:
  Correct React Hook Form usage anywhere in the monorepo — data flow, subscriptions,
  reset, dirty state, number inputs, controlled-input rules, and forms inside
  dialogs/sheets. Load this BEFORE writing or modifying ANY form code, adding a
  field to an existing form, touching watch/useWatch/formState/getValues/setValue/reset,
  wiring a form into a dialog or sheet, or building a submit/cancel footer — even
  when the change looks trivial. The codebase contains widespread RHF anti-patterns;
  without this skill you will copy them. For form layout and which components to use,
  also load studio-ui-patterns.
---

# React Hook Form

How to write forms that stay correct as they grow. The existing codebase is **not**
a safe reference: `form.watch()` off prop-drilled form objects, watches,
unguarded `valueAsNumber`, `?? undefined` controlled values, and raw `register()`
spread onto plain `<input>`s are all common in older code and all wrong. Follow
this skill, not the neighboring file.

**Policy — fix what you touch.** New code must follow these rules. When you modify
existing form code, upgrade the specific fields/hooks/components you're editing to
match (e.g. a component you touch that calls `form.watch` gets converted to
`useWatch`). Leave untouched code alone, but tell the user about anti-patterns you
noticed and didn't fix.

## Forms in a dialog or sheet: always their own component

**Any form rendered inside a `Dialog`, `Sheet`, `Drawer`, `Popover`, or similar
overlay — create _and_ edit — lives in its own file/component, and `useForm` is
called inside that component, never in the component that owns the overlay.**

Why: the overlay's content unmounts when it closes. If `useForm` (and
`useMutation`) live in a child component rendered _inside_ the content, closing
the overlay destroys the form instance, so reopening it starts from a clean slate:
no leftover typed values, validation errors, touched/dirty flags, or stale
mutation error/pending state. If `useForm` lives in the overlay owner instead,
that component stays mounted while the overlay is closed, and the half-typed
state (or the previous submit's errors) is still there on reopen — Cancel,
`onOpenChange`, and `reset()` calls scattered around to "clean up" are all
symptoms of getting this wrong.

Rules:

- **Two files, colocated:** `<name>-form.tsx` (owns `useForm`, `useMutation`,
  fields, the submit/cancel footer) and `<name>-button.tsx` / `<name>-dialog.tsx`
  / `<name>-sheet.tsx` (owns only `open` state, the trigger, and the overlay
  chrome — title, description).
- **The overlay owner holds no form state.** No `useForm`, no `useWatch`, no
  `reset()`, no `useEffect` that resets on `open` change. Its only form-related
  job is passing `onSuccess` / `onCancel` callbacks that close the overlay.
- **Render the form directly inside the overlay's content component**
  (`DialogContent`, `SheetContent`, …), so it mounts on open and unmounts on
  close. Don't hoist it above the content, and don't pass `forceMount` /
  `keepMounted` (or any equivalent "keep children mounted" prop) on the content.
- **Edit forms:** pass the record as a prop and feed it to `values:` inside the
  form component (see "defaultValues, server data, and reset"). Because the form
  remounts on every open, it re-initializes from the latest record each time —
  no reset-on-open effect needed.
- **Put the submit/cancel footer inside the form component**, not in the
  overlay owner. That keeps `isDirty`, `isPending`, and the submit handler next
  to the `useForm` call, and avoids the `form={FORM_ID}` external-button
  workaround entirely. Only fall back to `form={id}` when the design system's
  overlay component forces the footer outside the content slot — and then the
  form must still be mounted inside the content.
- **If the overlay implementation keeps its children mounted** while closed
  (a custom wrapper, `forceMount`, an always-rendered side panel that is only
  hidden with CSS), remount the form explicitly with a `key` that changes on
  open — e.g. an `openCount`/`formKey` bumped in `onOpenChange`. Prefer fixing
  the wrapper to unmount; use `key` only as a last resort.

```tsx
// ❌ useForm in the overlay owner — state survives close/reopen
export function EditPartnerButton({ partner }: Props) {
  const [open, setOpen] = useState(false);
  const form = useForm<FormValues>({
    resolver: zodResolver(FormSchema),
    values: toValues(partner),
  });
  //    ^ this component stays mounted while the dialog is closed
  return (
    <Dialog open={open} onOpenChange={setOpen}>
      <DialogContent>
        <Controller control={form.control} name="name" render={/* … */} />
      </DialogContent>
    </Dialog>
  );
}
```

```tsx
// ✅ edit-partner-button.tsx — owns only open state + overlay chrome
export function EditPartnerButton({
  idPrefix,
  partner,
  trigger,
}: EditPartnerButtonProps) {
  const [open, setOpen] = useState(false);

  return (
    <Dialog open={open} onOpenChange={setOpen}>
      <DialogTrigger asChild>{trigger ?? <Button /* … */ />}</DialogTrigger>
      <DialogContent data-testid={`${idPrefix}-edit-dialog-content`}>
        <DialogHeader>
          <DialogTitle>Edit partner details</DialogTitle>
          <DialogDescription>
            Update details for {partner.name}.
          </DialogDescription>
        </DialogHeader>
        {/* mounts on open, unmounts on close → fresh form every time */}
        <EditPartnerForm
          partner={partner}
          onSuccess={() => setOpen(false)}
          onCancel={() => setOpen(false)}
        />
      </DialogContent>
    </Dialog>
  );
}

// ✅ edit-partner-form.tsx — owns useForm, useMutation, fields, footer
export function EditPartnerForm({
  partner,
  onSuccess,
  onCancel,
}: EditPartnerFormProps) {
  const id = useId();
  const form = useForm<FormValues>({
    resolver: zodResolver(FormSchema),
    defaultValues,
    values: toValues(partner),
  });
  const { isDirty } = form.formState;
  // …useMutation, fields, Cancel / Save footer
}
```

The same applies to **create** forms: `CreatePartnerDialog` owns `open` and the
trigger, `CreatePartnerForm` owns `useForm` with static `defaultValues`. Closing
and reopening yields an empty form with no stale errors.

Note that `onSuccess` / `onCancel` only close the overlay — they don't call
`form.reset()`. Unmounting already discards the form, so a reset-on-close is
redundant. (Re-baselining with `reset(response)` after a successful save, per the
section below, is still correct for forms that stay open after saving.)

When touching an existing dialog/sheet that calls `useForm` in the overlay owner,
extract the form into its own file as part of the change (per the fix-what-you-touch
policy).

## Always wire fields with `Controller` — never bare `register`, never `FormField`

Every field in this codebase is a controlled component wired through RHF's
`Controller` render prop directly. **Never spread `register('name')` onto a raw
`<input>`** as a shortcut — it bypasses the `field`/`fieldState` contract every
other rule in this skill depends on (empty-string sentinels, `fieldState.invalid`,
scoped re-renders) and is invisible to `no-use-watch`/dirty-state tooling built
around `Controller`. If you see bare `register(...)` in a diff you're touching,
convert it.

This also means: don't reach for the shadcn `Form`/`FormField`/`FormItem`/
`FormControl`/`FormLabel`/`FormMessage` context wrapper, even though it also
wraps `Controller` internally. This codebase standardizes on explicit
`Controller` + `Field`/`FieldGroup` (below) — if you find `Form`/`FormField` in
a file you're touching, convert it to match.

```tsx
// ❌ never do this
<input {...form.register('email')} />

// ✅ minimum acceptable wiring
<Controller
  control={form.control}
  name="email"
  render={({ field, fieldState }) => (
    <Input {...field} aria-invalid={fieldState.invalid} />
  )}
/>
```

## Field wiring: `Field` / `FieldGroup` + explicit `Controller`

Every field is wrapped in `<Field>` inside a `<FieldGroup>`, with an explicit
`Controller` per field — no `FormProvider`/context. `fieldState` comes straight
off the render prop and is passed to `FieldError` explicitly, right next to the
`field` it belongs to:

```tsx
function MyForm() {
  const id = useId();
  const form = useForm<FormValues>({
    resolver: zodResolver(FormSchema),
    defaultValues,
  });

  return (
    <FieldGroup>
      <Controller
        control={form.control}
        name="name"
        render={({ field, fieldState }) => (
          <Field data-invalid={fieldState.invalid}>
            <FieldLabel htmlFor={`${id}-${field.name}`}>Full Name</FieldLabel>
            <Input
              {...field}
              id={`${id}-${field.name}`}
              aria-invalid={fieldState.invalid}
            />
            {fieldState.error && <FieldError errors={[fieldState.error]} />}
          </Field>
        )}
      />
    </FieldGroup>
  );
}
```

This per-field `fieldState` from the `Controller` render prop is a natural fit
for the "one read path per value" and "consume what you subscribed to" rules
above — there's no separate `formState` destructure to drift out of sync,
because the error for a field lives right next to its `field` in the same
render prop.

**Always build `id`/`htmlFor` from `useId()` + `field.name` — never a bare
string literal.** Call `useId()` once per form instance (not per field) and
compose each field's id as `` `${id}-${field.name}` ``. `field.name` guarantees
uniqueness across fields in the same form; `useId()` guarantees uniqueness
across mounts of the _same_ form (two open edit dialogs, a repeated form in a
list). A literal `id="name"` works until the form renders twice, at which point
both instances share a DOM id, `htmlFor` starts pointing at whichever input
rendered first, and clicking one label can focus the wrong field. This is the
same class of bug the `FORM_ID` guidance below addresses, and can reuse the
same `useId()` call.

## Reading values, by location

- **In the component that owns `useForm`:** destructure `formState`; prefer
  `useWatch` over `form.watch` even here (the `no-use-watch` rule flags every
  `watch`, and `useWatch` scopes the re-render if the JSX is later extracted).
- **In any child component or custom hook:** accept `control` (not the whole
  `form`) and use `useWatch({ control, name })` / `useFormState({ control })`.
  There's no `FormProvider`/context in this codebase's pattern, so `control`
  must always be passed explicitly — don't reach for `useFormContext()`.
- **Consume the return value.** Never call a watch for its subscription side
  effect and then read via `getValues()` — the watch list and the read list will
  drift apart (it has already happened; fields silently lost reactivity). The
  value you render must _be_ the value you subscribed to.
- **One read path per value per render.** Mixing `useWatch('x')` on one line and
  `getValues('x')` a few lines later lets the two disagree within a single render.
- **Name what you watch.** `useWatch({ control })` with no `name` re-renders on
  every keystroke in every field. Subscribe to the specific names you use.
- `watch(callback)` is deprecated — use `subscribe()` for render-free listeners,
  and always return its cleanup from `useEffect`.

```tsx
// ❌ common in the codebase — all three subscriptions hoist to the form owner
function Fields({ form }: { form: UseFormReturn<FormValues> }) {
  form.watch(['storageType', 'totalSize'])        // return value discarded
  const { errors } = form.formState               // prop-form formState
  const size = form.getValues('totalSize')        // non-reactive read in render
  ...
}

// ✅ child subscribes for itself and consumes what it watches
function Fields({ control }: { control: Control<FormValues> }) {
  const [storageType, totalSize] = useWatch({ control, name: ['storageType', 'totalSize'] })
  const { errors } = useFormState({ control })
  ...
}
```

## The canonical form

zod schema → `z.input` type (or `z.infer` if the schema has no coerced/
transformed fields — see "Type `FormValues`" under Number inputs) → `useForm`
with `zodResolver` and **complete**
`defaultValues` → `FieldGroup` + `Controller` + `Field`/`FieldLabel`/`FieldError`
above → primitive from `ui`.
Layout/container choices (Card vs Sheet, `layout=` variants) are covered by the
`studio-ui-patterns` skill and the demos in
`apps/design-system/registry/default/example/`
(`form-patterns-pagelayout.tsx`, `form-patterns-sidepanel.tsx`) — check them
before inventing structure. If the form is shown in a dialog or sheet, it is its
own component in its own file (see "Forms in a dialog or sheet" above).

```tsx
// data/validation-limits.ts — module level, exported, reused by schema + UI
export const POOL_VALIDATION = {
  MAX_CONNECTIONS_MIN: 1,
} as const

// Module level — static references, not recreated on every render
const FORM_ID = 'pool-config-form'

const FormSchema = z.object({
  name: z.string().min(1, 'Name is required'),
  maxConnections: z
    .union([
      z.literal(''),
      z.coerce
        .number<number>()
        .gte(
          POOL_VALIDATION.MAX_CONNECTIONS_MIN,
          `Must be at least ${POOL_VALIDATION.MAX_CONNECTIONS_MIN}`,
        ),
    ])
    .refine((v) => v !== '', 'Max connections is required'),
})
// z.input, not z.infer — see "Typing coerced fields" below
type FormValues = z.input<typeof FormSchema>

const defaultValues: FormValues = { name: '', maxConnections: '' }

// Inside the component
const id = useId()
const form = useForm<FormValues>({
  resolver: zodResolver(FormSchema),
  defaultValues,
})

<Controller
  control={form.control}
  name="name"
  render={({ field, fieldState }) => (
    <Field data-invalid={fieldState.invalid}>
      <FieldLabel htmlFor={`${id}-${field.name}`}>Name</FieldLabel>
      <Input {...field} id={`${id}-${field.name}`} aria-invalid={fieldState.invalid} />
      {fieldState.error && <FieldError errors={[fieldState.error]} />}
    </Field>
  )}
/>
```

Define the schema, `type`, static `defaultValues`, and the form's id at module
level, outside the component. Rebuilding them per render is wasted work and
unstable references — RHF reads `defaultValues` only on the first render, but
anything else comparing against these objects sees a fresh identity each time.
When they genuinely depend on runtime data, build the schema with `useMemo` and
feed server-driven defaults through the `values` option (next section) instead
of hoisting.

Submit buttons living outside the `<form>` (sheet/dialog footers) use the same
module-level `FORM_ID` via `form={FORM_ID}` on the button. Prefer putting the
footer inside the form component instead (see "Forms in a dialog or sheet"), which
makes this unnecessary. A module-level id is
only safe for singleton forms — if the component can mount more than once at a
time, duplicate ids make external buttons submit the first matching form, so
mint a per-instance id instead. Reuse the same `useId()` call from the field-id
guidance above for this: `const id = useId()`, then the `<form>`'s own id is
`id` (or `` `${id}-form` `` if you want it visually distinct) and each field's
id is `` `${id}-${field.name}` `` — one call, no separate id scheme to keep in
sync. When a form/dialog is rendered once per list row and you also need
stable per-instance selectors for e2e tests, suffix `data-testid` with an
incoming `index`/record-id prop instead (e.g.
`` `edit-faculty-name-${index}` ``) — that's a testing concern, separate from
`useId()`'s job of keeping `id`/`htmlFor` unique, and both can coexist on the
same element.

For the numeric literals inside a schema like `maxConnections` above, see
**Validation limits: define once, reuse everywhere** below — don't let a bare
number sit only inside `.gte()`/`.lte()`/`.min()`/`.max()`.

## Validation limits: define once, reuse everywhere

Numeric/length limits used in a zod schema (`min`, `max`, `gte`, `lte`, and
similar) belong in a single exported `as const` object per feature — never as
a bare number that appears twice: once inside the validator call and again,
independently, inside its message string.

```tsx
// features/faculty/data/validation-limits.ts
export const FACULTY_VALIDATION = {
  NAME_MIN: 3,
  NAME_MAX: 60,
  DESIGNATION_MIN: 2,
  DESIGNATION_MAX: 20,
} as const;

// schema
const formSchema = z.object({
  name: z
    .string()
    .trim()
    .min(FACULTY_VALIDATION.NAME_MIN, {
      message: `Name must be at least ${FACULTY_VALIDATION.NAME_MIN} characters`,
    })
    .max(FACULTY_VALIDATION.NAME_MAX, {
      message: `Name must be at most ${FACULTY_VALIDATION.NAME_MAX} characters`,
    }),
  designation: z
    .string()
    .trim()
    .min(FACULTY_VALIDATION.DESIGNATION_MIN, {
      message: `Designation must be at least ${FACULTY_VALIDATION.DESIGNATION_MIN} characters`,
    })
    .max(FACULTY_VALIDATION.DESIGNATION_MAX, {
      message: `Designation must be at most ${FACULTY_VALIDATION.DESIGNATION_MAX} characters`,
    }),
});
```

- One object per feature (or a shared one for a cross-feature limit),
  colocated under that feature's `data/`/`types/` folder and **exported** —
  not a `const` scoped to the file that owns the schema. Exporting it lets the
  same numbers drive an input's `maxLength`, a character counter, or a
  backend-parity check without a second copy of the limit appearing there too.
- Name keys `SCREAMING_SNAKE_CASE`, suffixed `_MIN`/`_MAX` (or `_LENGTH`,
  `_COUNT`, etc. for a single-sided limit).
- **Interpolate the same constant into the message — never write the number
  twice.** `min(3, 'must be at least 3 characters')` has the limit in two
  places; change one during a later edit and the validator and the message the
  user reads silently disagree.
- Applies anywhere a schema encodes a business rule as a number: string
  lengths, numeric ranges (`gte`/`lte`, as with `maxConnections` above), array
  lengths (`.min(1, ...)` on a `useFieldArray`), and so on.

## defaultValues, server data, and reset

- **Provide a complete `defaultValues` object — every field, no `undefined`.**
  `isDirty`, `dirtyFields`, and Cancel-reset all compare against it; a missing or
  `undefined` default breaks all three, and `undefined` also makes React treat the
  input as uncontrolled (see below).
- **Form populated from data the parent already has — a query, or a prop like an
  already-fetched row object — use the `values` option, not a hand-rolled
  `useEffect(() => form.reset(...), [deps])`.** `values` reacts automatically
  whenever the referenced fields change and resets the form for you; a
  `useEffect` that calls `reset()` is doing that job manually and worse — it
  re-runs on whatever dependency array you wrote, is easy to under- or
  over-specify (e.g. depending on `form.reset` instead of the source data), and
  is exactly the anti-pattern this section exists to avoid, even when the source
  is a prop (e.g. an edit-dialog opened with a specific record) rather than a
  query directly:

  ```tsx
  // ❌ manual reset-on-prop-change — drop this
  React.useEffect(() => {
    form.reset({ name: faculty.name, designation: faculty.designation });
  }, [faculty.name, faculty.designation, form.reset]);

  // ✅ let RHF react to the prop itself
  const form = useForm<FormValues>({
    resolver: zodResolver(FormSchema),
    values: { name: faculty.name, designation: faculty.designation },
  });
  ```

  Add `resetOptions: { keepDirtyValues: true }` when a background refetch must
  not clobber the user's in-progress edits. `keepDirtyValues` preserves whatever
  is in `formState.dirtyFields`; typed edits and
  `setValue(…, { shouldDirty: true })` populate it regardless of subscriptions,
  but `useFieldArray` operations (append/remove/move) only mark fields dirty
  while `dirtyFields` or `isDirty` is subscribed — so a form that combines
  `keepDirtyValues` with a field array must read one of them in the owner, or
  the next refetch will discard array edits.

- **After a successful mutation, re-baseline the form** in `onSuccess` so the
  saved state becomes the new baseline (`isDirty` returns to false, Cancel now
  reverts to the saved values). Prefer what the server actually persisted: if the
  form uses `values` and the mutation invalidates the query, the refetch handles
  this for you; if the mutation returns the updated resource, `reset(response)`.
  `reset(submittedValues)` is the fallback for APIs that store exactly what was
  sent — if the server normalizes or fills values, it baselines the form to data
  that was never saved. A bare `reset()` reverts to the _previous_ defaults —
  wrong after a save. (Forms in a dialog/sheet that close on success don't need
  this — the form unmounts; see "Forms in a dialog or sheet".)
- Cancel buttons call `form.reset()` in forms that stay mounted (page forms,
  inline cards). In a dialog/sheet form, Cancel just closes the overlay — the
  form unmounts and the next open starts fresh. A page-form Cancel only visually
  restores fields whose values round-trip through defined, controlled values —
  which is why the null rules below matter.
- **Gate the submit button on `isDirty` in edit forms.** Where the form is
  editing an existing record rather than creating a new one, disable Save while
  `!form.formState.isDirty` (in addition to the mutation's `isPending`) so users
  can't submit an unchanged record. Read `isDirty` via a destructured
  `formState` in the owning component, per the destructuring rule below.

## Controlled inputs: never let `value` flip to `undefined`

React decides controlled vs uncontrolled per render from whether `value` is
defined. A field whose value can be `undefined` (or becomes `undefined` on reset)
flips modes: console warnings, and — worse — `reset()` stops clearing the visible
text because React abandoned the DOM value. `value={field.value ?? undefined}` is
a bug, not a fix.

- Text fields: default to `''`, never `null`/`undefined`.
- **Normalize `null` from the API at the form boundary** (`growthPercent ?? ''`
  when building defaults) and convert back on submit (`'' → null`). Do not paper
  over a `null` default with a `placeholder` that looks like a value: the user
  sees "50", the form holds `null`, and every downstream comparison
  (`defaultValues.growthPercent !== watched` → `null !== 50`) reports a permanent
  phantom change while Cancel silently fails to reset the field.
- Selects/radios: default to `''` or a real option value; checkboxes/switches to
  `false`.

## Number inputs

The blessed pattern keeps `''` as the "empty" sentinel so the input stays
controlled, and lets zod coerce on validation (see `maxConnections` above):
`z.union([z.literal(''), z.coerce.number<number>()...]).refine((v) => v !== '', '…')`
with a plain `<Input {...field} type="number" />`.

If you instead wire `onChange` through `e.target.valueAsNumber` (or
`valueAsNumber: true`), an empty or partially-typed input produces `NaN`, which
lands in form state and propagates into every calculation, price preview, and
`value` attribute downstream. Guard it with the **same empty sentinel the
field's schema declares** — with the `''`-union schema above:
`field.onChange(Number.isNaN(e.target.valueAsNumber) ? '' : e.target.valueAsNumber)`.
Never let `NaN` into form state.

### Type coerced fields with `z.coerce.number<number>()`, not bare `z.coerce.number()`

Always give `z.coerce.number()` an explicit type argument —
`z.coerce.number<number>()`. Left bare inside a `z.union([...]).refine(...)`
chain like the sentinel pattern above, TS can fail to narrow the coerced
branch correctly and the field ends up mistyped or `unknown`. The generic
makes the intended type explicit instead of relying on inference through the
union/refine chain.

### Type `FormValues` with `z.input`, not `z.infer`, whenever the schema coerces

`z.infer<typeof Schema>` (an alias for `z.output`) is the shape _after_ zod has
parsed and coerced the data — for the sentinel pattern above that's
`{ age: number }`, never `''`. But the form itself holds pre-parse values: an
empty text input is `''` until the resolver runs. Feeding `z.infer` into
`useForm<FormValues>` makes `defaultValues: { age: '' }` and
`field.onChange('')` type errors, because TypeScript believes `age` is always
a `number`.

Use `z.input<typeof Schema>` for `FormValues` (and for the `onSubmit` handler's
parameter type) whenever any field in the schema is coerced, unioned with a
sentinel, or otherwise transformed — i.e. whenever the raw, in-progress form
value differs from the validated output value:

```tsx
const FormSchema = z.object({
  age: z
    .union([z.literal(""), z.coerce.number<number>().int().gte(0)])
    .refine((v) => v !== "", "Age is required"),
});

// ✅ z.input — matches what the form actually holds before validation
type FormValues = z.input<typeof FormSchema>;

// ❌ z.infer/z.output — says `age: number`, breaks defaultValues: { age: '' }
type FormValues = z.infer<typeof FormSchema>;
```

A schema with no coerced/transformed fields (plain strings, enums, booleans)
has identical `input` and `output` shapes, so `z.infer` is still fine there —
this rule only bites once coercion, `.transform()`, or a sentinel union enters
the schema, which in practice means: default to `z.input` for any form that
has a number field built on this skill's sentinel pattern.

A nullable API field (`null` = "unset", e.g. a platform default applies)
doesn't change the in-form sentinel — keep `''` inside the form and convert at
the boundaries:

```tsx
// inbound: null → '' when building defaults/values
values: { growthPercent: data.growth_percent ?? '' },
// schema: '' stays the in-form sentinel, zod coerces real input
growthPercent: z.union([z.literal(''), z.coerce.number<number>().gte(10).lte(100)]),
// outbound: '' → null in onSubmit
mutate({ growth_percent: values.growthPercent === '' ? null : values.growthPercent })
```

If `null` does end up in form state (some existing forms hold it), keep it out
of both the input and the coercion: render via `value={field.value ?? ''}`, and
don't pass the value through `z.coerce.number()` — `Number(null)` is `0`, so a
nullable field fed into the coercing union silently validates empty as `0`.
Either way it's one sentinel per field, used consistently across defaults,
schema, `onChange`, rendering, and the submit mapping.

## Submit and mutations

`onSubmit` receives validated, typed data — trust it; don't re-read via
`getValues()`. When `FormValues` is typed with `z.input` (see Number inputs
above), the `onSubmit` parameter is still typed as `z.input` — e.g.
`age: '' | number` — even though the resolver has already run and the value is
always a real `number` by the time your handler executes. This is a gap in the
single-generic `useForm<FormValues>` signature, not a runtime bug: trust the
value, but don't be surprised if TS still shows the sentinel type inside
`onSubmit` and requires a narrow (`values.age === '' ? ... : ...`) or a type
assertion before passing it somewhere that expects a plain `number`. Mutations
follow Studio conventions: `onSuccess` → `toast.success`
→ `reset(values)` (or query invalidation when using `values:`), `onError` →
`toast.error`; pass the mutation's `isPending` to the button's `loading`/`disabled`
prop, combined with `!form.formState.isDirty` for edit forms (see above). In a
dialog/sheet form, `onSuccess` calls the `onSuccess` prop to close the overlay
instead of resetting. Default
validation `mode: 'onSubmit'` is right for most forms — pick another mode
deliberately, not by copying.

- `reset(values)` (or query invalidation when using `values:`), `onError` →
  `toast.error`; pass the mutation's `isPending` to the button's `loading` prop.
  Default validation `mode: 'onSubmit'` is right for most forms — pick another mode
  deliberately, not by copying.

## Lint rules in force (Studio)

| Rule                                        | Level            | Meaning                                          |
| ------------------------------------------- | ---------------- | ------------------------------------------------ |
| `react-hook-form/destructuring-formstate`   | error            | destructure `formState`, never hold the object   |
| `react-hook-form/no-access-control`         | error            | don't reach into `control` internals             |
| `react-hook-form/no-nested-object-setvalue` | error            | `setValue('a.b', v)`, not `setValue('a', {b:v})` |
| `react-hook-form/no-use-watch`              | warn (ratcheted) | use `useWatch`, not `watch`                      |
