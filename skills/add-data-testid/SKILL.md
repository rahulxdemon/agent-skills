---
name: add-data-testid
description: Rules for adding data-testid attributes to HTML/JSX elements so the testing team can target them with Playwright. Use this skill whenever you create or modify any React/JSX/TSX component, page, form, list, card, dialog, table, or other UI markup, even if the user doesn't mention testing or test ids. Also use it when the user asks to "add test ids", "make this testable", or "add data-testid". This skill only covers adding the attribute. Never write test cases, test scripts, or test docs.
---

# data-testid Skill

Our testing team writes all tests with **Playwright**. Playwright targets elements through `data-testid` (its standard test id attribute), so every meaningful element you write or touch in HTML/JSX must carry a stable, predictable, unique `data-testid`.

Your only job here is to add the attribute. The testing team handles everything else.

## Rules

### 1. Add `data-testid` to the elements

Add `data-testid` to every element a tester would need to find, read, or interact with: inputs, buttons, links, headings, badges, status text, counts, error and empty states, dialogs, list items, and any element that displays a dynamic value.

### 2. Never add or edit `id` attributes

Do not add `id` as a testing hook. Playwright supports `data-testid` natively and it is the standard. Do not add `id` because "some testing libraries might not support `data-testid`". Our testers use Playwright only.

- **Do not add new `id` attributes** to any element, including form elements (`input`, `select`, `textarea`, `label` with `htmlFor`, and so on) and any other element.
- **Do not edit, rename, or remove existing `id` attributes** on form elements or any other elements. Leave them exactly as they are, even if they look redundant or inconsistent. Other code (labels, ARIA attributes, CSS, scripts) may depend on them.
- Add `data-testid` alongside whatever `id` is already there.
- The `data-testid` does **not** need to match the existing `id`. If an element already has an `id`, write the `data-testid` as a better, feature-based version following the naming convention below, instead of copying the `id`. The two values are independent, but the `id` itself must never change.

### 3. Never use React's `useId` for uniqueness

Do not use the `useId` hook to generate test ids or to achieve uniqueness. Build the id from the **feature/page and the element's role**, so it is readable and predictable.

- Good: `data-testid='partners-page-user-search'`
- Good: `data-testid='partners-page-refresh-partners-btn'`
- Bad: `data-testid={useId()}`

Naming convention: lowercase kebab-case, `<feature-or-page>-<element-purpose>[-<type>]`, for example `partners-page-user-search` or `partner-${index}-name`.

### 4. Put the attribute as deep as possible, on the element that holds the value

Place `data-testid` on the innermost element that directly contains the value, not on a parent wrapper. If the value is bare text next to an icon or other siblings, wrap the value in its own `<span>` and put the attribute on that span. This lets the tester read the value directly (for example with `toHaveText`) without also matching icons or surrounding text.

Wrong, the attribute is on the parent, so the tester gets the icon and extra content too:

```tsx
<span
  className="inline-flex items-center gap-1"
  data-testid={`partner-${index}-email`}
>
  <Mail className="size-3" /> {partner.contact_email}
</span>
```

Right, the attribute sits directly on the value:

```tsx
<span className="inline-flex items-center gap-1">
  <Mail className="size-3" />{" "}
  <span data-testid={`partner-${index}-email`}>{partner.contact_email}</span>
</span>
```

### 5. Lists rendered with `.map()`: use the array index

Elements rendered inside `.map()` would all share one `data-testid`, which breaks the rule that every `data-testid` on a page is unique. To fix this:

- Take the index from the map callback: `.map((item, index) => ...)`.
- Include that index in the `data-testid`, for example `` `partner-${index}-name` ``.
- **Do not** use `item.id` or any other property of the item. Those values are not predictable and break the testing team's scripts. The array index starts at 0, so testers can predict it.
- Pass the index down to child components as a prop (`index`) so they can build their own ids. For child components that render further nested test ids or reusable pieces, pass a prefix prop such as `idPrefix={`partner-${index}`}`. (For dialogs, sheets, and alert dialogs, only the trigger uses this prefix. See rule 6.)
- For nested maps, use a distinct index name for each level (`index`, `adminIdx`) and include all of them: `` `partner-${index}-admin-${adminIdx}-name` ``.

