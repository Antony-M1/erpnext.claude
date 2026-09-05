---
name: role-permission-manager
description: Use this skill whenever the user is working with Frappe or ERPNext role-based permissions — assigning Roles to a User (Has Role), creating or editing DocType permission rules (Custom DocPerm), setting permission levels (permlevel) for document-level vs field-level access, hiding or masking specific columns/fields for a Role, scoping User Permissions to restrict a user to specific linked records, or exporting any of this configuration as fixtures for deployment. Trigger this any time the user mentions Roles, DocType permissions, "Custom DocPerm", field-level or column-level hiding, permission levels, User Permissions, "Has Role", or asks to set up / edit / export role-based access control for a Frappe or ERPNext site — even if they don't name the doctypes explicitly (e.g. "hide the salary field from HR users", "restrict this role to only their own records", "export the role permissions as fixtures", "give Sales User read-only on Sales Order").
---

# Role & Permission Manager (Frappe / ERPNext)

This skill governs how to reason about and edit Frappe/ERPNext's role-based permission system safely, and how to package that configuration as fixtures for deployment. Permission changes affect every user on a site immediately, so the two things that matter most throughout this skill are: **understand the full picture before changing anything**, and **confirm the exact change with the user before saving it**.

## Mental model

Frappe's access control is layered. Get comfortable with the chain before touching any record:

```
User ──(Has Role)──> Role ──(Custom DocPerm)──> DocType, at a Permission Level
                                                        │
                                        Level 0 = document-level (gates everything)
                                        Level 1+ = field-level (only matters if a field
                                                    is set to that level via Customize Form)

User ──(User Permission)──> restricts which specific records of a linked DocType
                              the User can see at all, regardless of role rights
```

Three things follow directly from this model, and they should shape every task you do here:

1. **Level 0 is the gate.** If a Role has no Level 0 access to a DocType, any rule you add at Level 1+ for that Role does nothing. Always check Level 0 exists before adding or editing a field-level rule.
2. **Roles are the reusable unit, not permission sets.** The system's own best practice is: don't clone a permission rule onto a new Role just because a user needs a slightly different mix — instead, assign multiple existing Roles to that User. If a request looks like "create a new Role that's almost the same as X but with one extra right," check with the user whether combining Roles solves it before creating a new Role.
3. **Role permissions, field-level permissions, and User Permissions are independent layers.** A user can have full Write rights on Sales Invoice (Role/Custom DocPerm) but still be unable to see a specific customer's invoices (User Permission), or be able to open a document but not edit one specific field (field permlevel). When a user reports "I can't do X," identify which layer is actually responsible before editing anything.

## Permission rights, in plain terms

| Right | What it actually allows |
|---|---|
| Select | Pick a record in a link field / search, without opening its full form |
| Read | Open and view the document |
| Write | Edit fields on an existing document |
| Create | Make new documents (does not imply Write on existing ones) |
| Delete | Remove documents (in practice, usually only Draft/Cancelled ones) |
| Submit / Cancel / Amend | Move a submittable document through its workflow states |
| Print | Generate/download a PDF of the document |
| Email | Send the document by email from within it |
| Report | See this DocType's data inside Report view / linked reports |
| Export | Pull data out of Report view |
| Import | Use the Data Import tool to create/update records in bulk |
| Share | Grant another specific user access to one document |
| Mask | Allowed to turn on field masking (e.g. a phone number shown as `811XXXXXXX`) |
| Impersonate | Allowed to act as another user for testing — treat edits to this flag with the same caution as Delete; it's powerful and newer, so it won't appear in every version's docs |

`if_owner` is a qualifier, not a right: when set to `1` on a Custom DocPerm row, every right on that row only applies to documents the user themself created. It's how you implement "users can edit their own expense claims but not everyone else's" without a separate Role per user.

## The two core records you'll be creating and editing

### `Has Role` — attaches a Role to a User

Child table row living under the User document (`parenttype: "User"`, `parentfield: "roles"`). Example:

```json
{
  "role": "Sales User",
  "parent": "user@example.com",
  "parentfield": "roles",
  "parenttype": "User",
  "doctype": "Has Role"
}
```

To change what a user can do at the broadest level, you add or remove rows here — you're not editing rights, just attaching/detaching an existing Role.

