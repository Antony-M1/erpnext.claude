---
name: frappe-page-api
description: >
  Use this skill whenever working with Frappe Framework custom Desk pages —
  creating a page with frappe.ui.make_app_page, wiring up page.set_title,
  indicators, primary/secondary actions, menu/action dropdown items, inner
  toolbar buttons, or page form fields; when writing Jinja template code that
  calls whitelisted frappe.* helpers (frappe.get_all, frappe.get_doc,
  frappe.format, frappe.render_template, etc.); when writing client-side
  Frappe JS utilities (frappe.get_route, frappe.set_route, frappe.provide,
  frappe.require); when rendering Frappe/frappe.io charts (frappe.ui.RealtimeChart);
  when making AJAX/server calls with frappe.call or frappe.db.* methods; or
  when adding frappe.logger()-based logging to a Frappe app. Trigger this
  skill for ANY task that touches Frappe's Page API, Jinja API, Common
  Utilities (js-utils) API, Chart API, Server Calls (AJAX) API, or Logging
  API — even if the user just says "add a button to this Frappe page" or
  "fetch this doctype data in my template".
tools:
  - Read
  - Write
  - Bash
---

# Frappe Page API Skill

This skill packages the reference documentation for building custom Frappe
Desk pages and using the surrounding client/server helper APIs. **Only use
the information contained in this file (and the source links below) to
answer Frappe API questions — do not invent methods that aren't documented
here.** If something isn't covered below, say so and point the user to the
relevant doc link rather than guessing.

## Source documentation (canonical — refer back to these if unsure)

- Page API: https://docs.frappe.io/framework/user/en/api/page
- Jinja API: https://docs.frappe.io/framework/user/en/api/jinja
- Common Utilities API (js-utils): https://docs.frappe.io/framework/user/en/api/js-utils
- Chart API: https://docs.frappe.io/framework/user/en/api/chart
- Server Calls (AJAX): https://docs.frappe.io/framework/user/en/api/server-calls
- Logging: https://docs.frappe.io/framework/user/en/api/logging

---

## Code style rules (always follow these when writing/editing code)

- Use `async/await` for all functions that involve any asynchronous call
  (`frappe.call`, `frappe.db.*`, `frappe.require` callbacks wrapped in
  promises, etc.) — never chain raw `.then()` unless there's no way around it.
- Prefer **arrow functions** (`const foo = async (args) => { ... }`) for
  almost everything — event handlers, callbacks, helper functions. Only fall
  back to a regular `function` when you genuinely need your own `this`
  binding (e.g. inside a Frappe Form/List controller object where `this`
  refers to the frm/listview).
- Use `frappe.require` to load any additional JS/CSS assets asynchronously,
  and use it to properly scope/create namespaces on `window` (in combination
  with `frappe.provide`) instead of polluting the global scope directly.
- Avoid comment-based function docstrings/headers. Code should be
  self-explanatory through naming; only add a comment when the "why" isn't
  obvious from the code itself.
- Pages must be **mobile responsive**. Don't assume a wide desktop viewport:
  use responsive CSS (flex-wrap, `%`/`vw` widths, media queries) instead of
  fixed pixel widths, avoid layouts that only work in multi-column desktop
  view, and test that page fields, inner toolbar buttons, and any custom
  HTML added via `page.main` degrade gracefully on small screens (buttons
  should wrap or collapse into the Menu dropdown rather than overflow).
- Use **dedicated HTML and CSS files** for page markup — avoid keeping HTML
  strings inside the JS file, even for a small block. Name HTML partials
  `{page_name}_{block_name}.html` and load/append them into the page
  accordingly (e.g. via `frappe.require` or a template fetch) instead of
  inlining markup as JS strings. Keep **one CSS file per page**, define
  custom classes in it, and reference those classes from the HTML rather
  than writing inline styles or one-off styles in JS.

---

## 1. Page API

Every screen inside the Desk is rendered inside a `frappe.ui.Page` object.

### Creating a page

```js
let page = frappe.ui.make_app_page({
    title: 'My Page',
    parent: wrapper, // HTML DOM Element or jQuery object
    single_column: true // create a page without sidebar
});
```

