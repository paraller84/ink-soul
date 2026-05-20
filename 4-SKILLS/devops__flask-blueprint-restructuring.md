---
name: flask-blueprint-restructuring
category: devops
description: >-
  Systematically restructure an existing Flask application with a single
  overgrown Blueprint into cleanly separated Blueprints (e.g., parent vs student,
  admin vs user), while unifying fragmented data sources across independent write paths.
version: 1.6.0
---

# Flask Blueprint Restructuring

## When to Use

Use this skill when an existing Flask application needs to **split an overgrown Blueprint** or **restructure routing** due to:

- A single Blueprint contains mixed functionality (e.g., parent + student routes in one file)
- Multiple independent data write paths have evolved into fragmented data sources
- Route prefixes need renaming (e.g., `/v2` → `/parent`, `/v3` → `/student`)
- Routes are being **silently overridden** by another Blueprint at the same prefix (diagnose via Step 1a)
- Authentication needs to be added/strengthened across the new Blueprints
- The application has been running for a while and the data has already diverged

**Also use when** creating a **parallel Blueprint system** (Blueprint Replication) — a new subsystem that follows the same 3-BP architecture as an existing one (e.g., creating a Chinese module by copying the English module's Blueprint pattern). See `references/blueprint-replication-pattern.md`.

**Do NOT use** for:
- New Flask service onboarding (use `flask-service-onboarding`)
- Simple feature additions (use C005 workflow at appropriate tier)
- First-time deployment of a code-complete app (use `flask-service-onboarding`)

## Prerequisites

- Flask application running on a port
- JSON or file-based data storage (not a database)
- Existing Blueprint structure with clear route classification
- Source code accessible and modifiable

## Dependency Chain (Critical — Never Reorder)

The restructuring has a **strict dependency order** that must be followed:

```
Phase 1: Data Migration Script (independent)
    ↓
Phase 2: State Manager Refactoring (depends on Phase 1)
    ↓
Phase 3: Blueprint Prefix Changes (independent of 1-2, but must be before 4-5)
    ├── Phase 4: Source Blueprint Route Pruning
    └── Phase 5: Target Blueprint Route Addition + Auth
    ↓
Phase 6: Template + Engine Layer Changes
    ↓
Phase 7: Backward-compat Redirect Routes
    ↓
Phase 8: Authentication + Cascade Delete Fixes
    ↓
Phase 9: Template Link Updates
    ↓
Phase 10: Comprehensive Test
```

### Why This Order Matters

- **Data migration before state_manager**: The state_manager refactoring changes write paths. If you refactor first and then migrate, the migration might write to the old wrong path again.
- **Blueprint prefix BEFORE route changes**: You need the new prefixes registered before you can test routes. But changing prefix before making the old Blueprint stop handling those routes causes conflicts.
- **Routes BEFORE templates**: Templates reference routes. You need the new routes working before you know what template paths to update.
- **Backward-compat redirects AFTER new routes**: Only add old→new redirects after the new routes are confirmed working.

## Step-by-Step Process

### Step 1a: Diagnose Blueprint Registration-Order Overlap (New)

**Problem**: Two Blueprints registered at the same `url_prefix` in `app.py`. Flask matches routes in **registration order** — the first Blueprint's route wins, silently overriding the second. This causes the second Blueprint's routes to be unreachable even though they appear in the URL map.

**Symptoms**:
- A URL that should show page B instead shows page A (or redirects to it)
- `url_map` dump shows the expected route exists, but it never gets called
- Routes from the second Blueprint return 404 or hit the first Blueprint's catch-all handler

**Diagnosis**:
```python
# Dump all registered routes to check for duplicate prefixes
from app import app
for r in sorted(app.url_map.iter_rules(), key=lambda r: r.rule):
    print(f"{r.rule:50s} {sorted(r.methods)}")
# Look for Blueprints sharing the same prefix
```

**Fix**: Assign a distinct `url_prefix` to each Blueprint during registration:
```python
# ❌ BOTH at /english — eng_v3 wins, english_bp hidden
app.register_blueprint(eng_v3, url_prefix="/english")  # registered first
app.register_blueprint(english_bp, url_prefix="/english")  # never reached

# ✅ Distinct prefixes
app.register_blueprint(eng_v3, url_prefix="/english/student")
app.register_blueprint(english_bp, url_prefix="/english")
```

**Cascading effects to check after fix**:
1. Templates — any hardcoded `/english/portal` → `/english/student/portal`
2. Redirect routes — backward-compat `/v3/` → /english/student/`
3. `url_for()` calls — these auto-generate correct URLs, no change needed
4. Session authentication — if the old path bypassed role selection, session role was never set. Add role-setting at the entry point.

### Step 1: Discover Data Fragmentation

Before any code changes, audit **all data write paths** in the application:

```bash
# Find all write_file / save_state / save_* calls
grep -rn "write_json\|save_state\|save_content\|\.write(" app/ --include='*.py'
```

**Look for**: Multiple independent files that store the same type of data (e.g., `state.json` top-level vs `state.json` nested vs separate `dictation_records.json`). Each independent write path is a fragmentation point.

**Critical check**: Does the state manager's `record_answer()` or equivalent have a `student_id` / `user_id` parameter? If not, it's writing to a global namespace — the root cause of data fragmentation.

### Step 2: Write Data Migration Script

Create a **power-idempotent** migration script that:
1. Detects if migration is already done (check version field)
2. Merges all fragmented data sources into one unified structure
3. Uses "primary source wins" strategy (most recent/detailed source wins)
4. Backs up original files before writing
5. Handles edge cases: merged items, diverged history, orphaned entries
6. Verifies completeness after migration

The script must be safe to run multiple times (idempotent).

### Step 3: Refactor State Manager

After migration, update the state manager:
- Add `student_id` / `user_id` parameter to all write functions
- Add `_get_student_state(state, student_id)` helper that navigates to the correct nested path
- Ensure backward compatibility: existing callers without the parameter get `DEFAULT_STUDENT`
- Update `load_state()` to prefer the new unified structure
- Update `get_batch_stats()`, `get_wrong_items()`, `record_exercise()` to use student-scoped paths
- Add new functions: `record_dictation()`, `update_memory_curve()`, `get_student_item_state()`, session persistence

**Key pattern — dual model coexistence**: If the two Blueprints use different data models (e.g., v2 uses status tags, v3 uses SM-2 algorithm), both models must coexist in the same record:
```json
{
  "status": "learning",
  "wrong_count": 2,
  "consecutive_correct": 1,
  "memory_curve": { "ease_factor": 2.5, "interval": 1, "repetitions": 0 }
}
```

### Step 4: Blueprint Prefix Changes

Update Blueprint prefixes:

**Source Blueprint (overgrown one)**:
```python
# In app.py
app.register_blueprint(source_bp, url_prefix='/old-prefix')
# → Change to new prefix:
app.register_blueprint(source_bp, url_prefix='/new-prefix')
```

**Target Blueprint (receiving the moved routes)**:
```python
# In target/__init__.py
target_bp = Blueprint('target_bp', __name__, url_prefix='/old-student-prefix')
# → Change to:
target_bp = Blueprint('target_bp', __name__, url_prefix='/new-student-prefix')
```

**⚠️ Trap**: When a Blueprint's `__init__.py` sets `url_prefix` AND `app.py` also passes one via `register_blueprint`, resolve the duplication. If the Blueprint constructor specifies `url_prefix='/v3'`, then `register_blueprint(bp)` works without a prefix argument. But if `register_blueprint(bp, url_prefix='/parent')` is used, the Blueprint constructor should NOT set one.

### Step 5: Prune Source Blueprint Routes

In the source Blueprint's `routes.py`:
- Remove page routes for functionality being moved
- Remove API routes for functionality being moved
- Update remaining route paths to work with the new prefix
- Update API path prefixes (e.g., `/api/v2/students` → `/api/students`)
- Simplify imports: remove references to moved modules

**⚠️ Trap**: When removing URL parameter authentication mode from `require_role()`, check that the logger warning doesn't still reference the removed variable (`req_role`).

### Step 6: Extend Target Blueprint Routes

In the target Blueprint's `routes.py`:
- Keep all existing routes
- Add the migrated page routes and API routes
- Import shared engine modules from the source Blueprint's directory:
  ```python
  from ..v2.engine_x import function_y  # Not from the Blueprint
  ```
- Add authentication decorators to all API endpoints
- Add session persistence endpoints

### Step 7: Fix Template Cross-Blueprint Sharing

#### 7a — Template Directory Isolation

A Blueprint with `template_folder="templates"` can render templates in its own directory, but cannot render templates from another Blueprint's template directory directly.

**Solutions** — see Option A and Option B below.

#### 7b — `{% extends %}` Cross-Blueprint Contamination (Critical)

**Problem**: When a Blueprint view renders a template that `{% extends "base.html" %}`, Flask's `DispatchingJinjaLoader._iter_loaders()` searches **ALL** Blueprint jinja_loaders in **registration order**

**Quick Diagnosis** — curl the endpoint and check page identity markers (no code changes needed):

```bash
# Check <title> tag — reveals which subsystem's base/extended template is loaded
curl -sL http://localhost:5002/suspect-path/ | grep -o '<title>[^<]*</title>'

# Check CSS <link> — reveals which Blueprint's static files are referenced
curl -sL http://localhost:5002/suspect-path/ | grep -o 'href="[^"]*style.css"'

# Expected:  <title>语文学习 - 学习门户</title>
#            href="/chinese/student/static/style.css"
# If you see: <title>英语学习 - 学习门户</title>
#            href="/english/student/static/style.css"
# → template from a different Blueprint was loaded.
```

This catches two failure modes in one check:
- **Mode A** (direct template wrong): The `render_template()` call itself resolved to another Blueprint's template at the same subpath (e.g., `v3/portal.html` exists in both `english/v3/` and `chinese/v3/`)
- **Mode B** (extends wrong): The direct template resolves correctly, but `{% extends "v3/base.html" %}` resolves to another Blueprint's `base.html`

**Deep Diagnosis** — monkey-patch the Jinja2 loader to trace template loading (when curl check shows contamination but you need to confirm which layer):

**Problem**: When a Blueprint is defined **without** `template_folder`, Flask's `DispatchingJinjaLoader` does NOT skip it — it simply yields no loader for that Blueprint. But the search continues through ALL other Blueprints that **do** have `template_folder`. This means templates can be silently resolved from a completely different subsystem.

**Example**:
```python
# english/v3/__init__.py — NO template_folder set
eng_v3 = Blueprint("eng_v3", __name__, static_folder="static")

# english/__init__.py — HAS template_folder
english_bp = Blueprint("english_bp", __name__, template_folder="templates")
```

When `eng_v3` renders `render_template("v3/practice/session.html")`:
- `eng_v3` has no `jinja_loader` → skipped
- `english_bp` (registered later, but still searched) → **found in `english/templates/v3/practice/session.html`** instead of the intended `english/v3/templates/v3/practice/session.html`

**Symptoms**:
- Rendered page is **stale/old version** of the template despite the Blueprint's own template directory having a newer version
- Changing the template file in the Blueprint's directory has no effect
- `render_template()` returns 200 with wrong content, no error

**Diagnosis — check which Blueprint the template actually resolves from**:
```python
from app import app
with app.app_context():
    from flask import render_template
    result = render_template("suspect/path.html")
```

**Diagnosis — list all Blueprints and their template_folder settings**:
```python
from app import app
for bp_name, bp in app.blueprints.items():
    print(f"{bp_name:20s} template_folder={bp.template_folder!r}")
```
Output:
```
eng_v3                template_folder=None       # ❌ PROBLEM
english_bp            template_folder='templates' # resolves instead
```

**Fix** — always set `template_folder` on every Blueprint that renders templates:
```python
eng_v3 = Blueprint(
    "eng_v3",
    __name__,
    static_folder="static",
    template_folder="templates",  # ✅ MUST be set
)
```

**Prevention — audit during Blueprint creation or replication**:
```bash
grep -l "Blueprint(" ~/edu-hub/*/__init__.py ~/edu-hub/*/v*/__init__.py 2>/dev/null | \
  while read f; do
    if grep -q "template_folder" "$f"; then echo "✅ $f"; else echo "❌ $f"; fi
  done
```

**Root cause**: `DispatchingJinjaLoader._iter_loaders()` — a Blueprint without `template_folder` has `jinja_loader = None`, so it's skipped in the iteration. But the search **continues** through later Blueprints that do have one. If one of them happens to have a matching path, its template wins silently., not just the current Blueprint's loader. If another Blueprint (registered earlier) has a file with the same name, it shadows the current Blueprint's template.

**Example scenario**:
```python
# app.py — Blueprints registered in this order:
app.register_blueprint(math_v2, url_prefix="/math")          # 1st – has templates/base.html
app.register_blueprint(eng_v2, url_prefix="/english/parent")  # 2nd – also has templates/base.html
```

When `eng_v2` renders `parent/index.html` which `{% extends "base.html" %}`:
- `parent/index.html` → correctly resolves to `english/v2/templates/parent/index.html` ✅
- `base.html` → resolves to `math_sys/v2/templates/base.html` ❌ (found earlier in iteration order)

**Symptoms**:
- A page renders with **another subsystem's layout, CSS, JS, or title** — even though the route and template path are correct
- Changing the template file has no effect (wrong file is being loaded)
- `render_template("path/to/template.html")` loads the correct file, but its `{% extends %}` loads a wrong parent

**Diagnosis** — monkey-patch the Jinja2 loader to trace template loading:
```python
from app import app
with app.app_context():
    original = app.jinja_env.loader.get_source
    
    def traced(env, template):
        src, filename, uptodate = original(env, template)
        print(f'Loading: {template:40s} → {filename}')
        return src, filename, uptodate
    
    app.jinja_env.loader.get_source = traced
    
    with app.test_request_context(path='/suspect/path/'):
        from flask import render_template
        result = render_template('suspect/template.html')
```

Look for the `extends` target (`base.html`) resolving to an unexpected path.

**Root cause**: Flask's `DispatchingJinjaLoader._iter_loaders()` method:
```python
def _iter_loaders(self, template):
    loader = self.app.jinja_loader
    if loader is not None:
        yield self.app, loader               # 1. App-level templates
    
    for blueprint in self.app.iter_blueprints():
        loader = blueprint.jinja_loader
        if loader is not None:
            yield blueprint, loader           # 2. ALL Blueprints (not just current)
```

It does NOT filter by the current request's Blueprint — it iterates ALL Blueprints in registration order.

**Fixes** (choose one, ordered by preference):

**Fix A — Rename/move conflicting templates**: Ensure no two Blueprints have identically-named template files (especially `base.html`). If a Blueprint's unused templates (`math_sys/v2/templates/base.html`) are shadowing a used one, move or delete the unused ones.

**Fix B — Namespace templates with subdirectories**: Instead of `{% extends "base.html" %}`, use a subdirectory:
```
english/v2/templates/
  english/
    base.html
    parent/
      index.html   → {% extends "english/base.html" %}
```

Then `render_template("english/parent/index.html")` ensures the extends path is unique.

**Fix C — Move shared `base.html` to app-level `templates/`**: If multiple Blueprints share the same base template, move it up:
```bash
mkdir -p app/templates/shared/
cp english/v2/templates/base.html app/templates/shared/base.html
```
Then all Blueprints use `{% extends "shared/base.html" %}` — the app-level `templates/` is searched before Blueprint folders.

**Prevention**: During Phase 3 (Blueprint Prefix Changes), audit ALL Blueprint template directories for name conflicts using:
```bash
for bp_dir in $(find */v*/templates -type d); do
    find $bp_dir -name "*.html" -exec basename {} \; 
done | sort | uniq -d
```

This lists all `.html` filenames that appear in multiple Blueprints. Every duplicate is a potential contamination vector.

**⚠️ Known Trap 9**: {% extends %} resolves across Blueprint boundaries (see Known Traps below for full details).

**⚠️ Critical Trap**: Flask Blueprints with their own `template_folder` cannot render templates from another Blueprint's template directory.

Solutions (choose one):

**Option A — Move templates to app level**:
```python
# 1. Copy source Blueprint's shared templates to app's templates directory
cp -r source_bp/templates/shared/ app/templates/

# 2. Remove template_folder from target Blueprint (uses app default)
# In target/__init__.py:
target_bp = Blueprint('target_bp', __name__, static_folder="static", url_prefix="/new")
# NOTE: No template_folder = Flask searches app's templates/ first

# 3. Update render_template calls to use paths relative to app's templates dir
return render_template("v2/shared_page/index.html")  # not "shared_page/index.html"
```

**Option B — Copy specific directories**:
```python
import shutil
shutil.copytree("source_bp/templates/page_dir", "app/templates/page_dir")
```

Choose Option A when the target Blueprint needs significant template sharing. Option B is simpler for just 1-2 shared templates.

### Step 8: Add Backward-Compatible Redirect Routes

Add Flask routes at the app level for old URL patterns:

```python
@app.route("/old-prefix/")
@app.route("/old-prefix/<path:path>")
def redirect_old_to_new(path=""):
    """old prefix → new prefix redirect"""
    # Handle specific path mappings
    if path.startswith("old-path-segment/"):
        path = path[len("old-path-segment/"):]
    # Handle paths that were moved to a different Blueprint
    elif path.startswith("moved-function"):
        clean_path = path.rstrip("/")
        return redirect(f"/target-blueprint/{clean_path}")
    return redirect(f"/new-prefix/{path}")
```

**Edge cases to handle**:
- Trailing slashes: `/old/dictation/` vs `/old/dictation`
- Path name changes: `/old/student-records/` → `/new/records`
- Sub-paths: `/old/parent/word-bank` → `/new/word-bank` (strip the "parent/" segment)

### Step 9: Fix Cascade Delete

After discovering which data files exist, systematically update `delete_student_cascade()` or equivalent to cover ALL data sources:

1. Student record file
2. Wrong book file
3. Session store (files + directory)
4. State.json student layer
5. v3 dictation records file
6. v3 memory curve file
7. v3 student stats file

Each deletion should backup the data first, then use atomic write (.tmp→rename).

### Step 10: Comprehensive Testing

**Phase A — Route Conflict Check (always run first):**
```python
# Verify no Blueprint is shadowing another
from app import app
rules = {}
for r in app.url_map.iter_rules():
    if r.rule in rules:
        print(f"⚠️ DUPLICATE: {r.rule} already registered by {rules[r.rule]}")
    rules[r.rule] = list(r.methods)
print(f"Total unique routes: {len(rules)}")
```

Test all routes in this order:

```python
tests = [
    # Core pages (new routes)
    '/new-prefix/', '/new-prefix/sub-page/',
    '/target-blueprint/', '/target-blueprint/sub-page/',
    # API endpoints
    '/new-prefix/api/endpoint',
    '/target-blueprint/api/endpoint',
    # App-level routes (v1 compat)
    '/v1-compat/route',
    # Backward compat redirects
    '/old-prefix/', '/old-prefix/sub-page/',
    # Security
    '/new-prefix/api/protected-endpoint',  # → 403 without auth
    '/target-blueprint/api/protected-endpoint',  # → 403 without auth
]
```

## CSS Theme Switching Architecture

When a Flask application needs **different color themes for different user roles** (e.g., light/colorful parent theme vs dark/neutral student theme), use the **scope-based CSS override pattern** to avoid touching every template:

### Architecture

```python
# base.html — add block for body class
<body class="{% block body_class %}{% endblock %}">
  {% block content %}{% endblock %}
</head>
```

```css
/* static/css/parent-theme.css — scoped under .parent-theme */
.parent-theme {
  background: #f0f7ff;
}
.parent-theme .pg-card {
  background: #ffffff;
  border: 1px solid #dce8f5;
}
/* Use !important when inline <style> blocks in templates have dark hardcoded colors */
.parent-theme .stat-card { background: #ffffff !important; }
```

```jinja
{# Each parent page adds just 2 blocks #}
{% extends "base.html" %}
{% block body_class %}parent-theme{% endblock %}
{% block head_extra %}
  <link rel="stylesheet" href="{{ url_for('static', filename='css/parent-theme.css') }}">
{% endblock %}
```

### Key Design Decisions

| Decision | Why |
|----------|-----|
| **Scope under body class, not :root** | Student pages don't get parent-theme. Zero risk of leakage. |
| **`!important` for inline-style override** | Many templates have dark hardcoded `<style>` blocks from prototype-driven development. `!important` in the external CSS wins without touching each template's inline styles. |
| **`{% block body_class %}` in base.html** | Minimal template change — just add `class="{{ block body_class }}"` to body tag in base.html, all pages inherit. |
| **Separate CSS file, not inline** | Cacheable across pages, easy to swap themes, avoids duplicating CSS across 12+ templates. |

### When to Use

- Parent/admin gets a light/branded theme while student/user keeps a different look
- A subset of pages needs theme isolation without CSS variables polluting global scope
- The app already has hardcoded inline styles in templates (common in prototype-driven development)

### Pitfalls

- **!important maintenance burden**: Too many `!important` rules make future CSS changes fragile. Use only for overriding established inline styles; write new CSS without `!important`.
- **body_class not set**: If a new page template forgets `{% block body_class %}parent-theme{% endblock %}`, it falls back to the default theme. Add a smoke test that checks for the class in rendered HTML.
- **Template inheritance order**: `head_extra` is in `<head>`, inline `<style>` blocks are in `<body>` (inside `{% block content %}`). External CSS in `<head>` is overridden by later `<style>` in `<body>` — hence the need for `!important`. To avoid `!important`, load the theme CSS in a `{% block theme_styles %}` block placed AFTER `{% block content %}` in base.html.

### Actual Implementation Example

AI家庭教师 12 parent pages, all converted from dark to light blue (#f0f7ff) theme:
- `static/css/parent-theme.css` — 200+ lines of `.parent-theme` scoped overrides
- `templates/base.html` — added `{% block body_class %}`
- Each parent template — added `body_class` + `head_extra` (2 lines each)
- Verification: grep parent-theme class in rendered HTML

---

## Known Traps

1. **`req_role` survivor** — When removing Mode 2 authentication from `require_role`, the logger.warning call on the failure path still references `req_role`. Remove or replace the format argument.

2. **Template folder isolation** — A Blueprint with `template_folder="templates"` cannot render templates from another Blueprint's templates directory. Move shared templates to the app's templates directory or copy them.

3. **Prefix replacement in Blueprint registration** — If the Blueprint constructor sets `url_prefix='/v3'` AND `app.register_blueprint(bp, url_prefix='/parent')` is used, the registration prefix **replaces** the constructor prefix, it does NOT concatenate or double them. The final route is `/parent/route` not `/v3/parent/route` and not `/v3/route`. Choose one approach: either set prefix in the constructor (and don't pass it to register_blueprint), or don't set it in the constructor (and pass to register_blueprint).

4. **Static file path changes** — When Blueprint prefix changes from `/v2` to `/parent`, static files at `/v2/static/style.css` become `/parent/static/style.css`. Update all `<link>` and `<script>` tags in templates.

5. **Trailing slash mismatches between redirected URL and target route** — Flask's redirect sends the exact path. If the target route doesn't have a trailing slash but the redirect URL does, Flask returns 404 (or 500 if there's a broad exception handler). Strip trailing slashes in the redirect function.

6. **`@errorhandler(Exception)` masking 404s** — A broad `@app.errorhandler(Exception)` catches werkzeug 404s and returns them as 500 JSON. During restructuring, many routes temporarily 404. Either narrow the handler to `@app.errorhandler(500)` or temporarily remove it.

7. **State manager import cycling** — When state_manager.py imports from v2/migration.py and migration.py also imports back, this creates a circular import. Resolve by having state_manager import migration only in functions (lazy import) or only at module bottom for compatibility shims.

8. **Blueprint registration-order overlap** — Two Blueprints registered at the same `url_prefix` do NOT merge. Flask iterates Blueprints in registration order and uses the **first matching route**. The second Blueprint's routes are silently unreachable. This is distinct from Trap 3 (prefix replacement), which is about one Blueprint's constructor prefix vs registration prefix. This trap is about TWO DIFFERENT Blueprints at the SAME prefix. See Step 1a for diagnosis and fix.

**How to distinguish from Trap 3**: Trap 3 produces wrong URL paths (e.g., `/english/route` instead of `/english/student/route`). Trap 8 produces conflicting routes where one Blueprint shadows another at the same path level (both have `/english/` root route, only the first Blueprint's fires).

9. **`{% extends %}` resolves across Blueprint boundaries, not just current Blueprint** — Flask's `DispatchingJinjaLoader._iter_loaders()` iterates ALL Blueprint jinja_loaders in registration order (plus the app-level loader). When a template `{% extends "base.html" %}`, the search does NOT limit to the current Blueprint's template folder — it finds the first `base.html` across ALL Blueprints. This means a Blueprint registered earlier can shadow another Blueprint's template.

**Variant A — Both Blueprints have `base.html` (shadowing)**: The earlier-registered Blueprint's `base.html` shadows the current one. The page renders the wrong layout despite loading the correct direct template. Fix: move/rename the conflicting templates or namespace with subdirectories.

**Variant B — Current Blueprint has NO `base.html` (silent fallback)**: The current Blueprint's template uses `{% extends "base.html" %}` but there is NO `base.html` at that path in its own template directory. Flask searches ALL Blueprints and finds another Blueprint's `base.html`. Since the foreign base template uses different block names (e.g., `{% block content %}` instead of `{% block body %}`), the current template's `{% block body %}` content is **silently dropped** — only `{% block scripts %}` renders if it happens to match. The page appears empty or with only JS.

**Symptom of Variant B**: Page title changes to another subsystem's, no visible HTML structure loads, but `<script>` content is present. JavaScript errors show `Cannot set properties of null (setting 'innerHTML')` because the expected DOM elements were never rendered.

**Root cause of Variant B**: The extends path is relative to the template directory root (`base.html`), but the actual base template is in a subdirectory (`v2/base.html`). The current Blueprint has no file at `templates/base.html`, so the fallback search finds a completely unrelated Blueprint's `templates/base.html`.

**Diagnosis**: Verify that the extends target file exists in the current Blueprint's template directory:
```bash
# Check if base.html exists at the expected path
find /path/to/current-blueprint/templates/ -name "base.html"
# If it's in a subdirectory (e.g., templates/v2/base.html), the extends path is wrong
```

**Fix for Variant B**: Update the extends path to include the subdirectory:
```jinja
{# ❌ Current Blueprint has templates/v2/base.html but uses #}
{% extends "base.html" %}        {# → fallback to wrong Blueprint #}

{# ✅ Match the actual path #}
{% extends "v2/base.html" %}
```

**Diagnostic indicator**: Route returns 200 with correct template path, but the page renders another subsystem's layout/CSS/title. The `extends` target resolves to the wrong Blueprint's file, not the current one.

10. **Missing `template_folder` causes silent fallback** — When a Blueprint lacks `template_folder`, Flask's `DispatchingJinjaLoader` skips it and continues searching other Blueprints. A completely unrelated Blueprint's template directory can serve the template instead. See Step 7c for diagnosis and fix. Always set `template_folder="templates"` on every Blueprint that calls `render_template()`.

14. **`send_static_file()` vs `render_template()` confusion** — `Blueprint.send_static_file(path)` serves files from the Blueprint's **static** directory, NOT the templates directory. When migrating templates to a new Blueprint, a common error is:
    ```python
    # ❌ WRONG — looks for static/v2/portal.html, file is in templates/
    @chinese_v2_bp.route("/")
    def index():
        return chinese_v2_bp.send_static_file("v2/portal.html")
    
    # ✅ CORRECT
    @chinese_v2_bp.route("/")
    def index():
        return redirect(url_for("chinese_v2_bp.portal"))
    ```
    **Diagnosis**: If a route that should show HTML returns 404, check whether the route uses `send_static_file()` — grep for this pattern in the Blueprint's routes file.

15. **Missing template context variables** — When migrating templates to a new Blueprint, the template may reference variables (`student.name`, `student.avatar`, etc.) that were implicitly available in the old system but aren't being passed by the new route handler. This causes a 500 error with `jinja2.exceptions.UndefinedError`. 
    **Diagnosis**: Check the Flask server log for `UndefinedError` traceback. Template variables passed via `render_template("page.html", student=student)` must match exactly what the template expects.  
    **Prevention**: Before deploying a migrated route, grep the template for `{{ variable` and `{% if variable` patterns to enumerate all expected context variables, then ensure the route handler provides them all.

16. **Portal links out of sync after version upgrade** — When creating a new version of a subsystem (e.g., v2.0 rewrite with new features), the subsystem is deployed at a new URL path (e.g., `/chinese/v2/` instead of `/chinese/student/`). But portal links (in `app.py`, subsystem's own `__init__.py` landing page, and the role selection page) still point to the old path. This causes users to enter the old version despite the new one being deployed.
    
    **Prevention — portal link audit after version upgrade:**
    ```bash
    # List ALL links to the old path in the entire project
    grep -rn '/chinese/student/' ~/edu-hub --include="*.py" --include="*.html"
    grep -rn '/chinese/parent/' ~/edu-hub --include="*.py" --include="*.html"
    ```
    Update every match. Key locations to check:
    - `app.py` — `_render_subject_select()` function (parent and student branches)
    - `chinese/__init__.py` — LANDING_PAGE HTML with role selection links
    - Any static/template HTML files with hardcoded links

13. **Dead code remnants after feature removal** — Removing HTML sections leaves behind CSS selectors (`.upload-area` styling), JavaScript functions (`showTaskWizard`, `setMode`), and HTML references (`onclick="showTaskWizard('exam')"`). These remnants bloat the file and can cause stale references. **Prevention** — after each removal pass, run: `grep -n "removed-feature\|photo\|vision\|upload-area\|dead-function-name" file.html` to find all traces. Or better: use a single `grep` for the removed feature's content keywords across the entire file.

17. **`url_for()` endpoint name mismatch in templates** — When a template calls `{{ url_for('blueprint_name.function_name') }}` but `function_name` doesn't exist in the Blueprint, Flask raises `BuildError` → 500 Internal Server Error. This is distinct from Trap 15 (missing context vars → `UndefinedError`).

    **Symptom**: The page returns 500. Flask server log shows:
    ```
    BuildError: Could not build url for endpoint 'blueprint.endpoint'.
    Did you mean 'blueprint.some_other_endpoint' instead?
    ```
    The "Did you mean" hint reveals the template references a route function that was never created.

    **Diagnosis** — reproduce via test_client with a valid session to get the traceback:
    ```python
    from app import app
    with app.test_client() as client:
        with client.session_transaction() as sess:
            sess['v2_role'] = 'parent'
        r = client.get('/suspect/path/')
        print(f'Status: {r.status_code}')
    ```

    **Root cause**: Template `url_for()` references an endpoint that was either never created, renamed, or cargo-culted from another Blueprint's template without updating.

    **Fix** (choose one):
    - **Option A** — Add the missing route function to the Blueprint
    - **Option B** — Change the template's `url_for()` to a valid existing endpoint

    **Prevention** — during Phase 9, enumerate all `url_for()` calls in templates and cross-reference against route function names:
    ```bash
    # Extract all url_for('bp_name.*') calls from templates
    grep -roP "url_for\('([^']+)" templates/ | sed "s/url_for('//" | sort -u
    # Then manually check each against routes.py
    ```
    Also check hardcoded `href` paths in templates after Phase 9 — these 404 silently, harder to catch than `url_for()` failures.

### Step 11: Post-Consolidation Cleanup — Retire Old Standalone Entry Points

After consolidating multiple standalone Flask apps into a single unified app (edu-hub), the old `app.py` entry points remain as dead files that cause confusion. A future session may inadvertently restart the wrong process, apply fixes to the wrong codebase, or duplicate work.

**Symptom of the trap**: User says "问题依然存在" even though you fixed the right shared code — you restarted the standalone app instead of the unified one.

**When to apply**: Immediately after the final test pass of any consolidation/restructuring that merges multiple apps.

**Process:**

1. **Identify all legacy entry points** — Find all `app.py` files that are no longer used by the unified app:
   ```bash
   find ~/edu-hub -maxdepth 2 -name "app.py" | sort
   # Cross-reference with what the unified app.py actually imports (grep "from.*import" app.py)
   ```

2. **Create a `_retired/` directory** in the project root:
   ```bash
   mkdir -p ~/edu-hub/_retired
   ```

3. **For each legacy entry point**, move it to `_retired/` with a RETIRED header:
   ```python
   """
   ===============================================================================
     RETIRED — 已退役
   ===============================================================================
     文件: <relative-path>
     原用途: <description>（独立Flask入口）
     退役原因: 已被 <unified-app-path> (统一入口, port <port>) 替代
     退役日期: <YYYY-MM-DD>
     注意: 请勿启动本文件。所有功能已集成至统一应用。
   ===============================================================================
   """
   ```

4. **Remove the original file** (user explicitly chose this approach):
   ```bash
   rm ~/edu-hub/english/app.py  # example
   ```

5. **Check for crontab references** to the old entry point:
   ```bash
   crontab -l | grep -E "old/app\.py|old\/app\.py"
   ```

6. **Verify the unified app still works** after removal — since the unified app imports modules, not the standalone `app.py` files:
   ```bash
   curl -s -o /dev/null -w '%{http_code}' http://localhost:<unified-port>/health
   ```

**Verification checklist:**
- [ ] All legacy `app.py` files moved to `_retired/` with RETIRED headers
- [ ] Unified app's core pages render correctly (200 or 403 from auth)
- [ ] No crontab references to old entry points
- [ ] User's actual URL is served by the unified app, not a standalone one

### Step 12: Feature Modification Based on Test Feedback

Use after a user or QA team provides specific feedback requiring multiple targeted changes to an existing Flask application.

**When to use**: The user gives 5-15 specific changes (remove features X, merge Y into Z, add login overlay, fix broken links). This is NOT a full restructure (use Steps 1-10) and NOT a simple feature addition (use C005 at appropriate tier).

**Process**:

1. **Parse feedback → task breakdown**: Map each feedback item to specific file changes:
   ```
   Feedback: Remove "创建默写任务" from quick actions
     → template: remove <a> element from quick-grid
     → JS: check if showTaskWizard() still referenced elsewhere
     → check: no other onclick=showTaskWizard in HTML
   
   Feedback: Merge word bank + exam bank → unified material bank
     → route: add /material-bank page route
     → create templates/material-bank/index.html
     → template: change quick-grid link from /word-bank to /material-bank
   ```

2. **Allocate subagents by file group** (not by feedback item):
   ```
   Subagent 1: Template modifications (parent/index.html)
     - Remove sections (quick-actions, task wizard, photo sections)
     - Replace sections (import section → dual-tab)
     - Add sections (login overlay)
   
   Subagent 2: Routes + new page templates (routes.py + new .html files)
     - Add page routes for new pages
     - Add API routes needed by new JS
     - Create new templates
     - Fix existing templates (wrong_book, records)
   ```

3. **⚠️ Critical coordination step**: Before delegating, check what API endpoints exist in routes.py. If the template JS will call a new API endpoint, either:
   - Create it first in the main thread, OR
   - Explicitly tell the template subagent the exact API URL to call, AND ensure the routes subagent creates it

4. **Verify post-change**:
   ```
   # Check no dead references
   grep -n "拍照\|vision\|photo\|upload-area\|showTaskWizard\|setMode\|toggleChip" templates/parent/index.html
   
   # Verify removed content
   for term in removed_terms: assert term not in page
   for term in added_terms: assert term in page
   
   # Check JS→API consistency
   grep -oP "fetch\('/english/parent/api/[a-z/]+'" templates/parent/index.html | while read api; do
     route=$(echo "$api" | grep -oP "api/[a-z/]+" | sed 's/\/$//')
     grep -q "$route" routes.py || echo "⚠️ MISSING route: $route"
   done
   ```

5. **Clean up remnants**: After removing a feature:
   - Remove its CSS block from `<style>` section
   - Remove its JavaScript functions
   - Remove any `onclick` or `href` references in remaining HTML
   - Run content grep to verify

See `references/test-feedback-to-feature-changes.md` for a full case study.

### Step 10 (updated): Comprehensive Testing — Add Dead Code Check + Portal Link Audit

After the standard Phase A/B route testing, add:

```python
# Dead code check — ensure no remnants of removed features
DEAD_TERMS = {
    "拍照": "Photo section remnant",
    "拍摄": "Photo section remnant",
    "vision": "Vision JS remnant",
    "upload-area": "Upload CSS remnant",
    "showTaskWizard": "Task wizard JS remnant still referenced",
}

for term, warning in DEAD_TERMS.items():
    for template in template_files:
        if term in template_content:
            print(f"⚠️ {warning}: '{term}' found in {template}")

# Portal link audit — after version upgrade, ensure all links point to new paths
OLD_PATHS = {
    "/chinese/student/": "Old Chinese student path — should be /chinese/v2/",
    "/chinese/parent/": "Old Chinese parent path — should be /chinese/v2/parent/",
}
for old_path, warning in OLD_PATHS.items():
    for py_file in Path("~/edu-hub").rglob("*.py"):
        content = py_file.read_text()
        if old_path in content:
            print(f"⚠️ {warning}: found in {py_file}")
```

## 关联文件

| 文件 | 说明 |
|:----|:-----|
| `references/standalone-to-blueprint-conversion.md` | 多独立 Flask App → 单应用子 Blueprint 的转换模式，含内置模块命名冲突陷阱和 EXTERNAL_URL 策略 |
| `references/extends-cross-blueprint-debugging.md` | `{% extends %}` 跨 Blueprint 污染问题的完整调试流程和根因分析 |
| `references/blueprint-replication-pattern.md` | **Blueprint 副本创建**：参考已有子系统创建平行 Blueprint 系统的完整模式（数据模型适配、模块复用、门户扩展、废弃功能清理） |
| `references/template-folder-missing-debug.md` | **template_folder 缺失调试**：Bluepoint 无 template_folder 时 Jinja2 静默回退到其他 Blueprint 的完整诊断流程、根因分析和预防清单 |
| `references/test-feedback-to-feature-changes.md` | **测试反馈→功能修改工作流**：基于用户/QA测试反馈对已有Flask应用进行定向修改的完整模式，含子代理协调陷阱和残留码清理 |
| `references/post-deployment-portal-audit.md` | **升级后 portal 链接审计**：创建子系统新版本后，检查所有 portal 链接（app.py + __init__.py）是否指向新版路径，见 `blueprint-replication-pattern.md` 的「升级后检查清单」 |
| `references/cross-module-portal-unification.md` | **跨模块门户统一**：三模块（英语/数学/语文）统一门户页面的完整模式 — 使用 `{% set %} + {% include %}` 共享模板组件、base.html block 定制、护眼配色方案、科目选择器一致性。包含关键陷阱（block 名不一致、容器差异、重复徽章、url_for 跨 Blueprint 500、预览稿实现差异）|
