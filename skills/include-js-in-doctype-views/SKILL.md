---
name: include-js-in-doctype-views
description: >
  Add client-side JS (Form, List, Tree, or Calendar overrides) for a doctype
  in a custom Frappe app and wire it up in hooks.py. Use whenever the user
  asks to "add JS to a doctype", "override a doctype's form/list/tree view",
  "add custom buttons/validations to a doctype", or references
  doctype_js / doctype_list_js / doctype_tree_js / doctype_calendar_js in
  hooks.py.
tools:
  - Read
  - Write
  - Bash
---

# Include JS in DocType Views (Frappe Custom App)

This skill scaffolds the client-side JS file for a doctype inside a custom
Frappe app's `public/js/` folder and registers it in the app's `hooks.py`,
so the script is injected into the correct Desk view (Form, List, or Tree).

Reference docs (use these links if you need details beyond what's here):
- Form overrides: https://docs.frappe.io/framework/user/en/api/form
- List overrides: https://docs.frappe.io/framework/user/en/api/list
- Tree overrides: https://docs.frappe.io/framework/user/en/api/tree

## Which override do you need?

| View you're customizing | hooks.py dict        | File naming convention      | JS global object                    |
|--------------------------|----------------------|------------------------------|--------------------------------------|
| Form (single record)     | `doctype_js`          | `public/js/{doctype}.js`      | `frappe.ui.form.on(doctype, {...})`  |
| List view                | `doctype_list_js`     | `public/js/{doctype}_list.js` | `frappe.listview_settings[doctype]`  |
| Tree view                | `doctype_tree_js`     | `public/js/{doctype}_tree.js` | `frappe.treeview_settings[doctype]`  |
| Calendar view            | `doctype_calendar_js` | `public/js/{doctype}_calendar.js` | `frappe.views.calendar[doctype]` |

The `{doctype}` file naming segment is the doctype's snake_case name
(e.g. "Expense Claim" -> `expense_claim.js`, "Location" -> `location.js`).

## Workflow

1. **Confirm the doctype(s) and view type(s)** the user wants JS added for
   (Form / List / Tree / Calendar). If ambiguous, default to Form since
   that's the most common override.
2. **Find the app root** — look for `hooks.py` at the top level of the
   custom app (e.g. `<app>/<app>/hooks.py`), and the `public/js/` folder
   next to it (e.g. `<app>/<app>/public/js/`). Create `public/js/` if it
   doesn't exist.
3. **Create the JS file(s)** at `public/js/{doctype_snake_case}.js` (and/or
   `_list.js`, `_tree.js`, `_calendar.js`) using the templates below.
   Always prefer arrow functions, and use `async/await` when the logic
   needs to await a server call, promise, or dialog result.
4. **Register the file(s) in `hooks.py`** by adding an entry to the
   relevant dict (`doctype_js`, `doctype_list_js`, `doctype_tree_js`,
   `doctype_calendar_js`). Uncomment the dict if it's currently commented
   out, and add to it if it already has entries — don't overwrite existing
   doctype keys.
5. **Remind the user to run** `bench build --app <app_name>` (or
   `bench start` in dev mode) and clear cache (`bench clear-cache`) so the
   new JS asset is picked up, and to hard-refresh the browser.

## hooks.py registration pattern

```py
doctype_js = {
    "Expense Claim": "public/js/expense_claim.js",
    "Location": "public/js/location.js",
    # add new doctype JS below, keep existing keys intact
}

doctype_list_js = {
    "Expense Claim": "public/js/expense_claim_list.js",
}

doctype_tree_js = {
    "Account": "public/js/account_tree.js",
}

# doctype_calendar_js = {"DocType": "public/js/doctype_calendar.js"}
```

Rules when editing hooks.py:
- If the dict assignment (e.g. `# doctype_list_js = {...}`) is commented
  out and you need it, uncomment it and turn it into a real dict.
- If the dict already exists, add a new key rather than replacing the
  whole block.
- Always use the exact doctype name (with correct casing/spacing) as the
  dict key — that's what Frappe matches against, not the filename.

## Form JS template (`public/js/{doctype}.js`)

Form scripts hook into Form Events (`setup`, `onload`, `refresh`,
`validate`, `before_save`, `on_submit`, `{fieldname}`, etc. — see the Form
Scripts doc for the full event list) via `frappe.ui.form.on(doctype, {...})`.
Handlers receive `frm` as the first argument; child table events also
receive `cdt` and `cdn`.

```js
frappe.ui.form.on("Expense Claim", {
	refresh: (frm) => {
		if (!frm.is_new()) {
			frm.add_custom_button(__("Custom Action"), async () => {
				const result = await frm.call("some_whitelisted_method", {
					some_arg: frm.doc.name,
				});
				if (result.message) {
					frappe.msgprint(__("Done: {0}", [result.message]));
				}
			});
		}
	},

	validate: (frm) => {
		if (!frm.doc.some_field) {
			frappe.throw(__("Some Field is required"));
		}
	},

	some_field: (frm) => {
		// triggered when `some_field` changes
		frm.set_value("dependent_field", frm.doc.some_field);
	},
});

// Child table events go in the same file, keyed by the child doctype name
frappe.ui.form.on("Expense Claim Detail", {
	expense_type: (frm, cdt, cdn) => {
		const row = frappe.get_doc(cdt, cdn);
		// react to the child row field change
	},
});
```

Common `frm` API you'll reach for: `frm.set_value()`, `frm.refresh()`,
`frm.save()`, `frm.add_custom_button()`, `frm.set_df_property()`,
`frm.toggle_enable()` / `frm.toggle_reqd()` / `frm.toggle_display()`,
`frm.set_query()` (for filtering Link fields — call this in `setup` or
`onload`), `frm.add_child()`, `frm.call()` (for whitelisted server
methods), and `frm.trigger()`.

## List JS template (`public/js/{doctype}_list.js`)

List views are customized via `frappe.listview_settings[doctype] = {...}`.

```js
frappe.listview_settings["Expense Claim"] = {
	add_fields: ["status", "total_claimed_amount"],
	filters: [["docstatus", "=", 1]],
	hide_name_column: true,

	onload: (listview) => {
		// triggers once before the list loads
	},

	get_indicator: (doc) => {
		if (doc.status === "Approved") {
			return [__("Approved"), "green", "status,=,Approved"];
		}
		return [__("Pending"), "orange", "status,=,Pending"];
	},

	formatters: {
		total_claimed_amount: (val) => frappe.format(val, { fieldtype: "Currency" }),
	},

	button: {
		show: (doc) => doc.status === "Pending",
		get_label: () => __("Approve"),
		get_description: (doc) => __("Approve {0}", [doc.name]),
		action: async (doc) => {
			await frappe.call({
				method: "frappe.client.set_value",
				args: { doctype: "Expense Claim", name: doc.name, fieldname: "status", value: "Approved" },
			});
			cur_list.refresh();
		},
	},
};
```

## Tree JS template (`public/js/{doctype}_tree.js`)

Only relevant for doctypes with "Is Tree" enabled. Configured via
`frappe.treeview_settings[doctype] = {...}`.

```js
frappe.treeview_settings["Account"] = {
	breadcrumb: "Accounting",
	title: __("Chart of Accounts"),

	get_tree_nodes: "path.to.whitelisted_method.get_children",
	add_tree_node: "path.to.whitelisted_method.handle_add_account",

	fields: [
		{ fieldtype: "Data", fieldname: "account_name", label: __("New Account Name"), reqd: true },
		{ fieldtype: "Check", fieldname: "is_group", label: __("Is Group") },
	],

	ignore_fields: ["parent_account"],

	onload: (treeview) => {},
	post_render: (treeview) => {},

	toolbar: [
		{
			label: __("Add Child"),
			condition: (node) => node && node.is_group,
			click: (node) => frappe.treeview_settings["Account"].add_node(node),
		},
	],
};
```

## Notes

- Prefer arrow functions everywhere in these scripts; use `async/await`
  whenever a handler calls `frm.call`, `frappe.call`, or otherwise awaits a
  promise, instead of chaining `.then()`.
- Client Scripts (created from the Desk UI under
  Customization > Client Script) are the alternative to app-level JS files
  — use those only for site-specific logic that shouldn't ship with the
  app; for anything meant to be part of the app/repo, use the
  `public/js/{doctype}.js` + hooks.py approach in this skill.
- If something about an event, API method, or option isn't covered above,
  check the linked docs rather than guessing.