### Page instance methods

| Method | Purpose |
|---|---|
| `page.set_title(title)` | Set the page title + document (browser tab) title |
| `page.set_title_sub(subtitle)` | Set secondary title, shown on the right of the header |
| `page.set_indicator(label, color)` | Set an indicator label + color |
| `page.clear_indicator()` | Clear the indicator |
| `page.set_primary_action(label, handler, icon_class)` | Set primary action button; `icon_class` shown on mobile |
| `page.clear_primary_action()` | Remove primary action button |
| `page.set_secondary_action(label, handler, icon_class)` | Set secondary action button |
| `page.clear_secondary_action()` | Remove secondary action button |
| `page.add_menu_item(label, handler, standard)` | Add item to the Menu dropdown (`standard=true` marks it as a standard item) |
| `page.clear_menu()` | Remove the Menu dropdown items |
| `page.add_action_item(label, handler)` | Add item to the Actions dropdown |
| `page.clear_actions_menu()` | Remove the Actions dropdown items |
| `page.add_inner_button(label, handler, group)` | Add a button to the inner toolbar; pass `group` to nest it under a dropdown group |
| `page.change_inner_button_type(label, group, type)` | Change a custom button's type (e.g. `'primary'`, `'danger'`) by label (and optional group) |
| `page.remove_inner_button(label, group)` | Remove an inner toolbar button (optionally scoped to a group) |
| `page.clear_inner_toolbar()` | Remove the entire inner toolbar |
| `page.add_field(df)` | Add a form control to the page's form toolbar; `df` is a field-definition object (`label`, `fieldtype`, `fieldname`, `options`, `change()`, etc.) |
| `page.get_form_values()` | Returns an object of all current page form toolbar field values |
| `page.clear_fields()` | Clear all page form toolbar fields |

### Example usage (following code style rules)

```js
frappe.pages['my-custom-page'].on_page_load = (wrapper) => {
    const page = frappe.ui.make_app_page({
        title: 'My Page',
        parent: wrapper,
        single_column: true
    });

    page.set_indicator('Pending', 'orange');

    page.set_primary_action('New', async () => {
        await create_new_record();
    }, 'octicon octicon-plus');

    page.add_menu_item('Send Email', async () => {
        await open_email_dialog();
    });

    const status_field = page.add_field({
        label: 'Status',
        fieldtype: 'Select',
        fieldname: 'status',
        options: ['Open', 'Closed', 'Cancelled'],
        change: async () => {
            await refresh_list(status_field.get_value());
        }
    });
};
```

---

## 2. Jinja API

Whitelisted `frappe.*` methods usable inside Jinja templates (web pages,
print formats, email templates, etc.).

| Method | Purpose |
|---|---|
| `frappe.format(value, df, doc)` | Format a raw DB value to a user-presentable format, e.g. `{{ frappe.format('2019-09-08', {'fieldtype': 'Date'}) }}` → `09-08-2019` |
| `frappe.format_date(date_string)` | Format a date into a long human-readable form, e.g. `September 8, 2019` |
| `frappe.get_url()` | Returns the site URL |
| `frappe.get_doc(doctype, name)` | Fetch a document by name, e.g. `{% set doc = frappe.get_doc('Task', 'TASK00002') %}` |
| `frappe.get_all(doctype, filters, fields, order_by, start, page_length, pluck)` | Returns all records of a doctype (no permission filtering); only `name`s if `fields` omitted |
| `frappe.get_list(doctype, filters, fields, order_by, start, page_length)` | Same as `get_all` but filtered for the current session user's permissions |
| `frappe.db.get_value(doctype, name, fieldname)` | Returns a single field value, or a list if `fieldname` is a list |
| `frappe.db.get_single_value(doctype, fieldname)` | Returns a field value from a Single DocType |
| `frappe.get_system_settings(fieldname)` | Returns a field value from System Settings |
| `frappe.get_meta(doctype)` | Returns doctype meta (fields, title_field, image_field, etc.) |
| `frappe.get_fullname(user_email)` | Returns the full name for a user email (defaults to current session user) |
| `frappe.render_template(template_name, context)` | Render a Jinja template string or file with the given context |
| `frappe._(string)` / `_(string)` | Translate a string |
| `frappe.session.user` | Current session user |
| `frappe.session.csrf_token` | Current session's CSRF token |
| `frappe.form_dict` | Dict of query params if evaluated in a web request, else `None` |
| `frappe.lang` | Current two-letter, lowercase language code |

