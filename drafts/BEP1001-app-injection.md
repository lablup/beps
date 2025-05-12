---
Author: Joongi Kim (joongi@lablup.com)
Status: Draft
---

# App Injection

## Motivation

* Decouple the update cycle of container apps (e.g., Jupyter) and the container images (e.g., NGC PyTorch)
* Allow users to pin a desired version of the apps (e.g., Jupyter Notebook v7 has many breaking changes and most nbextensions do not work)

## Proposed structure

### App Package

- `service-def.json` (mounted as `/opt/kernel/service-defs/{app-name}.json`)
- `app/` (mounted as `/opt/apps/{app-name}/`)

### Updates to Manager API

- List the available app packages 
- Auto-populate required app packages into agent nodes when creating sessions and kernels  
  (or, we need to assume that we have consistent app package files in all agent nodes...)
- Specify the app packages when creating sessions
- Perform a compatibility check by comparing some metadata  
  (e.g., is it a static build or does it depend on a specific glibc version?)

### Updates to Agent API

- TODO

## Migration of Existing Apps

### Overall technical issues

- If we mount the extracted app directories as _read-only_, some apps may not work as expected.
- If the app depends on directories that only root can write to, it may not work as expected.
- The delay required to load and extract the app packages
- Managing the cached extracted app packages within the disk size limit of agents
- Should we ship PBS (python-build-standalone) individually by each app?

### Jupyter

Recently Jupyter has changed its configuration format. (TODO: issue link)

We need to generate version-specific `.cfg` file, by replacing the service definition
to execute an (injected) script instead of directly writing the configuration file from the embedded template string.

**Design discussion:**
- Make a wheelhouse directory and install them when first-launching the app  
  _vs._ Make a pre-installed site-packages directory and mount subdirs into `/opt/backend.ai`?

### VSCode

- Could we make a statically built app package?
- Could we allow users to install their own extensions in container while the app package is read-only?