### `Custom DocPerm` — the actual rule (Role × DocType × Permission Level)

One row per Role, per DocType, per permission level. Example:

```json
{
  "parent": "Expense Claim Type",
  "role": "HR User",
  "if_owner": 0,
  "permlevel": 0,
  "select": 0, "read": 1, "write": 1, "create": 1,
  "delete": 0, "submit": 0, "cancel": 0, "amend": 0,
  "mask": 0, "report": 0, "export": 0, "import": 0,
  "share": 0, "print": 0, "email": 0, "impersonate": 0,
  "doctype": "Custom DocPerm"
}
```

`parent` here is the target DocType's name (e.g. "Expense Claim Type"), not a user. This is the record you'll be editing for almost every permission request — granting a right, restricting a right, or setting up a field-level rule.

## Workflow 1 — Assigning or changing a User's Roles

1. Confirm the exact user (email/username) and the exact Role name — Frappe Role names are case- and spelling-sensitive, and a near-miss silently creates no effect rather than erroring.
2. Check what Roles the user already has. If the request is "give them access to X," first check whether an existing Role already covers it — adding a redundant Role is harmless but adding a redundant *permission rule* is not (see Workflow 2).
3. Add/remove the `Has Role` row for that user. This alone does nothing if the Role itself has no Custom DocPerm rows — flag that to the user if it's the case, since "I added the Role but nothing changed" is a common point of confusion.
4. Never invent a new Role to solve a one-off access need if combining Roles the user already has (or already exist in the system) would do it — that's the system's own stated best practice, and it keeps the permission surface auditable.

## Workflow 2 — Creating or editing DocType permissions (Custom DocPerm)

This is the highest-stakes part of the skill: a Custom DocPerm change is live for every user with that Role the moment it saves. Before writing anything:

1. **State the change back in plain language and get explicit confirmation.** E.g. "This will let HR User create and edit Expense Claim Type records, but not delete or submit them. It applies to every user with the HR User role. Shall I save this?" Do this even if the user's original request seemed unambiguous — permission edits are exactly the place where a small misunderstanding (wrong Role, wrong DocType, wrong level) causes real damage.
2. **Read before you write.** Fetch the existing Custom DocPerm rows for that Role + DocType (all permission levels) so you're editing the actual current state, not guessing at it. Report what you found before proposing a change.
3. **Check Level 0 first.** If the request is about a field-level (Level 1+) restriction and the Role has no Level 0 row for that DocType, say so — the field-level rule will have no effect until Level 0 access exists.
4. **Change only the flags that were asked for.** Don't touch unrelated rights on the same row, don't touch other Roles' rows on the same DocType, and don't touch other permission levels unless the task requires it.
5. **Avoid duplicate rows.** A given (Role, DocType, permlevel, if_owner) combination should have exactly one Custom DocPerm row. If one already exists, edit it in place rather than creating a second row that will conflict or shadow it.

### Setting up field-level (column) hiding specifically

This needs two coordinated pieces, not just one:

1. Use Customize Form to set the target field's Permission Level to a number ≥ 1 (this detaches that field from Level 0 and puts it under its own gate).
2. Create or edit a Custom DocPerm row for the Role you want to restrict, at that same permlevel, with `read: 0` (fully hidden) or `write: 0` with `read: 1` (visible but read-only).
3. Make sure any Role that should still see/edit the field has a Custom DocPerm row at that permlevel with the right flags set — raising a field's permlevel silently restricts *every* Role that doesn't have an explicit row at that level, so check for Roles you might be unintentionally locking out.

## Workflow 3 — Editing an existing permission correctly

When the ask is "change" rather than "create":

1. Look up the record by its actual identity — (Role, parent DocType, permlevel, if_owner) — not by name/ID alone, since the same logical rule can exist across environments with different record names.
2. Toggle only the specific boolean(s) the user asked about.
3. Re-fetch or re-state the row after saving and confirm it matches what was intended — this is cheap insurance against a typo turning "write: 1" into "write: 0" on the wrong row.

## Workflow 4 — Record-level restriction via User Permissions

When the need is "this user should only see *their* records" rather than "this user should have fewer rights," that's a User Permission, not a Custom DocPerm change:

