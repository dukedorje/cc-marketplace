---
name: status
description: Quick alias for /update-status --show. Displays sprint phase, artifacts created, and epic/story status dashboard.
user-invocable: true
argument-hint: "[--sprint=ID]"
---

# status: Sprint Status Overview

This is a convenience alias for `/update-status --show`.

If the user provided a `--sprint=ID` argument, pass it through: invoke `/update-status --show --sprint=ID`.

Otherwise, invoke `/update-status --show` with no additional arguments.
