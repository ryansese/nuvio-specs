---
paths:
  - "repos/NuvioSwitch/**"
---

# NuvioSwitch (repos/NuvioSwitch)

This file provides guidance to Claude Code (claude.ai/code) when working with code in the NuvioSwitch submodule.

## What this is

NuvioSwitch is a Nintendo Switch homebrew port of NuvioTV: C++ with devkitPro/libnx, the borealis UI library and libmpv. It is the user's own repo (`ryansese/NuvioSwitch`) and has no app code yet.

Unlike the other submodules, it carries its own `CLAUDE.md` (`repos/NuvioSwitch/CLAUDE.md`), which is tracked in that repo and holds the plan, the borealis/CMake build notes and the NuvioTV porting map. Treat that file as the source of truth and keep it updated there; this rule only adds what is specific to working from the workspace.

## Working from the workspace

- `library/borealis` is a nested submodule (`xfangfang/borealis`). A plain clone leaves it empty; run `git submodule update --init --recursive` from the workspace root. Don't edit files inside it.
- The NuvioTV reference checkout is the sibling submodule `repos/NuvioTV` (`../NuvioTV` from inside NuvioSwitch). Read it to port behaviour and data flow, not code, and don't modify it from NuvioSwitch work.
- Changes to NuvioSwitch are commits in its own repo. Getting them into the workspace takes a separate submodule pointer bump commit here.
