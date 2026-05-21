---
name: dissect
description: "Triggered when the user types `/dissect` or `/dissect <feature>`. Without argument: scans the entire codebase and lists all detected feature modules. With argument (e.g. `/dissect login`): extracts all code related to that feature, traces call chains, and outputs a structured report by architectural layer. The project must be available locally (Claude runs bash commands from the project root)."
---

# /dissect — Feature Code Extractor

## Overview

Two modes depending on whether a feature name is provided:

| Command | Behavior |
|---|---|
| `/dissect` | Scan the whole project and **list all features** |
| `/dissect <feature>` | **Extract all code** for that specific feature |

---

## MODE A — No argument: List all features

When the user runs `/dissect` with no argument, scan the project and produce a feature map.

### A1 — Get file tree

```bash
find . -type f \
  -not -path '*/node_modules/*' \
  -not -path '*/.git/*' \
  -not -path '*/dist/*' \
  -not -path '*/build/*' \
  -not -path '*/__pycache__/*' \
  -not -path '*/vendor/*' \
  | sort
```

### A2 — Read key entry points

Focus on files that typically declare features: routes, controllers, main files, API definitions. Read them to understand what the project does.

### A3 — Output feature list

```
# Project Feature Map
Generated from: [project root]

## Core Features
- auth / login
- user management
- ...

## Supporting Modules
- logger
- config
- error handling
- ...

## External Integrations
- stripe
- s3
- redis
- ...

---
> Run `/dissect <feature>` to extract the full code for any feature above.
```

---

## MODE B — With argument: Extract feature code

When the user runs `/dissect <feature>`, follow Steps 0–6 below.

---

## Step 0 — Confirm inputs

Before starting, confirm with the user:
- **Feature name**: e.g. `login`, `auth`, `payment`, `upload`
- **Project root**: default is current working directory
- **Language/framework** (if not obvious): helps pick the right search strategy

---

## Step 1 — Understand the project structure

```bash
find . -type f \
  -not -path '*/node_modules/*' \
  -not -path '*/.git/*' \
  -not -path '*/dist/*' \
  -not -path '*/build/*' \
  -not -path '*/__pycache__/*' \
  -not -path '*/vendor/*' \
  | sort
```

Also check for config files that reveal the stack:
```bash
ls package.json requirements.txt go.mod pom.xml Cargo.toml 2>/dev/null
```

---

## Step 2 — Build keyword list from feature name

Given feature name (e.g. `login`), expand into variants:

| Feature | Keywords to search |
|---|---|
| login | `login`, `signin`, `sign_in`, `authenticate`, `auth`, `session`, `token`, `jwt`, `passport` |
| register | `register`, `signup`, `sign_up`, `createUser`, `newUser` |
| payment | `payment`, `pay`, `checkout`, `stripe`, `billing`, `invoice`, `order` |
| upload | `upload`, `file`, `multipart`, `s3`, `storage`, `blob` |
| search | `search`, `query`, `filter`, `elasticsearch`, `index` |

**For unknown features**, derive keywords from the feature name itself (camelCase, snake_case, kebab-case variants) and ask Claude to suggest related terms based on the codebase context.

---

## Step 3 — Search for relevant files

```bash
grep -rl "login\|signin\|authenticate\|jwt\|session" . \
  --include="*.js" --include="*.ts" --include="*.jsx" --include="*.tsx" \
  --include="*.py" --include="*.go" --include="*.java" --include="*.rb" \
  --include="*.php" --include="*.cs" \
  --exclude-dir=node_modules --exclude-dir=.git --exclude-dir=dist \
  --exclude-dir=build --exclude-dir=vendor
```

Also search config and route files specifically:
```bash
grep -rl "login\|auth" . \
  --include="*.json" --include="*.yaml" --include="*.yml" --include="*.env*" \
  --exclude-dir=node_modules --exclude-dir=.git
```

---

## Step 4 — Read and analyze each matched file

For each file found in Step 3:

```bash
cat <filepath>
```

Identify which parts are relevant:
- Function/class definitions that match the feature
- Route handlers (e.g. `POST /login`, `router.post('/auth')`)
- Middleware related to the feature
- Database queries/models
- Utility/helper functions called by the above
- Config values (e.g. JWT secret, session options)

