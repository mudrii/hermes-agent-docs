# Dashboard Plugins

Dashboard plugins let you add custom tabs to the Hermes web dashboard without patching the dashboard source tree. A plugin can ship frontend assets, optional backend API routes, and navigation metadata from the same plugin directory you already use for CLI and gateway extensions.

## Minimal structure

```text
~/.hermes/plugins/my-plugin/
  plugin.yaml
  __init__.py
  dashboard/
    manifest.json
    dist/
      index.js
      style.css
    plugin_api.py
```

Only the `dashboard/` subtree is required for dashboard extensions.

## Minimal manifest

```json
{
  "name": "my-plugin",
  "label": "My Plugin",
  "icon": "Sparkles",
  "version": "1.0.0",
  "tab": {
    "path": "/my-plugin",
    "position": "after:skills"
  },
  "entry": "dist/index.js"
}
```

## Frontend registration

The dashboard exposes a plugin SDK on `window.__HERMES_PLUGIN_SDK__`. Your bundle registers a page component on `window.__HERMES_PLUGINS__`.

```javascript
(function () {
  var SDK = window.__HERMES_PLUGIN_SDK__;
  var React = SDK.React;

  function MyPage() {
    return React.createElement("div", null, "Hello from a dashboard plugin");
  }

  window.__HERMES_PLUGINS__.register("my-plugin", MyPage);
})();
```

## Optional backend routes

If `manifest.json` sets `api`, Hermes loads that Python file and mounts its FastAPI router under `/api/plugins/<name>/`.

```python
from fastapi import APIRouter

router = APIRouter()

@router.get("/data")
async def get_data():
    return {"ok": True}
```

This becomes:

```text
GET /api/plugins/my-plugin/data
```

## Manifest fields

| Field | Required | Description |
|-------|----------|-------------|
| `name` | Yes | Unique plugin identifier |
| `label` | Yes | Tab label shown in the dashboard |
| `icon` | No | Lucide icon name; defaults to `Puzzle` |
| `version` | No | Plugin version string |
| `tab.path` | Yes | URL path for the tab |
| `tab.position` | No | Placement such as `end`, `after:skills`, or `before:config` |
| `entry` | Yes | Frontend bundle path relative to `dashboard/` |
| `css` | No | CSS asset to inject |
| `api` | No | Python file that exports a FastAPI router |

## Discovery order

The dashboard scans these locations for `dashboard/manifest.json`:

1. `~/.hermes/plugins/<name>/dashboard/manifest.json`
2. Bundled repo plugins in `plugins/<name>/dashboard/manifest.json`
3. `./.hermes/plugins/<name>/dashboard/manifest.json` when project plugins are enabled

## When to use this

Use dashboard plugins when you want:

- custom operational tabs for your own workflows
- local analytics or reporting panels
- project-specific dashboards backed by Hermes session or config data
- internal automation UIs that run beside the standard Hermes dashboard