### Example

```html
{% set tasks = frappe.get_all('Task', filters={'status': 'Open'}, fields=['title', 'due_date'], order_by='due_date asc') %}
{% for task in tasks %}
### {{ task.title }}
Due Date: {{ frappe.format_date(task.due_date) }}
{% endfor %}
```

---

## 3. Common Utilities API (js-utils)

| Method | Purpose |
|---|---|
| `frappe.get_route()` | Returns the current route as an array, e.g. `["List", "Task", "List"]` |
| `frappe.set_route(route)` | Changes the current route; accepts parts, an array, a string, or route + options object |
| `frappe.format(value, df, options, doc)` | Format a raw value into a user-presentable string (client-side variant) |
| `frappe.provide(namespace)` | Creates a namespace on `window` if it doesn't already exist |
| `frappe.require(asset_path, callback)` | Asynchronously load one or more JS/CSS assets, then run `callback` |

### Example (per code style: async/await, arrow functions, frappe.require for namespacing)

```js
frappe.provide('frappe.ui.my_app');

const load_my_app_assets = async () => new Promise((resolve) => {
    frappe.require(['/assets/my_app/widget.js', '/assets/my_app/widget.css'], () => {
        resolve();
    });
});

const open_my_widget = async () => {
    await load_my_app_assets();
    frappe.ui.my_app.widget = new frappe.ui.my_app.Widget();
};
```

```js
// route helpers
frappe.set_route(['List', 'Task', 'Task'], { status: 'Open' });
const current_route = frappe.get_route();
```

---

## 4. Chart API