**Do not include** unrelated functions that happen to be in the same file.

---

## Step 5 — Trace the call chain (important!)

Feature code is often spread across layers. After finding entry points, trace:

```
Route handler → Service/Controller → Model/DB → Utility/Helper
```

For each function found, check if it calls other functions and follow them:

```bash
grep -rn "findUserByEmail\|exports.findUserByEmail\|def findUserByEmail\|func FindUserByEmail" . \
  --exclude-dir=node_modules --exclude-dir=.git
```

Repeat until the full chain is mapped.

---

## Step 6 — Output structured report

```
# Feature Extraction Report: [FEATURE NAME]
Generated from: [project root path]

## Summary
- Total files involved: N
- Layers: Route → Controller → Service → Model → Util

## File Map
[list showing which file handles which responsibility]

---

## [1] Route / Entry Point
File: src/routes/auth.js (lines 12–45)
[code block]

## [2] Controller / Handler
File: src/controllers/authController.js (lines 8–67)
[code block]

## [3] Service / Business Logic
File: src/services/authService.js (lines 1–89)
[code block]

## [4] Model / Database
File: src/models/User.js (lines 34–56)
[code block]

## [5] Utilities / Helpers
File: src/utils/jwt.js (full file)
[code block]

## [6] Config / Environment
File: config/auth.js
[code block]

---

## Dependencies
External packages used by this feature:
- bcrypt (password hashing)
- jsonwebtoken (token generation)
- express-session (session management)

## Notes
- [any circular dependencies, TODOs, or gotchas found]
```

---

## Tips & Edge Cases

**Monorepos**: run the search from the relevant package subdirectory.

**Generated files**: skip `*.min.js`, `*.d.ts`, lockfiles.

**Test files**: include them in a separate `## Tests` section.

**Large files (>500 lines)**: extract only relevant functions by line range, don't dump the whole file.

**Ambiguous feature names**: if matches exceed 100 files, ask the user to narrow down.

**Multiple languages**: extract both backend and frontend sides, label clearly.

---

## Trigger & Usage

```
/dissect              → list all features in the project
/dissect login        → extract all login-related code
/dissect payment      → extract all payment-related code
/dissect file-upload  → extract all upload-related code
```

---

## Step 7 — Write extracted code to .dissect/<feature>/ folder

After generating the report, write all extracted code to disk so the user can migrate it to another project.

### Folder structure

```
.dissect/
└── <feature>/
    ├── index.md          ← migration guide (always create this first)
    └── <flattened files> ← one file per source file involved
```

File naming rule: replace path separators with `_` and keep the original extension.
- `src/index.ts` → `src_index.ts`
- `frida/hook.js` → `frida_hook.js`
- `frida/config/addresses.19823.json` → `frida_config_addresses.19823.json`

### File contents

- **Whole file involved**: write the complete original file, add a comment at the top:
  ```
  // dissect: full file — <original path>
  ```
- **Partial file (only some functions extracted)**: write only the relevant code blocks, add a comment at the top:
  ```
  // dissect: extracted from <original path> (lines X–Y)
  // ⚠ This is a partial extract. Do not use as-is without the rest of the file.
  ```

### index.md format

```markdown
# Dissect: <feature>
Extracted from: <project root>
Date: <today>

## Files
| File in this folder | Original path | Type |
|---|---|---|
| src_index.ts | src/index.ts | partial extract (lines 139–246) |
| frida_hook.js | frida/hook.js | full file |
| frida_config_addresses.19823.json | frida/config/addresses.19823.json | full file |

## How to migrate
1. Copy the files to the target project following the Original Path column.
2. Install dependencies: `npm install <dep1> <dep2>`
3. [any wiring or config changes needed]

## Gotchas
- [list from Step 6 Notes, copy here]
```

### Bash commands to create the folder and files

```bash
mkdir -p .dissect/<feature>
```

Then for each file, use:
```bash
cat > .dissect/<feature>/<flattened_name> << 'ENDOFFILE'
<file content>
ENDOFFILE
```

Write `index.md` last, after all code files are in place.

### After writing

Tell the user:
```
✅ Extracted to .dissect/<feature>/
   <N> files written. See .dissect/<feature>/index.md for migration guide.
   Copy the entire folder to your target project to get started.
```