Keep `key` as it is for React's reconciliation (for example `key={partner.id}`). The index rule applies only to `data-testid`, not to `key`.

### 6. Dialog, AlertDialog, and Sheet: index only on the trigger, not on the content

Only one dialog, alert dialog, or sheet can be open at a time, and their content is mounted in the DOM only while open. This changes how uniqueness works:

- **The trigger** (the button or element that opens the dialog/sheet) is always in the DOM, once per list item. It **must** be unique, so include the index: `` `partner-${index}-toggle-status-trigger` ``.
- **Elements inside the content** do not _need_ the index, because only one dialog is open at a time. By default, give generic elements (title, description, form fields, confirm/cancel/submit buttons, error messages) plain feature-based ids such as `change-status-confirm-btn`. This keeps the testing team's scripts simpler.
- **Exception, decided per feature:** keep the `idPrefix` on content elements that _identify which item the dialog belongs to_, such as the item's name or code shown in the dialog, and the content container when it helps. This lets the tester cross-verify that the dialog shows the same item as the card they clicked (for example, the partner name in the dialog matches `partner-${index}-name` on the card), proving the right dialog was triggered and not another one.
- Use judgment per feature: generic controls are index-free, identifying elements may keep the prefix. When unsure, keep the prefix on the element that shows the item's identity and leave the rest index-free.
- This applies to every dialog-like overlay (Dialog, AlertDialog, Sheet). It does not apply to regular inline content, which is always in the DOM and still follows the index rule in rule 5.

Example: `PartnerCard` passes `idPrefix={`partner-${index}`}` to this dialog. The trigger uses the prefix. Inside `AlertDialogContent`, the generic elements (title, description, cancel, confirm) are index-free, while the content container and the partner name keep the prefix so the tester can cross-check the dialog's partner name against the card's `partner-${index}-name`.

