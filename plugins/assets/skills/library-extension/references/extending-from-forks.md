# Extending from forks

An extension that changes a library's released contract ends by bumping that library's version and raising the pins that
point to it. A lab working from its own forks cannot publish the bumped release, so a raised pin never resolves against
PyPI. This file replaces that release step, the tox gates of each fork that depends on another fork, and the plugin
install from the `Sun-Lab-NBB/sollertia` marketplace. Every other recipe step stays unchanged.

---

## Contents

- Which libraries to fork
- Local version labels
- One environment for every fork
- MCP servers and skills
- Verifying the shared environment
- Keeping the work

---

## Which libraries to fork

A new session type or acquisition system registers through edits inside each library. `SessionTypes` and
`AcquisitionSystems` are enums, and the `sollertia-shared-assets` and `sollertia-forgery` registries are module-level
tables checked at import, so no separate package is able to add a member from outside. Fork `sollertia-shared-assets`,
`sollertia-experiment`, `sollertia-forgery`, and the `sollertia` marketplace, plus `sollertia-micro-controllers` and
`sollertia-virtual-reality` when the extension reaches them, then clone the forks side by side in one parent directory.
Neither of those two carries a PyPI pin, so neither needs a local version label.

---

## Local version labels

You MUST leave each fork's public version and every pin unchanged, and append a local version label to the version of
each edited fork, such as `10.0.0+lab1`. A specifier without a local label ignores the label when it matches, so the
exact pin `sollertia-shared-assets==10.0.0` in `sollertia-experiment` and the range `>=10.0.0,<11` in
`sollertia-forgery` both accept the labeled fork as they stand. The label also tells the fork apart from the PyPI
release in `uv pip list`.

---

## One environment for every fork

You MUST install the three library forks into one shared environment as editable installs, in dependency order:

```bash
mamba create -n sollertia_lab python=3.14 uv tox tox-uv -y
mamba activate sollertia_lab
uv pip install -e ./sollertia-shared-assets -e ./sollertia-experiment -e ./sollertia-forgery
uv pip install pytest
```

An editable install makes every later edit live without a reinstall. The per-repository tox environments never see a
sibling fork, because `tox -e create` installs the dependencies from PyPI, `tox -e install` runs a non-editable
`uv pip install .`, and the test environments build an isolated wheel against PyPI. Run each fork's tests with `pytest`
inside the shared environment.

A remote `sollertia-forgery` batch runs in the server's configured environment, so the `sollertia-shared-assets` and
`sollertia-forgery` forks have to be installed there too.

---

## MCP servers and skills

Each plugin starts its MCP server with the bare `slsa`, `sle`, or `slf` command, resolved on the PATH that Claude Code
inherits at launch. Activate the shared environment and launch Claude Code from it, then confirm with `/mcp` that all
three servers run from the forks. Restart Claude Code after every edit to library code, because a running server keeps
the code it imported at startup.

Remove the `Sun-Lab-NBB/sollertia` marketplace and add the marketplace fork as a directory marketplace. Claude Code
caches installed plugins by version, so an edited skill reaches the agent only after its plugin's `version` is bumped.

---

## Verifying the shared environment

Run every import-time check inside the shared environment, because no other environment holds all three library forks.
Use `python -c "import sollertia_shared_assets"` for the shared-assets import-time checks, `sle --help` for the
Mesoscope-VR session-type check, and `slf --help` for the `sollertia-forgery` import-time checks. Then confirm that
`uv pip list` shows each library fork with its local label and its clone path.

---

## Keeping the work

Commit the extension on a branch in each fork and push it to that fork. A change worth sharing goes upstream as a pull
request against the matching `Sun-Lab-NBB` repository, whose maintainers publish the release replaced by this file.
