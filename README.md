# portal-builder

A starting point for a **Krateo Composable Portal page set** — one or more portal pages shipped as
their own Helm chart, installed like any other Krateo composition.

Create a repository from this one ("Use this template"), rename the chart, replace the example
page, tag a release.

There are two ways a page set gets authored, and they produce different repositories:

| | **Hand-authored** (this template) | **Composed in the Portal Builder** |
|---|---|---|
| repo | you create it from this template | the `BuilderPublish` claim creates it, empty |
| chart | `helm/<your-name>/` | the repo root — the repo *is* the chart |
| `compositiondefinition.yaml` | scaffolded here | **not emitted — you add it** |
| release workflow | scaffolded here | **not emitted — you add it** |

The composer writes the chart and nothing else. `builder-publish` creates the destination
repository for it (`repository.create`, default true) and auto-inits it, so what you get is an empty
repo with a chart in it — no release workflow, no `compositiondefinition.yaml`, and therefore no way
to release or register itself. Copy both out of this template, and point the CompositionDefinition's
`url` at the page set's own chart name.

`builder-publish` *can* seed a destination from a template repo — `source.url` renders a
git-provider `Repo` (`fromRepo` → `toRepo`) that copies this repository in before the held files are
committed, which would make the two columns above converge. The Portal Builder does not set it
today; the composer publishes into a bare repo.

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
| `helm/portal-builder/` | rename the directory to name your section |
| `Chart.yaml` `name:` | **becomes the CRD Kind** — `portal-builder` → `PortalPages`, plural `portalpages` |
| `templates/flex.page-example.yaml` | your page root. The name must be `page-<slug>` exactly |
| `templates/pageheader.example.yaml` | your page header, or delete it |
| `values.schema.json` | **is** the generated CRD's `spec` — no schema, no CRD, not installable |
| `compositiondefinition.yaml` | the `name`, `namespace` and chart `url` |

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
2. **Release** — tag `X.Y.Z`. `.github/workflows/release-tag.yaml` discovers every `Chart.yaml`,
   substitutes `CHART_VERSION`, and pushes each chart to
   `oci://ghcr.io/<owner>/charts/<chart name>:<tag>`. It also stamps `compositiondefinition.yaml`
   with the tag and **attaches it to the GitHub release**.
3. **Register** — apply the stamped `CompositionDefinition` from the release:

   ```bash
   kubectl apply -f https://github.com/<owner>/<repo>/releases/download/<tag>/compositiondefinition.yaml
   ```

   Apply the copy from the **release**, not from the branch — the branch still carries the
   `CHART_VERSION` placeholder, and registering that gets you a chart version that does not exist.

4. **Install** — create a claim of the generated Kind. core-provider generates the CRD from
   `values.schema.json`; the claim's Helm release is what puts the pages on the cluster, and
   deleting the claim removes them.

## Rendering locally

`helm template` refuses the `CHART_VERSION` placeholder, so pass a version:

```bash
helm template example helm/portal-builder --namespace krateo-system --version 0.1.0
```

## Checking your pages

The widget composition rules run from `krateo-platformops/frontend`:

```bash
helm template example helm/portal-builder --namespace krateo-system --output-dir /tmp/r
python3 path/to/frontend/design/lint/lint-portal-consistency.py /tmp/r/portal-builder/templates
```

`P0 (page-discovery-alive)` firing means no page root carries `krateo.io/nav-label` or
`krateo.io/nav-path` — your pages exist but nothing can navigate to them.