```tsx
interface TogglePartnerStatusDialogProps {
  idPrefix: string;
  partner: Partner;
}

export function TogglePartnerStatusDialog({
  idPrefix,
  partner,
}: TogglePartnerStatusDialogProps) {
  const [open, setOpen] = useState(false);
  const queryClient = useQueryClient();
  const isDeactivating = partner.is_active;
  const nextStatus = isDeactivating ? "inactive" : "active";
  const mutation = useMutation({
    mutationFn: togglePartnerStatusApi,
    onSuccess: () => {
      queryClient.invalidateQueries({
        queryKey: ["super-admin", "partners", "all"],
      });
      toast.success(`${partner.name} is now ${nextStatus}`);
      setOpen(false);
    },
    onError: (err: FetchError<JsonResponse>) => {
      toast.error(
        err?.data?.message ?? err.message ?? "Unable to update partner status",
      );
    },
  });

  return (
    <AlertDialog open={open} onOpenChange={setOpen}>
      <AlertDialogTrigger asChild>
        {/* Trigger is always in the DOM (once per partner), so it carries the index prefix */}
        <Button
          type="button"
          size="icon-sm"
          variant="ghost"
          aria-label={`Change ${partner.name}'s status`}
          title={`Change ${partner.name}'s status`}
          data-testid={`${idPrefix}-change-status-btn`}
        >
          <UserRoundKey />
        </Button>
      </AlertDialogTrigger>
      {/* Content is mounted only while open (one at a time). Generic elements are index-free; identifying elements (container, partner name) keep the prefix for cross-verification with the card */}
      <AlertDialogContent
        data-testid={`${idPrefix}-alert-dialog-content`}
        data-current-partner-status={partner.is_active ? "active" : "inactive"}
      >
        <AlertDialogHeader>
          <AlertDialogTitle data-testid="change-status-alert-title">
            {isDeactivating
              ? "Pause partner access?"
              : "Enable partner access?"}
          </AlertDialogTitle>
          <AlertDialogDescription data-testid="change-status-alert-description">
            {isDeactivating ? (
              <>
                <span
                  data-testid={`${idPrefix}-change-status-alert-partner-name`}
                >
                  {partner.name}
                </span>{" "}
                will become inactive immediately. You can reactivate the partner
                at any time.
              </>
            ) : (
              <>
                <span
                  data-testid={`${idPrefix}-change-status-alert-partner-name`}
                >
                  {partner.name}
                </span>{" "}
                will regain access to its assigned operations admins.
              </>
            )}
          </AlertDialogDescription>
        </AlertDialogHeader>
        <AlertDialogFooter>
          <AlertDialogCancel
            disabled={mutation.isPending}
            data-testid="change-status-alert-cancel-btn"
          >
            Cancel
          </AlertDialogCancel>
          <SubmitButton
            type="button"
            variant={isDeactivating ? "destructive" : "default"}
            onClick={() =>
              mutation.mutate({ params: { partnerId: partner.id } })
            }
            isSubmitting={mutation.isPending}
            data-testid="change-status-confirm-btn"
          >
            {isDeactivating ? "Pause Access" : "Enable Access"}
          </SubmitButton>
        </AlertDialogFooter>
      </AlertDialogContent>
    </AlertDialog>
  );
}
```

### 7. Do not write tests or test docs

Do not write test cases, test scripts, Playwright specs, test plans, or testing documentation. Only add the `data-testid` attributes. Do not suggest or generate tests unless the user explicitly asks in that conversation.

## Full example

A list component passes `index` to each card, and the card builds all of its ids from it, including nested lists:

```tsx
export function PartnersList({ partners }: PartnersListProps) {
  const [query, setQuery] = useState("");
  const filteredPartners = useMemo(() => {
    const normalizedQuery = query.trim().toLocaleLowerCase();

    if (!normalizedQuery) return partners;

    return partners.filter((partner) =>
      [partner.name, partner.code].some((value) =>
        value.toLocaleLowerCase().includes(normalizedQuery),
      ),
    );
  }, [partners, query]);

  return (
    <div className="space-y-4">
      <div className="flex items-center justify-between gap-3">
        <InputGroup className="max-w-md bg-card">
          <DebouncedInput
            group
            name="users-search"
            data-testid="partners-page-user-search"
            value={query}
            onChange={(value) => setQuery(String(value))}
            placeholder="Search partners by name or code..."
          />
          <InputGroupAddon>
            <SearchIcon />
          </InputGroupAddon>
        </InputGroup>
        <RefreshQueryButton
          queryKey={["super-admin", "partners", "all"]}
          variant={"outline"}
          data-testid="partners-page-refresh-partners-btn"
        />
      </div>
      {filteredPartners.length === 0 ? (
        <div className="rounded-xl border bg-card p-6 text-center text-sm text-muted-foreground">
          No partners match &quot;{query}&quot;.
        </div>
      ) : (
        <div className="space-y-4">
          {filteredPartners.map((partner, index) => (
            <PartnerCard key={partner.id} index={index} partner={partner} />
          ))}
        </div>
      )}
    </div>
  );
}

interface PartnerCardProps {
  index: number;
  partner: Partner;
}

