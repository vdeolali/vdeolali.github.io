---
title: "All Setup"
date: 2024-03-22
---

# All Setup

I wanted a quick way to answer a very specific OCI question: if I need a given compute shape in a given region, can I actually get capacity there right now?

That is what `oci-cap-check` does. The repository is a Flask application that turns an OCI capacity-report workflow into a browser-based tool. Instead of running a long CLI flow manually every time, I can open a page on `localhost:5000`, connect to a tenancy, choose a shape and region, and get back a formatted capacity report.

## What I was trying to solve

Capacity checks in OCI are rarely just about one value. In practice, I usually need to know:

- which subscribed regions are reachable
- which availability domains and fault domains should be checked
- whether a shape needs extra configuration such as OCPUs or memory
- whether I am querying at tenancy level or a specific compartment
- whether I want the raw availability state only or the available count as well

That makes even a simple "do I have capacity?" check more than a single API call.

The implementation in `oci-cap-check` is built around that reality. The app keeps the browser UI simple, but the backend still does the same OCI work I would otherwise script by hand.

## Architecture at a glance

The repository is split into a few focused modules:

- `app.py` handles the Flask routes and page rendering
- `oci_runner.py` runs the reporting workflow and captures terminal-style output
- `modules/identity.py` handles authentication, region subscription lookup, compartment selection, and AD/FD discovery
- `modules/capacity.py` builds and submits compute capacity report requests
- `templates/index.html` provides a two-step UI for connect-then-run

That separation is useful because the web app is really a thin layer over a reusable OCI reporting flow.

## The web flow

The browser flow is intentionally split into two stages.

### Step 1: Connect and load tenancy data

The first form asks for:

- OCI config profile
- full OCI config file path

The current UI defaults to config-file based authentication, which maps to `user_auth=cf`.

When I click **Connect & Load Tenancy Data**, `app.py`:

1. calls `init_authentication()`
2. creates an OCI identity client
3. loads the tenancy's subscribed regions
4. loads the available compute shapes

That data is then used to populate the dropdowns for the second form.

This is a good design choice because it prevents the user from trying to run a capacity report before the tenancy connection is actually proven to work.

### Step 2: Run the capacity report

Once the tenancy data is loaded, the second form appears. It lets me choose:

- shape
- target region or all subscribed regions
- OCPUs for Flex shapes
- memory for Flex shapes
- optional compartment OCID
- tenancy-level admin mode
- DRCC mode

Submitting that form calls `run_capacity_check()` in `oci_runner.py`, which runs the full backend workflow and returns the report as text to display in the page.

## How authentication works

One of the more useful parts of the repo is the authentication logic in `modules/identity.py`.

The backend supports multiple OCI authentication paths:

- Cloud Shell delegation token
- local OCI config file
- instance principals

For the web UI, the main path is config-file authentication. The code loads the config profile from the supplied file path, validates it, builds an OCI signer, and then confirms the login by fetching tenancy information through `IdentityClient`.

That validation step matters. It means the app does not just trust that the file exists. It verifies that the credentials are actually usable against OCI before it proceeds.

If every auth method fails, the underlying module still has a retry path, which comes from the original script-oriented design. In the browser flow, the main value is that the authentication errors are surfaced clearly instead of failing later during the capacity call.

## Region handling

After authentication, the next problem is deciding what to check.

`get_region_subscription_list()` in `modules/identity.py` handles three cases:

- no region specified, which falls back to the home region
- a specific subscribed region
- `all_regions`, which expands to every subscribed region in the tenancy

That is then followed by `validate_region_connectivity()`, which uses a `ThreadPoolExecutor` to test region connectivity concurrently.

I like this part of the implementation because it avoids wasting time on a long serial check across many regions. Each region is validated by creating an identity client in that region and confirming the tenancy is reachable there. Regions that fail are reported and skipped.

In other words, the app distinguishes between:

- regions that exist
- regions the tenancy is subscribed to
- regions that are actually reachable for the current credentials

