---
title: "OCI Maintenance Utility"
date: 2023-10-12
---

# OCI Maintenance Utility

One of the repetitive operational tasks in OCI is dealing with instances that have been marked for maintenance reboot. The hard part is usually not the reboot itself. It is figuring out which instances are affected, grouping them into a rollout plan, and then executing those reboots in a controlled way.

That is exactly what `oci-maintenance` is built to do.

The repository is intentionally small. It is a single Python script, `maint.py`, backed by the OCI Python SDK. But the script captures a practical maintenance workflow:

- discover all instances flagged for maintenance reboot
- spread them into maintenance pools
- reboot one pool at a time using `REBOOT_MIGRATE`
- support dry runs before changing anything

## What problem the script is solving

If I have a fleet spread across multiple subscribed OCI regions, a maintenance event can affect more than one place at once. I do not want to reboot everything immediately, and I do not want to sort instance lists manually every time.

I want a flow that answers three questions cleanly:

1. Which instances are currently marked for maintenance reboot?
2. How can I split them into batches for staged execution?
3. How do I reboot only the batch I am ready to move?

The design of `oci-maintenance` follows that exact sequence.

## Structure of the repository

The repo is simple enough that the entire implementation lives in one file:

- `maint.py`

That script is structured around a few focused functions:

- `get_config()` loads OCI config and authentication details
- `get_subscribed_regions()` discovers the tenancy's subscribed regions
- `search_instances_needing_reboot()` finds affected instances using OCI Resource Search
- `list_instances()` prints the currently affected instances
- `assign_pools()` applies a `maintenance_pool` freeform tag
- `reboot_pool()` triggers `REBOOT_MIGRATE` for one selected pool

The script is then exposed through a small CLI built with `argparse`.

## Authentication and config handling

The first thing the script does is load OCI credentials from a config file.

`get_config()`:

- expands `~` in the config path
- loads the selected profile
- extracts the configured `pass_phrase` if present
- returns both the config and the passphrase

The script then passes that configuration into the OCI SDK clients. One useful detail here is that the script explicitly accounts for encrypted private keys by reading the passphrase from the OCI config profile.

That is a pragmatic choice. A lot of small operational scripts fail the moment the key is encrypted or the config path is not the default one. This implementation makes those assumptions visible and configurable from the CLI.

The CLI supports:

- `--config` for a custom OCI config path
- `--profile` for a non-default profile
- `--dry-run` for safe previews

## How affected instances are discovered

The core discovery step is in `search_instances_needing_reboot()`.

Instead of iterating through every instance with a compute list call and then filtering in code, the script uses OCI Resource Search with a structured query:

```text
query instance resources where timeMaintenanceRebootDue > 'Now'
```

That is a strong implementation choice because it lets OCI do the filtering. The script only receives the instances that matter for the maintenance workflow.

The search result gives the script the data it needs for the next steps:

- display name
- instance OCID
- availability domain
- compartment ID
- freeform tags

## Multi-region coverage

After loading credentials, the script uses `IdentityClient.list_region_subscriptions()` to retrieve all subscribed regions in the tenancy.

That region list is then used in each of the main workflows:

- listing instances
- assigning pools
- rebooting a pool

For each region, the script creates a region-specific client configuration and runs the maintenance search there.

That matters because maintenance actions are often distributed across regions, and the script is clearly designed to work at tenancy scope instead of assuming a single-region environment.

## Listing instances that need reboot

The `list` command is the read-only entry point:

```bash
python maint.py --profile CVM list
```

`list_instances()`:

1. loads the OCI config
2. gets the subscribed regions
3. searches each region for instances where `timeMaintenanceRebootDue > 'Now'`
4. collects those results into a single list
5. prints a readable table

The printed output includes:

- region
- instance name
- OCID
- availability domain
- current maintenance pool tag

That last value is important because it lets the listing serve two purposes:

