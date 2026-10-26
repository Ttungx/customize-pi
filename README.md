# customize-pi

A skill that teaches a coding agent how to work on **Pi** (`pi.dev`, the
`@earendil-works/pi-coding-agent` CLI): settings, MCP servers, codemode,
extensions, skills, prompt templates, themes, Pi packages, models and auth,
keybindings, project trust, sessions, and Windows/terminal setup.

The skill's `version` tracks the Pi release it documents. This checkout is
**1.4.0**; a mismatch with `pi --version` is the signal to re-verify details
before editing someone's configuration.

## Install

Copy the directory into a skills location:

```bash
# user scope
cp -r customize-pi ~/.pi/agent/skills/

# or project scope
cp -r customize-pi <project>/.pi/skills/
```

Or install it as a Pi package straight from git:

```bash
pi install git:github.com/Ttungx/customize-pi@v1.4.0
```

Then run `/reload` and check that `customize-pi` appears in the startup
resource listing (or in `pi config`).

## Use

The skill triggers on Pi configuration work. You can also force it:

```text
/skill:customize-pi add a project MCP server for the docs site and keep it hidden from other projects
```

## Layout

```text
customize-pi/
├── SKILL.md                        # routing, workflow, invariants
└── references/
    ├── configuration.md            # paths, scope, trust, settings groups, reload, env vars
    ├── mcp-and-codemode.md         # mcp.json, exposure, OAuth, codemode API and limits
    ├── resources.md                # skills, prompt templates, themes, packages
    ├── runtime.md                  # models, models.json, auth, classifiers, extensions
    └── security-and-windows.md     # trust/isolation, Windows shells, terminal, keys, sessions
```

`SKILL.md` stays short on purpose: the metadata is always advertised, the body
loads on demand, and the reference files load only when a change needs their
detail.

## Maintenance

1. `pi --version` and read `<pi-package>/CHANGELOG.md` for the new release.
2. Diff the changed areas against `<pi-package>/docs/*.md` and
   `<pi-package>/schemas/*.schema.json` (also published at
   `https://pi.dev/schemas/`).
3. Update the affected reference file first, then `SKILL.md` if a default,
   command, or invariant changed.
4. Set the `version` entry under `metadata` in `SKILL.md` and `version` here,
   add a `CHANGELOG.md` entry, and tag the release to match Pi. Validate with
   skill-creator's `quick_validate.py`, which rejects unknown top-level keys.