export function PartnerCard({ index, partner }: PartnerCardProps) {
  const adminCount = partner.operations_admins.length;

  return (
    <Card className={!partner.is_active ? "opacity-70" : undefined}>
      <CardHeader className="gap-3 sm:flex-row sm:items-start">
        <div className="flex items-center justify-between gap-4">
          <div
            className="flex size-9 shrink-0 items-center justify-center rounded-lg bg-primary text-sm font-semibold text-primary-foreground"
            style={{
              backgroundColor: partner.brand_accent_color ?? "var(--muted)",
            }}
            aria-hidden="true"
            data-testid={`partner-${index}-logo-initials`}
          >
            {partner.logo_initials ?? partner.name.slice(0, 2).toUpperCase()}
          </div>
          <div className="flex flex-wrap items-center gap-2">
            <TogglePartnerStatusDialog
              idPrefix={`partner-${index}`}
              partner={partner}
            />
            <EditPartnerButton partner={partner} />
            <AssignPartnerAdminButton partner={partner} />
          </div>
        </div>
        <div className="min-w-0 flex-1 space-y-1">
          <div className="flex flex-wrap items-center gap-2">
            <h2 className="font-semibold" data-testid={`partner-${index}-name`}>
              {partner.name}
            </h2>
            <Badge variant="secondary">
              Code •{" "}
              <span data-testid={`partner-${index}-code`}>{partner.code}</span>
            </Badge>
            <Badge
              variant={partner.is_active ? "default" : "outline"}
              data-testid={`partner-${index}-status`}
            >
              {partner.is_active ? "Active" : "Inactive"}
            </Badge>
          </div>
          <div className="flex flex-wrap gap-x-4 gap-y-1 text-xs text-muted-foreground pt-1">
            <span className="inline-flex items-center gap-1">
              <Mail className="size-3" />{" "}
              <span data-testid={`partner-${index}-email`}>
                {partner.contact_email}
              </span>
            </span>
            <span className="inline-flex items-center gap-1">
              <Globe className="size-3" />{" "}
              <span
                data-testid={`partner-${index}-website`}
                data-has-website={Boolean(partner.website)}
              >
                {partner.website ?? "No website"}
              </span>
            </span>
            <span className="inline-flex items-center gap-1">
              <Users className="size-3" />{" "}
              <span data-testid={`partner-${index}-assigned-admins-count`}>
                {adminCount}
              </span>{" "}
              admin
              {adminCount === 1 ? "" : "s"} assigned
            </span>
          </div>
        </div>
      </CardHeader>
      <CardContent>
        <div className="divide-y overflow-hidden rounded-lg border">
          {adminCount === 0 ? (
            <div
              className="flex items-center gap-2 p-3 text-sm text-muted-foreground"
              data-testid={`partner-${index}-no-assigned-admins`}
            >
              <Building2 className="size-4" /> No admins assigned yet.
            </div>
          ) : (
            partner.operations_admins.map((admin, adminIdx) => (
              <div key={admin.id} className="p-3">
                <div className="flex flex-wrap items-center gap-3 pb-2.5">
                  <div className="min-w-0 flex-1">
                    <div className="flex flex-wrap items-center gap-2 font-medium text-sm">
                      <span
                        data-testid={`partner-${index}-admin-${adminIdx}-name`}
                      >
                        {admin.name}
                      </span>
                      <span className="text-sm text-muted-foreground">•</span>
                      <span
                        className="text-xs font-normal text-muted-foreground"
                        data-testid={`partner-${index}-admin-${adminIdx}-designation`}
                      >
                        {admin.designation}
                      </span>
                      <Badge
                        variant={admin.is_active ? "default" : "outline"}
                        data-testid={`partner-${index}-admin-${adminIdx}-status`}
                      >
                        {admin.is_active ? "Active" : "Inactive"}
                      </Badge>
                    </div>
                    <div className="text-xs text-muted-foreground pt-1 flex items-center justify-start gap-1">
                      <Mail className="size-3" />{" "}
                      <span
                        data-testid={`partner-${index}-admin-${adminIdx}-email`}
                      >
                        {admin.email}
                      </span>
                    </div>
                  </div>
                  <EditPartnerOpsAdminPermissionsSheet
                    adminId={admin.id}
                    adminName={admin.name}
                    partnerId={partner.id}
                    partnerName={partner.name}
                    permissions={admin.permissions}
                  />
                  <UnassignOpsAdminFromPartnerDialog
                    adminId={admin.id}
                    adminName={admin.name}
                    partnerId={partner.id}
                    partnerName={partner.name}
                  />
                </div>
                <PermissionSummary
                  idPrefix={`partner-${index}-admin-${adminIdx}-permissions`}
                  permissions={admin.permissions}
                />
              </div>
            ))
          )}
        </div>
      </CardContent>
    </Card>
  );
}
```

## Checklist before finishing

- Every interactive or value-bearing element has a `data-testid`.
- No `id` attributes were added, and no existing `id` attributes (on form elements or anywhere else) were edited or removed. `useId` is not used.
- Each `data-testid` is built from the feature and element purpose, and is unique on the page. It does not have to equal the element's existing `id`, and the `id` is left untouched.
- Every `data-testid` inside a `.map()` includes the map index, never `item.id` or another property.
- Dialog/AlertDialog/Sheet triggers include the list index. Inside their content, generic elements are index-free, while identifying elements (such as the item's name) may keep the prefix for cross-verification with the card.
- The attribute sits on the innermost element holding the value, not on a parent wrapper.
- No test files, test cases, or testing docs were created.