1. Identify the linking DocType (e.g. restrict a user to Blog Posts by a specific Blogger) — User Permissions work by walking a Link field, so the target DocType needs a field that links to the value you're restricting on.
2. Create the User Permission (user, allowed document type, allowed document value).
3. Note for the user: only System Manager, or a Role with the "Set User Permissions" right, can create these for other users — check that whoever will be doing this in the actual system has that right.

## Workflow 5 — Tracking requirements and exporting as fixtures

When a request is really "set up this whole permission scheme" (several Roles, several DocTypes, some field-level hiding, maybe User Permissions), don't just make the edits one by one — track the full requirement first, then execute, then export.

1. **Track before you act.** Write out the intended end-state plainly: which Roles, which DocTypes, which permission levels, which fields get hidden/masked, which User Permissions apply. Share this with the user and get confirmation before creating a single record — this is the same "confirm before editing" principle from Workflow 2, just applied to the whole batch instead of one row.
2. **Create the records** on the working site following Workflows 1–4.
3. **Export as fixtures** so the config can move to other environments (staging → prod) without re-doing manual clicks. In the app's `hooks.py`:

   ```python
   fixtures = [
       {"doctype": "Custom DocPerm", "filters": [["parent", "in", ["Expense Claim Type", "Sales Order"]]]},
       {"doctype": "Has Role", "filters": [["parent", "in", ["user@example.com"]]]}
   ]
   ```

   Then generate the actual JSON with:

   ```bash
   bench --site <site-name> export-fixtures
   ```

### The special lifecycle rule for `Custom DocPerm` and `Has Role` fixtures

Permission requirements here change often — day to day, not release to release — so these two doctypes do **not** follow normal fixture practice of accumulating and versioning every past configuration. Instead:

- The fixtures JSON and the `hooks.py` filter entries for `Custom DocPerm` and `Has Role` should only ever represent the **current, active** requirement — not a history of every requirement that's ever existed.
- When a new permission requirement comes in, **replace** the existing fixtures/filters for these two doctypes rather than adding alongside them: remove the old filter conditions and JSON, regenerate fresh ones from the new current state.
- Once that fresh export has gone to production, it becomes the new baseline. The *next* time a requirement changes, repeat the same replace-not-append step.
- **This behavior is scoped strictly to `Custom DocPerm` and `Has Role`.** Every other doctype's fixtures (Roles themselves, Property Setters, workflow states, whatever else the app exports) should keep normal additive/versioned handling — do not apply this trash-and-regenerate pattern to them.
- Before deleting any existing fixtures file or filter entry, say what you're about to remove and why, and get confirmation — this is a destructive step on a config file, and the same "confirm before changing something live-affecting" principle applies even though it's a file rather than a database row.

If it's ever unclear whether "these two doctypes" refers to something other than `Custom DocPerm` and `Has Role` in a given conversation, ask — don't assume the scope silently, since applying the replace behavior to the wrong doctype would erase history that was meant to be kept.

## Before saving any permission change — quick checklist

- Did I read the current state before proposing a change, instead of assuming it?
- Did I state the change in plain language and get explicit confirmation?
- Am I touching only the Role/DocType/level/field the user asked about?
- If this is a field-level rule, does Level 0 access already exist for that Role on that DocType?
- If I'm raising a field's permlevel, have I checked which other Roles need an explicit rule at that level so they aren't accidentally locked out?
- Am I about to create a new Role where combining existing Roles on the User would do the job instead?
- If this touches fixtures for `Custom DocPerm` or `Has Role`, have I replaced the old config rather than piling onto it — and confirmed the deletion first?

## Anti-patterns to avoid

- Cloning a permission rule into a brand-new Role instead of assigning an additional existing Role to the user.
- Adding a field-level (Level 1+) Custom DocPerm row without verifying Level 0 access exists.
- Editing a Custom DocPerm row's unrelated flags "while I'm in there."
- Creating a second Custom DocPerm row for a (Role, DocType, permlevel) pair that already has one, instead of editing the existing row.
- Letting `Custom DocPerm` / `Has Role` fixtures accumulate silently across requirement changes instead of replacing them.
- Applying the fixtures replace-not-append rule to any doctype other than `Custom DocPerm` / `Has Role`.
- Saving any permission or User Permission change without first stating the change and getting confirmation.
