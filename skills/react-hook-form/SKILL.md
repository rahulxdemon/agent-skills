---
name: react-hook-form
description:
  Correct React Hook Form usage anywhere in the monorepo — data flow, subscriptions,
  reset, dirty state, number inputs, and controlled-input rules. Load this BEFORE
  writing or modifying ANY form code, adding a field to an existing form, touching
  watch/useWatch/formState/getValues/setValue/reset, wiring a form into a dialog or
  sheet, or building a submit/cancel footer — even when the change looks trivial.
  The codebase contains widespread RHF anti-patterns; without this skill you will
  copy them. For form layout and which components to use, also load
  studio-ui-patterns.
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
noticed and didn't fix. Never add new violations: `react-hook-form/no-use-watch`
is ratcheted in Studio CI — any increase in the warning count fails the build.

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
<FieldGroup>
  <Controller
    control={form.control}
    name="name"
    render={({ field, fieldState }) => (
      <Field data-invalid={fieldState.invalid}>
        <FieldLabel htmlFor="name">Full Name</FieldLabel>
        <Input {...field} id="name" aria-invalid={fieldState.invalid} />
        {fieldState.error && <FieldError errors={[fieldState.error]} />}
      </Field>
    )}
  />
</FieldGroup>
```

This per-field `fieldState` from the `Controller` render prop is a natural fit
for the "one read path per value" and "consume what you subscribed to" rules
above — there's no separate `formState` destructure to drift out of sync,
because the error for a field lives right next to its `field` in the same
render prop.

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

zod schema → `z.infer` type → `useForm` with `zodResolver` and **complete**
`defaultValues` → `FieldGroup` + `Controller` + `Field`/`FieldLabel`/`FieldError`
above → primitive from `ui`.
Layout/container choices (Card vs Sheet, `layout=` variants) are covered by the
`studio-ui-patterns` skill and the demos in
`apps/design-system/registry/default/example/`
(`form-patterns-pagelayout.tsx`, `form-patterns-sidepanel.tsx`) — check them
before inventing structure.

```tsx
// Module level — static references, not recreated on every render
const FORM_ID = 'pool-config-form'

const FormSchema = z.object({
  name: z.string().min(1, 'Name is required'),
  maxConnections: z
    .union([z.literal(''), z.coerce.number().gte(1, 'Must be at least 1')])
    .refine((v) => v !== '', 'Max connections is required'),
})
type FormValues = z.infer<typeof FormSchema>

const defaultValues: FormValues = { name: '', maxConnections: '' }

// Inside the component
const form = useForm<FormValues>({
  resolver: zodResolver(FormSchema),
  defaultValues,
})

<Controller
  control={form.control}
  name="name"
  render={({ field, fieldState }) => (
    <Field data-invalid={fieldState.invalid}>
      <FieldLabel htmlFor="name">Name</FieldLabel>
      <Input {...field} id="name" aria-invalid={fieldState.invalid} />
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
module-level `FORM_ID` via `form={FORM_ID}` on the button. A module-level id is
only safe for singleton forms — if the component can mount more than once at a
time, duplicate ids make external buttons submit the first matching form, so
mint a per-instance id with `useId()` (or an incoming `index`/`id` prop, as
list-row dialogs already do) and share it between the `<form>` and its buttons.
When a form/dialog is rendered once per list row, suffix every element's `id`
and `data-testid` with that same per-instance key (e.g.
`` `edit-faculty-name-${index}` ``) — this is required for e2e tests to target
the right instance and should be treated as part of "the canonical form," not
an optional extra.

## defaultValues, server data, and reset

- **Provide a complete `defaultValues` object — every field, no `undefined`.**
  `isDirty`, `dirtyFields`, and Cancel-reset all compare against it; a missing or
  `undefined` default breaks all three, and `undefined` also makes React treat the
  input as uncontrolled (see below).
- **Form populated from an API? Use the `values` option, not a hand-rolled
  effect.** `values` reacts to the query resolving and resets the form for you;
  computing `defaultValues` from a query that may not have loaded freezes whatever
  happened to be in cache at mount. Add
  `resetOptions: { keepDirtyValues: true }` when a background refetch must not
  clobber the user's in-progress edits. `keepDirtyValues` preserves whatever is
  in `formState.dirtyFields`; typed edits and `setValue(…, { shouldDirty: true })`
  populate it regardless of subscriptions, but `useFieldArray` operations
  (append/remove/move) only mark fields dirty while `dirtyFields` or `isDirty`
  is subscribed — so a form that combines `keepDirtyValues` with a field array
  must read one of them in the owner, or the next refetch will discard array
  edits. (Good examples:
  `components/interfaces/Settings/Database/ConnectionLogging.tsx`,
  `components/interfaces/Storage/EditBucketModal.tsx`.)
- **After a successful mutation, re-baseline the form** in `onSuccess` so the
  saved state becomes the new baseline (`isDirty` returns to false, Cancel now
  reverts to the saved values). Prefer what the server actually persisted: if the
  form uses `values` and the mutation invalidates the query, the refetch handles
  this for you; if the mutation returns the updated resource, `reset(response)`.
  `reset(submittedValues)` is the fallback for APIs that store exactly what was
  sent — if the server normalizes or fills values, it baselines the form to data
  that was never saved. A bare `reset()` reverts to the _previous_ defaults —
  wrong after a save.
- Cancel buttons call `form.reset()`. This only visually restores fields whose
  values round-trip through defined, controlled values — which is why the null
  rules below matter.
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
`z.union([z.literal(''), z.coerce.number()...]).refine((v) => v !== '', '…')`
with a plain `<Input {...field} type="number" />`.

If you instead wire `onChange` through `e.target.valueAsNumber` (or
`valueAsNumber: true`), an empty or partially-typed input produces `NaN`, which
lands in form state and propagates into every calculation, price preview, and
`value` attribute downstream. Guard it with the **same empty sentinel the
field's schema declares** — with the `''`-union schema above:
`field.onChange(Number.isNaN(e.target.valueAsNumber) ? '' : e.target.valueAsNumber)`.
Never let `NaN` into form state.

A nullable API field (`null` = "unset", e.g. a platform default applies)
doesn't change the in-form sentinel — keep `''` inside the form and convert at
the boundaries:

```tsx
// inbound: null → '' when building defaults/values
values: { growthPercent: data.growth_percent ?? '' },
// schema: '' stays the in-form sentinel, zod coerces real input
growthPercent: z.union([z.literal(''), z.coerce.number().gte(10).lte(100)]),
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
`getValues()`. Mutations follow Studio conventions: `onSuccess` → `toast.success`
→ `reset(values)` (or query invalidation when using `values:`), `onError` →
`toast.error`; pass the mutation's `isPending` to the button's `loading`/`disabled`
prop, combined with `!form.formState.isDirty` for edit forms (see above). Default
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
