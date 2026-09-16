# portal-page-template

A starting point for a **Krateo Composable Portal page set** — one or more portal pages shipped as
their own Helm chart, installed like any other Krateo composition.

Scaffold a new repository from this one (the Portal Builder does it for you via the publish form's
*Scaffold from* field), rename the chart, replace the example page, tag a release.

## Why a page set, and not one chart per page

Every `CompositionDefinition` registered on a cluster spawns its own `composition-dynamic-controller`
Deployment — **50m CPU and 128Mi of memory requests, one replica**. A portal page is static YAML
with nothing to reconcile, so a controller per page buys no capability and costs real resource.

Group related pages into one chart, the way `portal-agents` carries the whole agent control plane.
Publishing *into an existing page chart* should be the common case; a new chart is a deliberate
choice for a genuinely separate section.

## What you must change

| | |
|---|---|
| `helm/portal-pages/` | rename the directory to name your section |
| `Chart.yaml` `name:` | **becomes the CRD Kind** — `portal-pages` → `PortalPages`, plural `portalpages` |
| `templates/flex.page-example.yaml` | your page root. The name must be `page-<slug>` exactly |
| `templates/pageheader.example.yaml` | your page header, or delete it |
| `values.schema.json` | **is** the generated CRD's `spec` — no schema, no CRD, not installable |

Leave `version: CHART_VERSION` alone: the release workflow substitutes it from the git tag.

## The sidebar is declared on the page, not in the portal

The portal's Menu lists page roots across the cluster and builds its entries from their annotations.
Installing this chart adds the pages to the nav; uninstalling removes them. **No edit to the portal
chart, and no values flag to keep in sync with what is installed.**

| Annotation | Meaning |
|---|---|
| `krateo.io/nav-path` | the route. Absent = not a page |
| `krateo.io/nav-label` | the sidebar entry. Absent = route-only (deep-linkable, hidden) |
| `krateo.io/nav-icon` | Font Awesome name, e.g. `fa-gauge` |
| `krateo.io/nav-order` | sort key across the whole sidebar — 1–40 is the portal's own; authored pages conventionally sit at 900+ |
| `krateo.io/nav-group` | the section. Dividers are synthesised where the group changes |
| `krateo.io/nav-alias` | extra routes onto this page with **no** sidebar entry of their own, comma-separated |

Two rules the CRDs enforce and nothing defaults, both of which fail quietly rather than loudly:

- a container must list the **plural** of every kind it holds in `widgetData.allowedResources`, or
  the child renders as nothing;
- every `resourcesRefs` entry needs a **namespace** — snowplow's resolver has no defaulting, so an
  entry without one resolves against the empty namespace and the child never appears.

## From repo to installed page

1. **Publish** — commit the chart. The Portal Builder does this through a `BuilderPublish` claim,
   which creates the repository if it does not exist (existing ones are adopted, not re-created)
   and opens a change request.
2. **Release** — tag `X.Y.Z`. `.github/workflows/release-oci.yaml` calls the org-wide reusable
   workflow, which substitutes `CHART_VERSION` and pushes to
   `oci://ghcr.io/krateo-platformops/charts/<chart name>`.
3. **Register** — in the portal, install it the way a blueprint is installed: the form creates a
   `CompositionDefinition` pointing at the chart URL and version. core-provider generates the CRD
   from `values.schema.json`; a claim of that Kind installs the pages.

## Rendering locally

`helm template` refuses the `CHART_VERSION` placeholder, so pass a version:

```bash
helm template example helm/portal-pages --namespace krateo-system --version 0.1.0
```

## Checking your pages

The widget composition rules run from `krateo-platformops/frontend`:

```bash
helm template example helm/portal-pages --namespace krateo-system --output-dir /tmp/r
python3 path/to/frontend/design/lint/lint-portal-consistency.py /tmp/r/portal-pages/templates
```

`P0 (page-discovery-alive)` firing means no page root carries `krateo.io/nav-label` or
`krateo.io/nav-path` — your pages exist but nothing can navigate to them.
