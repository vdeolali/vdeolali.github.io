---
title: "Building Custom OCI Images with Flask and Packer"
date: 2026-04-16
---

# Building Custom OCI Images with Flask and Packer

I recently spent some time reading through my [`oci-image-builder`](https://github.com/vdeolali/oci-image-builder) repository. The project is a small web application that helps build a custom Oracle Cloud Infrastructure image starting from an existing OCI platform image, adding packages, and then handing the actual image creation off to Packer.

What I like about this repo is that it keeps the workflow simple:

- use a Flask UI to collect the image build inputs
- query OCI dynamically for the available profiles, images, and shapes
- store each build request in SQLite
- generate a Packer template from the submitted form data
- launch a Packer build that creates a new image in OCI

That makes it less of a giant platform and more of a focused control plane for one job: turning a repeatable image customization request into an OCI image build.

## What the application does

At a high level, the app gives the user a web form with these fields:

- OCI profile
- base image
- instance shape
- optional Flex shape sizing
- a package list to install

From there, the app creates a build record and starts the image build asynchronously.

The code path is straightforward:

1. The Flask app in [`app.py`](https://github.com/vdeolali/oci-image-builder/blob/main/app.py) renders the build form.
2. The OCI helper functions in [`oci_utils.py`](https://github.com/vdeolali/oci-image-builder/blob/main/oci_utils.py) read the local OCI CLI config and query OCI for profiles, Oracle Linux images, and shapes.
3. The form submission is stored as an `ImageBuild` record through SQLAlchemy in [`models.py`](https://github.com/vdeolali/oci-image-builder/blob/main/models.py).
4. A background thread calls [`run_packer_build`](https://github.com/vdeolali/oci-image-builder/blob/main/packer_utils.py) to generate a JSON template and execute `packer build`.
5. The build status and captured output are written back to the database.

That separation is clean enough that you can understand the system quickly:

- Flask handles the UI and request lifecycle
- OCI SDK calls handle discovery
- SQLAlchemy tracks state
- Packer performs the actual image creation

## The UI flow

The HTML templates are basic Bootstrap templates, which is fine for this type of tool. The important part is the user flow in [`templates/index.html`](https://github.com/vdeolali/oci-image-builder/blob/main/templates/index.html).

When the page loads, JavaScript calls two Flask API endpoints:

- `/api/images/<profile_name>`
- `/api/shapes/<profile_name>`

Those endpoints return JSON that populates the dropdowns based on the selected OCI profile. That means the form is not hardcoded to a single environment. Instead, it uses the OCI configuration already present on the system and lets the user choose which profile to authenticate with.

The shape handling is also practical. If the selected shape contains `Flex`, the page reveals fields for:

- `ocpus`
- `memory_in_gbs`

That is a small touch, but it matches how OCI actually works with Flex shapes and avoids forcing those parameters for fixed-size shapes.

## How the OCI integration works

The OCI-specific logic lives in [`oci_utils.py`](https://github.com/vdeolali/oci-image-builder/blob/main/oci_utils.py).

There are three main helpers:

- `get_oci_profiles()` parses the local OCI config file and returns the available profile names.
- `get_oci_images()` uses the OCI Python SDK to list available Oracle Linux images in the configured compartment.
- `get_available_shapes()` lists the shapes available in the compartment.

This design makes the tool tenancy-aware without forcing the user to manually paste long OCIDs into the form every time.

A few implementation details stood out to me:

- the OCI config file path is taken from `~/.oci/config`
- the compartment, subnet, and availability domain are loaded from environment variables
- the image list is filtered to `operating_system='Oracle Linux'`

That last point is important. Right now the app is opinionated toward Oracle Linux based image builds, which is a reasonable place to start for OCI-focused automation.

## How the build request is persisted

The model is intentionally small. The `ImageBuild` table stores:

- cloud provider
- OCI profile
- base image
- package list
- shape
- Flex sizing values when applicable
- status
- captured Packer output
- submission timestamp

For a tool like this, SQLite is a sensible default. It keeps the application lightweight and easy to run locally without requiring an external database just to track a build queue.

The app also forces the database path into the local `instance` directory in [`app.py`](https://github.com/vdeolali/oci-image-builder/blob/main/app.py), which avoids some of the common relative-path confusion that often shows up in simple Flask projects.

## How the Packer template is generated

The most important backend logic is in [`packer_utils.py`](https://github.com/vdeolali/oci-image-builder/blob/main/packer_utils.py).

When a build starts, the code:

1. splits the submitted package list into lines
2. converts those package names into a `yum install -y` command
3. loads the selected OCI profile from the local OCI config
4. extracts the authentication fields Packer needs
5. builds a JSON document for the `oracle-oci` Packer plugin
6. writes that JSON to a temporary `.pkr.json` file
7. runs `packer build -force`

The generated provisioner is simple and clear:

```json
"inline": [
  "sudo yum update -y",
  "sudo yum install -y ..."
]
```

That means the repository is currently optimized for package-based customization instead of a more general provisioning pipeline with shell scripts, Ansible, or cloud-init fragments. For many internal image-builder use cases, that is enough to be immediately useful.

Another detail I liked is that the code injects the OCI authentication fields directly into the Packer source block rather than depending on external shell state at build time. That reduces ambiguity around which OCI profile is actually being used.

## Why the async build model makes sense

The application starts the Packer work in a background thread after saving the build request. That is the right tradeoff for a simple Flask app because image creation is slow and should not block the request thread while the browser waits for a response.

The build history view in [`templates/builds.html`](https://github.com/vdeolali/oci-image-builder/blob/main/templates/builds.html) then gives a simple status table showing:

- build ID
- cloud
- base image
- status
- submission time

It is intentionally minimal, but it is enough to confirm that a request was queued and whether it completed or failed.

## What I think this project does well

After reading the code, I think the repo does a few things particularly well:

- It keeps the problem scope tight.
- It uses OCI discovery instead of hardcoded dropdown values.
- It records build state in a way that is easy to inspect.
- It generates Packer configuration programmatically from user input.
- It supports Flex shapes, which matters in OCI.

Most importantly, it captures a workflow that many teams eventually want: "start from a known OCI image, add the packages we care about, and produce a reusable custom image."

## Where I would extend it next

The current code works as a solid first version, but reading it also made the next set of improvements pretty obvious:

- filter shapes based on base image compatibility instead of listing all shapes
- support more provisioning options than package installation alone
- add stronger validation around package input and numeric Flex fields
- surface the packer output in the UI instead of only storing it in the database
- move background execution to a real task queue if concurrency grows
- add migrations for schema changes instead of relying on ad hoc database updates

None of those points take away from the value of the current implementation. They are mostly signs that the repo has reached the stage where a working prototype can evolve into a more durable internal tool.

## Final thoughts

The main idea behind `oci-image-builder` is practical: wrap OCI discovery and Packer execution in a small web interface so a repeatable image build becomes easier to launch and track.

That combination works well here:

- Flask provides a lightweight front end
- the OCI SDK supplies tenancy-aware discovery
- SQLAlchemy keeps build metadata
- Packer performs the actual image creation

For anyone building custom Oracle Linux based images on OCI, this is a useful pattern. You do not need a large image factory platform to get value. A small app with the right boundaries can already provide a repeatable and understandable image build workflow.