Frappe wraps Frappe Charts (SVG, fully configurable — see
https://frappe.io/charts for the underlying charting library) with a
real-time helper.

### frappe.ui.RealtimeChart

`new frappe.ui.RealtimeChart(dom_element, event_name, max_label_count, data)`

- `dom_element`: element (or selector) to render the chart into
- `event_name`: socket event that streams data updates
- `max_label_count`: max number of x-axis labels shown
- `data`: initial chart config (`title`, `data`, `type`, `height`, `colors`, etc.)

| Method | Purpose |
|---|---|
| `frappe.ui.RealtimeChart.start_updating()` | Start listening to the socket event and updating the chart |
| `frappe.ui.RealtimeChart.stop_updating()` | Stop listening to the socket event |
| `frappe.ui.update_chart(label, data)` | Manually append a label + data point(s) to the chart |

### Example

```js
const init_realtime_chart = async () => {
    const data = { datasets: [{ name: 'Some Data', values: [] }] };

    const chart = new frappe.ui.RealtimeChart('#chart', 'test_event', 8, {
        title: 'My Realtime Chart',
        data,
        type: 'line',
        height: 250,
        colors: ['#7cd6fd', '#743ee2']
    });

    chart.start_updating();
};
```

Server side (Python, run e.g. as a scheduled Hook job):

```py
data = {
    'label': 1,
    'points': [10]
}
frappe.publish_realtime('test_event', data)
```

`label` is the value appended to the x-axis; `points` is the array of values
plotted (one per dataset).

---

## 5. Server Calls (AJAX)

| Method | Purpose |
|---|---|
| `frappe.call(method, args)` / `frappe.call({...})` | Call a whitelisted Python method; supports `btn` (disable during request), `freeze` (freeze screen), `callback`, `error` |
| `frappe.db.get_doc(doctype, name, filters)` | Fetch a full Document object by name or by filters |
| `frappe.db.get_list(doctype, { fields, filters })` | Fetch a list of records |
| `frappe.db.get_value(doctype, name_or_filters, fieldname)` | Fetch one field value or several (pass a fieldname array) |
| `frappe.db.get_single_value(doctype, field)` | Fetch a field value from a Single DocType |
| `frappe.db.set_value(doctype, docname, fieldname, value)` | Set one field (or pass an object to set several) and save |
| `frappe.db.insert(doc)` | Insert a new document |
| `frappe.db.count(doctype, filters)` | Count records matching filters |
| `frappe.db.delete_doc(doctype, name)` | Delete a document |
| `frappe.db.exists(doctype, name)` | Returns `true`/`false` if a record exists |

### Example (per code style: async/await + arrow functions)

```js
const load_role_profile = async (role_profile) => {
    const r = await frappe.call('frappe.core.doctype.user.user.get_role_profile', {
        role_profile
    });
    return r.message;
};

const update_task_status = async (task_name, status) => {
    const r = await frappe.db.set_value('Task', task_name, 'status', status);
    return r.message;
};

const create_task = async (subject) => {
    const doc = await frappe.db.insert({ doctype: 'Task', subject });
    return doc;
};
```

For a "freeze screen + disable button" call:

```js
const submit_form = async ($btn) => {
    await frappe.call({
        method: 'frappe.core.doctype.user.user.get_role_profile',
        args: { role_profile: 'Test' },
        btn: $btn,
        freeze: true
    });
};
```

---

## 6. Logging

Frappe uses Python's built-in `logging` module, with per-bench and per-site
log files (log rotation keeps the last 20 files, 100kB each, by default).

| API | Purpose |
|---|---|
| `frappe.log_level` | Current log level of Frappe processes |
| `frappe.utils.logger.set_log_level(level)` | Set the log level and regenerate loggers dynamically |
| `frappe.loggers` | Dict of active loggers, keyed by `"{module}-{site}"` |
| `frappe.logger(module, with_more_info, allow_site, filter, max_size, file_count)` | Get (or create) a `logging.Logger`, with site/bench-aware log files |

`frappe.logger` arguments:
- **module** — logger name (and log filename)
- **with_more_info** — also logs the Form Dict, useful for request logging
- **allow_site** — pass a site name to log under that site explicitly (or `True` to guess)
- **filter** — a filter function for the logger
- **max_size** — max bytes per log file
- **file_count** — max number of rotated log files to retain

### Example (Python — server side)

```py
frappe.utils.logger.set_log_level("DEBUG")
logger = frappe.logger("api", allow_site=True, file_count=50)

@frappe.whitelist()
def update(value):
    user = frappe.session.user
    logger.info(f"{user} accessed counter_app.update with value={value}")

    current_value = frappe.get_single_value("Value", "counter")
    updated_value = current_value + value
    logger.debug(f"{current_value} + {value} = {updated_value}")

    frappe.db.set_value("Value", "Value", "counter", updated_value)
    logger.info(f"{user} updated value to {value}")

    return updated_value
```

This writes to `./logs/api.log` and `./sites/<site>/logs/api.log`.

---

## Usage notes for Claude

1. When a user asks about any of these six areas, answer **only** from the
   tables/examples above. If the exact method they need isn't listed here,
   say you don't have it documented in this skill and link to the relevant
   source doc above rather than guessing at an API surface.
2. When writing new code for the user, always apply the **Code Style
   rules** section (async/await, arrow functions, `frappe.require` for
   asset loading/namespacing, no docstring-style comments).
3. Keep Python-side code (Jinja helpers, `frappe.logger` calls, hooks) in
   plain, idiomatic Frappe Python — the async/await + arrow function rules
   are JS-specific and don't apply to the Python examples.
4. If the user explicitly says they want a **full page** / **standalone
   page** / **no existing ERPNext/Desk components** (e.g. "utilize full
   page", "no ERPNext existing component", "don't want to use the Page
   API"), do NOT build it with `frappe.ui.make_app_page` / the Page API.
   Instead build it as a standalone full-viewport page (e.g. a Website/Web
   Page route, a custom route rendered outside the Desk shell, or a
   full-page HTML template) so that none of the standard Desk chrome shows
   up — no top navbar/profile dropdown, no global search bar, no sidebar,
   no notifications icon, nothing from the default ERPNext/Desk shell.
   Only the user's own content should render on the page. Confirm this
   intent with the user if it's ambiguous whether they want a Desk page
   (with standard chrome) or a fully custom page (with none of it).