That is the right way to build a useful OCI capacity tool.

## Shape handling and why it matters

Compute shapes in OCI are not all configured the same way, so a capacity checker cannot treat them all uniformly.

That logic lives in `modules/capacity.py`.

The implementation handles a few important cases:

- bare metal shapes
- standard Flex shapes
- DenseIO Flex shapes
- special handling for `VM.Standard.A2.Flex`

For fixed shapes and bare metal, the code can often rely on the shape defaults returned by OCI.

For ordinary Flex shapes, the app clamps the requested OCPUs and memory to the limits allowed by the shape metadata.

For DenseIO Flex shapes, the code uses predefined valid configurations. That is important because those shapes are not just "pick any OCPU and memory combination." They need a valid mapping that also accounts for NVMe layout.

That extra logic is one of the places where the repository stops being a thin UI wrapper and becomes a genuinely useful implementation.

## How the capacity report is built

The core OCI call is `create_compute_capacity_report()`.

For each validated region, the backend:

1. switches the OCI client config to that region
2. fetches availability domains
3. fetches fault domains inside each AD
4. derives the correct shape configuration
5. builds `CreateComputeCapacityReportDetails`
6. submits a capacity report request for the shape and fault domain

The report is then printed in a table-style format that includes:

- region
- availability domain
- fault domain
- shape
- OCPU
- memory
- availability

If DRCC mode is enabled, the output also includes `available_count`.

That gives the app two reporting modes:

- a simpler availability view
- a more detailed count-aware view for DRCC use cases

## Why I kept the output text-based

The report display in the UI is a `<pre>` block, not a rich data grid. That is a pragmatic choice.

The backend already had a print-oriented reporting flow, so `oci_runner.py` uses `redirect_stdout()` to capture that output into an in-memory buffer and return it to the page. That lets the same operational output format work in both script and web contexts.

For a first implementation, that is efficient:

- no need to redesign all reporting output into JSON-first rendering
- easy to preserve familiar script output
- faster to debug because the raw report remains visible

If I wanted to evolve the tool later, the next step would be to return structured data and render it as a sortable HTML table. But for an operator-facing internal tool, the captured text output is a reasonable tradeoff.

## Access model and compartment handling

Another useful design point is that the app does not assume tenancy-wide admin access.

`set_user_compartment()` in `modules/identity.py` lets the workflow run in either of these modes:

- tenancy-level if the user has broader admin rights
- compartment-level if the user only has access to a specific compartment

That matters in real OCI environments where permissions are often scoped narrowly. The app also validates compartment state before running the report, which avoids misleading failures later in the flow.

## What I think works well

After reading through the repo, a few implementation choices stand out as solid:

- The UI is split into a connection step and a reporting step.
- Authentication is validated early.
- Region connectivity is checked concurrently.
- Shape-specific constraints are handled explicitly instead of ignored.
- Availability domains and fault domains are expanded automatically.
- The tool supports both tenancy-level and compartment-level usage.

Those are the parts that make this more than just a demo Flask form.

## What I would improve next

If I continue building on `oci-cap-check`, the next improvements are fairly clear:

- replace the text report with structured HTML table rendering
- remove leftover interactive CLI prompts from shared modules when running in web mode
- add stronger UI validation for Flex fields
- let the user save common report presets
- export results as CSV or JSON
- add pagination or filtering if the result set gets large across many regions

The current version already proves the main idea, though: OCI capacity checks can be wrapped in a simple browser workflow without losing the important tenancy, region, compartment, and shape logic underneath.

## Final thoughts

`oci-cap-check` is essentially an operational script promoted into a small web application.

That is why the implementation works well. It did not start by chasing a fancy UI. It started with the real OCI tasks that matter:

- authenticate correctly
- discover the right regions
- handle shape configuration properly
- query capacity across ADs and FDs
- show the result in a format that is easy to read

For a tool meant to answer "where can I actually launch this shape?", that is the right level of engineering.