- identify maintenance candidates
- confirm how they are currently staged

## Assigning maintenance pools

The next operational step is controlled rollout.

That logic lives in `assign_pools()`.

The command takes a pool count:

```bash
python maint.py --profile CVM --dry-run assign-pools --num-pools 3
```

The implementation does a few useful things:

- gathers all affected instances across regions
- sorts them by OCID for deterministic ordering
- assigns pool numbers in round-robin fashion
- writes the result into the `maintenance_pool` freeform tag

The round-robin assignment is simple:

```python
new_pool = str((i % num_pools) + 1)
```

That means if I choose three pools, the script will distribute the affected instances across pool `1`, `2`, and `3` in a stable sequence.

The tagging model is also practical because it stores rollout state directly on the instance as metadata. Once the tags exist, the reboot step can use them without maintaining a separate inventory file.

### Why dry-run matters here

`assign_pools()` has a dry-run branch that prints the assignments without updating any tags.

That is exactly what I want for a maintenance script. Before changing anything, I can see:

- which instances would be placed in which pool
- whether an instance is already in the expected pool
- whether the chosen pool count produces a sensible distribution

For operational tooling, dry-run support is one of the highest-value safety features, and this script includes it in the right places.

## Rebooting one pool at a time

Once pools are assigned, `reboot_pool()` handles the execution step.

Its job is:

1. search the affected instances again across subscribed regions
2. filter only the instances whose `maintenance_pool` tag matches the requested pool number
3. issue an OCI instance action of `REBOOT_MIGRATE`

Example usage:

```bash
python maint.py --profile CVM reboot-pool --pool 2
```

Or safer:

```bash
python maint.py --profile CVM --dry-run reboot-pool --pool 2
```

That workflow is a good operational pattern because it makes the reboot stage explicit and auditable. The maintenance pool tag is the handshake between planning and execution.

## The actual OCI action used

The script does not perform a generic reboot. It specifically calls:

```python
compute_client.instance_action(inst['ocid'], 'REBOOT_MIGRATE')
```

That matters because the whole utility is about OCI maintenance-related reboot migration, not arbitrary instance restarts.

So the implementation aligns with the operational intent:

- discover instances flagged for maintenance
- use the maintenance-specific reboot action

## Error handling and observability

The script is direct rather than elaborate, but it does include useful debug output.

It prints:

- config path being used
- profile being loaded
- whether a passphrase was found
- region-specific client initialization
- success or failure of update and reboot operations

For small operations tooling, that is often enough. The script is not trying to be a full logging platform. It is trying to be debuggable when run by an operator.

It also catches `ServiceError` around the mutating OCI operations so an update or reboot failure for one instance is reported clearly.

## What I think works well in this implementation

After reading the code, a few design choices stand out:

- OCI Resource Search is used to discover only the instances that matter.
- Region coverage is handled automatically through subscribed region discovery.
- Pool assignment is deterministic because the instance list is sorted.
- Rollout state is stored as a freeform tag on the instance.
- Dry-run exists for both assignment and reboot flows.
- The reboot action is maintenance-specific rather than generic.

That gives the script a clear operational lifecycle:

- observe
- stage
- execute

## Where I would extend it next

If I were taking this further, the next improvements would probably be:

- add concurrency for tag updates and reboot actions
- add CSV or JSON export for the list command
- include compartment name in the printed report
- add filters for region, AD, or tag scope
- add a summary count per pool before execution
- add retry behavior or waiter logic for reboot tracking

None of those are required for the current script to be useful. The existing implementation already does the main job well.

## Final thoughts

`oci-maintenance` is a good example of a small operational utility that solves a concrete problem without overengineering it.

It does not try to be a full fleet management platform. Instead, it focuses on a narrow and valuable maintenance workflow:

- find the instances OCI says need reboot migration
- group them into staged pools
- execute those reboots one pool at a time

For a script that lives in a single file, that is a solid implementation boundary.
