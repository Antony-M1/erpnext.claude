---
name: override-doctype-class
description: >
  Use this skill when the user wants to override an existing DocType class from Frappe, ERPNext,
  HRMS, Payments, CRM, or any other Frappe app, and keep the override inside their custom app.
  Triggers: any mention of "override doctype", "extend class", "custom validation", "add to
  existing controller", "hook doctype", or when the user wants to add functionality on top of
  existing Frappe/ERPNext behavior without modifying core files. Also trigger when the user
  references override_doctype_class in hooks.py or asks to put custom logic in an overrides/
  folder. Do NOT use for brand-new DocTypes, client-side JS customizations, or fixture-based
  customizations done via the Frappe UI.
tools:
  - Read
  - Write
  - Bash
---

# Override DocType Class — Frappe / ERPNext

## Reference
- Controllers docs: https://docs.frappe.io/framework/user/en/basics/doctypes/controllers
- Hooks docs: https://docs.frappe.io/framework/user/en/python-api/hooks

---

## Workflow

### 1. Read CLAUDE.md first
Custom app name, bench path, and any conventions are defined there. All paths below assume
`{bench}` = bench root and `{custom_app}` = the custom app name found in CLAUDE.md.

### 2. Find the source DocType
Common locations (search under `doctype/` in the relevant app):

| App | Base path |
|-----|-----------|
| frappe | `{bench}/apps/frappe/frappe/` |
| erpnext | `{bench}/apps/erpnext/erpnext/` |
| hrms | `{bench}/apps/hrms/hrms/` |
| crm (Frappe CRM) | `{bench}/apps/crm/crm/fcrm/` |
| payments | `{bench}/apps/payments/payments/` |

Run `find {bench}/apps -type d -name "<doctype_snake_case>"` to locate the exact folder when
unsure. The controller class lives at `<module>/<doctype_snake_case>/<doctype_snake_case>.py`.

### 3. Check hooks.py for existing override conventions
Open `{bench}/apps/{custom_app}/{custom_app}/hooks.py` and look for `override_doctype_class`.
- If a key already exists, follow the same folder structure used by existing entries.
- If none exists yet, default to `{custom_app}/overrides/<doctype_snake_case>.py`.

### 4. Create the override file
File path: `{bench}/apps/{custom_app}/{custom_app}/overrides/<doctype_snake_case>.py`
Class name: `Custom<OriginalClassName>` (prefix `Custom` to the imported class name).

### 5. Register in hooks.py
```python
override_doctype_class = {
    "DocType Name": "{custom_app}.overrides.<doctype_snake_case>.Custom<OriginalClassName>"
}
```
If the dict already exists, add a new key — do not replace the entire dict.

---

## Standard Controller Events (server-side)

| Method | When it fires | Call `super()`? |
|--------|--------------|-----------------|
| `before_insert` | Before a new doc is inserted | If adding to existing logic → yes |
| `after_insert` | After insert, before page reload | If adding to existing logic → yes |
| `validate` | On Save (before DB write) | **Almost always yes** |
| `before_save` | Just before DB write on save | If adding to existing logic → yes |
| `on_update` | After DB write on save | If adding to existing logic → yes |
| `before_submit` | Before docstatus → 1 | If adding to existing logic → yes |
| `on_submit` | After docstatus set to 1 | If adding to existing logic → yes |
| `before_cancel` | Before docstatus → 2 | If adding to existing logic → yes |
| `on_cancel` | After docstatus set to 2 | If adding to existing logic → yes |
| `on_trash` | Before permanent deletion | If adding to existing logic → yes |
| `after_delete` | After deletion | If adding to existing logic → yes |
| `on_update_after_submit` | On amend after submit | If adding to existing logic → yes |
| `before_update_after_submit` | Before amend save | If adding to existing logic → yes |

---

## Rules

1. **Preserve existing behavior by default** — call `super().<method>()` unless the user
   explicitly says to skip the original logic (keyword: *"replace"* or *"ignore existing"*).
2. **Ambiguous workflows** — when unsure whether to chain with `super()`, default to calling it
   and notify the user: *"This also runs the original `{method}` via `super()`."*
3. **Keyword contract**
   - User says **"replace"** or **"ignore existing"** → omit `super()`.
   - User says **"extend"**, **"add"**, or **"also"** → always call `super()`.
4. **Use docstrings**, not inline comments, for explanations. Add inline comments only for
   non-obvious logic.
5. **Name custom helper methods** with a `{custom_app}_` prefix to avoid collisions
   (e.g., `custom_app_validate_unique_mobile_no`).
6. **One file per DocType** — keep overrides flat in the `overrides/` folder; do not nest.

---

## Canonical Example

**Scenario:** Add a unique-mobile-no check to Customer (ERPNext) in a custom app.

**`{bench}/apps/{custom_app}/{custom_app}/hooks.py`** (add/update):
```python
override_doctype_class = {
    "Customer": "{custom_app}.overrides.customer.CustomCustomer"
}
```

**`{bench}/apps/{custom_app}/{custom_app}/overrides/customer.py`**:
```python
import frappe
from erpnext.selling.doctype.customer.customer import Customer


class CustomCustomer(Customer):
    """Custom Customer — adds unique mobile number validation."""

    def validate(self):
        super().validate()
        self.custom_app_validate_unique_mobile_no()

    def custom_app_validate_unique_mobile_no(self):
        """Throw if another Customer already uses the same mobile_no."""
        if not self.mobile_no:
            return

        existing = frappe.db.get_value(
            "Customer",
            {"mobile_no": self.mobile_no, "name": ("!=", self.name)},
            "name",
        )
        if existing:
            frappe.throw(
                frappe._("Mobile No {0} is already linked to customer {1}.").format(
                    frappe.bold(self.mobile_no), frappe.bold(existing)
                ),
                title=frappe._("Duplicate Mobile No"),
            )
```

---

## Common Patterns

### Add extra validation (preserve original)
```python
def validate(self):
    super().validate()
    self.custom_app_extra_check()
```

### Replace a method entirely (skip original)
```python
def get_party_account(self, party_type, party):
    # Custom logic only — original skipped intentionally (user requested "replace")
    ...
```

### Override `on_submit` to send a custom notification
```python
def on_submit(self):
    super().on_submit()
    self.custom_app_notify_team()

def custom_app_notify_team(self):
    """Send notification on submission."""
    ...
```

### Guard with `self.flags` to avoid double-runs
```python
def on_update(self):
    super().on_update()
    if not self.flags.in_insert:
        self.custom_app_sync_external_system()
```

---

## Checklist Before Finishing

- [ ] Correct import path verified by reading the source file.
- [ ] `override_doctype_class` entry added/updated in `hooks.py`.
- [ ] Class name follows `Custom<OriginalClass>` convention.
- [ ] Helper methods prefixed with `{custom_app}_`.
- [ ] `super()` called where appropriate (default: yes).
- [ ] User notified if `super()` behavior was assumed.
- [ ] No unnecessary inline comments; docstrings used instead.
