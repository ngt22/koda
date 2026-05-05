# koda-cli

A **terminal launcher, snippet store, and cross-machine clipboard**. Save commands, config templates, notes, and any text to SQLite; retrieve and execute them instantly — by index, shortcut, or fuzzy search. Sync via a private Git repository to share the same store across machines. Built with Python, Typer, and Rich.

## Features

- **Launcher**: Run any saved command with `exec` or `pick`, with variable substitution at call time.
- **Command substitution**: Embed stored values directly in any command — `ssh $(kr bastion)`, `tail -f $(kr log-path)`.
- **Fast save and recall**: `add`, `list`, `show`, `edit`, `pick`, `remove` — all with one-letter aliases.
- **Flexible input**: Arguments, heredocs, pipes, or `$EDITOR`.
- **Shortcuts**: Assign a memorable string alias to any entry and use it in place of a numeric index.
- **Variable substitution**: Expand `${KEY}` and `$1 $2 ...` placeholders at recall time with `-V`.
- **Shell-friendly output**: `raw` prints body-only text for pipes, `eval`, and scripts.
- **Tags**: Classify, filter, and batch-edit entries with multiple tags.
- **Display index**: Stable `uid` (SHA1 short hash) plus user-controlled `idx`. Reorder with `move`/`swap`; close gaps with `compact`.
- **Terminal clipboard**: Save any text with `koda a`, recall it instantly with `koda r` or `koda s` — paste into prompts, commands, or scripts without retyping.
- **Cross-machine sync**: Push and pull via a private Git repository — the same store is available from every terminal, on every machine.
- **XDG-friendly**: Data under `~/.local/share/koda/`, config under `~/.config/koda/`.
- **Configurable defaults**: Persist preferences in `~/.config/koda/config.toml`.

## In action

**① A 60-character command becomes two tokens:**

```bash
# Save the verbose command once
koda a "docker run --rm -it -v \$(pwd):/app -w /app -p 8080:8080 node:20-alpine sh" \
  -t docker -s devbox

# From now on, just:
koda x 1        # run by index
koda x devbox   # or by shortcut
```

**② A deep path lives in one place, works everywhere:**

```bash
# Save it once
koda a "/home/deploy/apps/myservice/logs/app.log" -t log -s app-log

# Embed in any command with $()
tail -f $(koda r app-log)
grep "ERROR" $(koda r app-log) | tail -20
less +F $(koda r 2)   # same thing, by index
```

---

**③ One template, any target — variable substitution at call time:**

```bash
# Save a deploy command with a named placeholder
koda a "aws s3 sync ./dist s3://\${bucket}/app/ --delete --profile prod" -t aws -s deploy

# Swap the bucket at call time — no editing, no copy-paste
koda x deploy -V bucket=my-staging-frontend
koda x deploy -V bucket=my-prod-frontend
```

**④ Pipe any output in, reuse it immediately:**

```bash
# Capture a container's IP from docker inspect
docker inspect app | jq -r '.[0].NetworkSettings.IPAddress' | koda a -t docker

# Reuse it in the next command — no switching windows, no copy-paste
curl http://$(koda r):3000/healthz
psql postgres://postgres@$(koda r):5432/mydb
```

---

**⑤ Build a command library once — run it on every machine:**

> **Security note**: All entries are stored in plaintext SQLite.
> If Git sync is enabled, `koda-sync.jsonl` will contain every entry in
> plaintext and will be committed to the sync repository.
> **Do not store passwords, API keys, or tokens.**

```bash
# Save a library of reusable snippets on machine A
koda a "kubectl rollout restart deployment/\${svc} -n production" -t k8s -s k8s-restart
koda a "ssh -i ~/.ssh/prod.pem ec2-user@\$1 'sudo journalctl -u app -n 100'" -t ops -s prod-log
koda push   # commit and push to the Git sync repo

# On any other machine — one pull, then go
koda pull
koda x k8s-restart -V svc=api-gateway
koda x prod-log -V 10.0.1.42
```

## Quick reference

### Subcommands

| Command | Alias | Description |
|---|---|---|
| `add` | `a` | Save a new entry |
| `raw` | `r` | Print entry body to stdout |
| `list` | `l` | List and filter entries |
| `exec` | `x` | Run a saved entry as a shell command |
| `edit` | `e` | Open entry in `$EDITOR` |
| `pick` | `p` | Interactive selector (requires fzf) |
| `show` | `s` | Display entry with full metadata |
| `remove` | `d` | Delete entries |
| `copy` | `c` | Duplicate an entry |
| `tag` | `t` | Batch-add or remove tags |
| `move` | `m` | Move entry to a display index |
| `swap` | `w` | Swap display positions of two entries |
| `shift` | `h` | Shift entries up or down by N |
| `compact` | `k` | Reassign indices to 0..n-1 |
| `config` | `g` | Read/write configuration |

Single-letter aliases are reserved and cannot be used as entry shortcuts.

### Options

| Option | Description |
|---|---|
| `-s` / `--shortcut` | Assign a memorable alias to an entry |
| `-t` / `--tag` | Assign tags (on `add`) or filter by tag |
| `-V` / `--var` | Variable substitution at recall time |

## Installation

```bash
# uv (recommended)
uv tool install "koda-cli @ git+https://github.com/ngt22/koda-cli.git"

# pipx
pipx install "git+https://github.com/ngt22/koda-cli.git"
```

Turso remote backend (optional):

```bash
uv tool install "koda-cli[turso] @ git+https://github.com/ngt22/koda-cli.git"
# or
pipx install "git+https://github.com/ngt22/koda-cli.git" --pip-args ".[turso]"
```

## Update

```bash
# uv
uv tool upgrade koda-cli

# pipx
pipx upgrade koda-cli
```

## Command reference

### Add

Save a new entry from arguments, heredoc, stdin, or `$EDITOR`.

```bash
koda a "docker compose up --build" -t docker -s dc
koda a "YOUR_TEXT_HERE" -t work
koda a              # opens $EDITOR
```

Heredoc for multi-line content:

```bash
koda a -t infra <<'EOF'
FROM python:3.12-slim
RUN pip install uv
WORKDIR /app
EOF
```

Pipe any command output directly into `koda a`:

```bash
history | grep ffmpeg | tail -1 | koda a -t ffmpeg
kubectl get pods -o wide    | koda a -t k8s
```

Full form and aliases:

```bash
koda add "YOUR_TEXT_HERE" -t tag --shortcut sc   # long form
kd a "YOUR_TEXT_HERE" -t tag -s sc               # kd prefix (alias kd='koda')
ka "YOUR_TEXT_HERE" -t tag -s sc                  # two-letter alias
```

---

### Raw — body-only output

Print the entry body to stdout (no Rich formatting). Use for pipes, `eval`, and command substitution.

```bash
koda r web-srv        # by shortcut
koda r 5              # by index
koda r                # latest entry
echo 5 | koda r       # ref from stdin
```

Full form and aliases:

```bash
koda raw web-srv   # long form
kd r web-srv       # kd prefix
kr web-srv         # two-letter alias
```

**Command substitution — embed a stored value inside any command:**

```bash
# Without koda — retype long strings inline every time
ssh -i ~/.ssh/key.pem ec2-user@bastion.prod.example.com
tail -f /var/log/nginx/access.log

# Save once
koda a "bastion.prod.example.com"  -t ssh -s bastion
koda a "/var/log/nginx/access.log" -t log -s nginx-log

# Embed with $()
ssh -i ~/.ssh/key.pem ec2-user@$(koda r bastion)
tail -f $(koda r nginx-log)

# Same with shell aliases: kr = koda raw, kd r = koda raw with kd prefix
ssh -i ~/.ssh/key.pem ec2-user@$(kr bastion)
tail -f $(kd r nginx-log)
```

**Workflow example — capture a transient value once, reuse in requests:**

> **Security note**: Do not store passwords, API keys, or tokens.
> All entries are saved in plaintext SQLite, and Git sync will expose them
> in plaintext in the sync repository. Use this pattern for non-sensitive
> transient values only (container IPs, port numbers, generated paths, etc.).

```bash
# Step 1: capture an ephemeral value
docker inspect web \
  | jq -r '.[0].NetworkSettings.IPAddress' \
  | koda a -t docker -s web-ip

# Step 2: reuse it in subsequent commands — no copy-paste
curl http://$(koda r web-ip):3000/healthz

# Update when the value changes
koda e web-ip   # opens $EDITOR
```

`raw` strips shell-style inline comments (`#` at line start or after whitespace). Use `show` to see the original stored text.

```bash
koda a 'echo hello  # this is a comment'
koda r 5    # → echo hello
koda s 5    # → echo hello  # this is a comment
```

---

### List

```bash
koda l                          # all entries ordered by display index
koda l -q "docker"              # substring search on body
koda l -t linux                 # filter by tag substring
koda l -T archive               # exclude entries tagged "archive"
koda l -S                       # only entries that have a shortcut
koda l -n 50 -p 2              # 50 entries per page, page 2
koda l -s created_at --desc     # sort by creation date descending
koda l --columns idx,uid,sc,tags,content,created_at   # all columns
koda l --columns idx,content    # minimal view
```

Full form and aliases:

```bash
koda list -q docker -t dev   # long form
kd l -q docker -t dev        # kd prefix
kl -q docker -t dev          # two-letter alias
```

Default columns: `IDX`, `SC`, `Tags`, `Content`. Available columns: `idx`, `uid`, `sc`, `tags`, `content`, `created_at` (`idx` is required).
Sort columns: `id`, `idx`, `uid`, `tags`, `content`, `created_at`, `modified_at`, `shortcut`.

---

### Execute (`exec`) — run a saved command

Run a saved entry as a shell command, with optional variable substitution.

```bash
# Without koda — retype or search history every time
ssh -i ~/.ssh/key.pem ec2-user@192.168.1.100

# Save once
koda a "ssh -i ~/.ssh/key.pem ec2-user@\$1" -t ssh -s web-srv
```

```bash
koda x web-srv              # run by shortcut
koda x web-srv -V prod      # with variable substitution
koda x 12                   # run by index
```

Full form and aliases:

```bash
koda exec web-srv -V localhost   # long form
kd x web-srv -V localhost        # kd prefix
kx web-srv -V localhost          # two-letter alias
```

**Workflow example — query a local LLM with a one-liner:**

```bash
# Save once via heredoc
koda a -t llm -s ask <<'EOF'
curl -sS http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"messages": [{"role": "user", "content": "$1"}], "stream": false}' \
  | jq -r '.choices[0].message.content'
EOF

# Ask anything — no copy-paste, no editing
koda x ask -V "How high is Mt. Fuji?"
kx ask -V "Summarize the last git commit"   # two-letter alias
```

> **Security**: only store trusted commands. `exec` runs the body through the configured shell (`sh` by default).

---

### Edit

Open an entry in `$EDITOR`. The footer contains editable metadata (tags, shortcut).

```bash
koda e web-srv        # by shortcut
koda e 5              # by index
```

Full form and aliases:

```bash
koda edit web-srv   # long form
kd e web-srv        # kd prefix
ke web-srv          # two-letter alias
```

---

### Pick — interactive launcher (fzf)

Interactively select an entry with `fzf`, then run an action. Requires [`fzf`](https://github.com/junegunn/fzf) and an interactive TTY.

```bash
koda p -x                      # pick from all entries, execute immediately
koda p -x -q docker -t dev     # pre-filter by query and tag, then pick
```

Full form and aliases:

```bash
koda pick --exec -q docker -t dev   # long form
kd p -x -q docker -t dev            # kd prefix
kp -x -q docker -t dev              # two-letter alias
```

**Compound patterns — use pick as a selector:**

```bash
koda x "$(koda p -p)"          # pick IDX, pass to exec
kd x "$(kd p -p)"              # kd prefix
kx "$(kp -p)"                  # two-letter alias

eval $(kp -p | xargs kr)       # pick IDX, eval the body
```

Other action flags: `-e` edit, `-r` raw, `-s` show.
`-p` (print IDX only) cannot be combined with action flags.

---

### Show

Display a single entry with full metadata.

```bash
koda s web-srv        # by shortcut
koda s 5              # by index
echo 5 | koda s       # ref from stdin
```

Full form and aliases:

```bash
koda show web-srv   # long form
kd s web-srv        # kd prefix
ks web-srv          # two-letter alias
```

---

### Remove

Delete one or more entries.

```bash
koda d web-srv              # by shortcut
koda d 5                    # by index
koda d 1 3 5-8              # multiple entries and ranges
koda d -t archive           # delete all tagged "archive"
koda d -q "tmp"             # delete entries matching body substring
koda d --all -f             # delete everything (--all always requires -f)
```

Full form and aliases:

```bash
koda remove web-srv   # long form
kd d web-srv          # kd prefix
kd web-srv            # two-letter alias (kd = koda remove)
```

---

### Copy

Duplicate an entry. The body and tags are copied; the shortcut is not.

```bash
koda c web-srv        # by shortcut
koda c 5              # by index
```

Full form and aliases:

```bash
koda copy web-srv   # long form
kd c web-srv        # kd prefix
kc web-srv          # two-letter alias
```

---

### Tag

```bash
koda t 1 3 5 -t work          # add tag to individual entries
koda t 2-6 -t archive         # add tag to a range
koda t 1 3-5 7 -T old         # remove tag from mixed selection
koda t 1 -t new -T old        # add one tag and remove another in one command
```

Full form and aliases:

```bash
koda tag 1 3-5 -t archive   # long form
kd t 1 3-5 -t archive       # kd prefix
kt 1 3-5 -t archive         # two-letter alias
```

Re-tagging with an already-present tag is idempotent (no-op).

---

### Reorder entries (`move`, `swap`, `shift`, `compact`)

Each entry has a display index (`IDX`) you can freely rearrange — useful for keeping frequently used snippets at low numbers.

```bash
koda w 3 0        # swap display positions of entries 3 and 0
koda m 7 1        # move entry 7 to empty position 1
koda h 1          # shift entries at index 1+ up by 1 (makes room at 1)
koda h 1 -n 3     # shift up by 3 positions
koda h 5 -n -1    # shift entries from index 5 downward by 1
koda k            # reassign all indices to 0..n-1, fill gaps
```

Full form and aliases:

```bash
koda swap 3 0    kd w 3 0    kw 3 0
koda move 7 1    kd m 7 1    km 7 1
koda shift 1     kd h 1      kh 1
koda compact     kd k        kk
```

`move` requires the destination index to be unoccupied. Use `shift` to make room first, or `swap` to exchange two occupied positions.

---

### Configuration (`config`)

```bash
koda g                           # show all settings with source
koda g get defaults.cmd          # print a single value
koda g set defaults.cmd list     # write to config file
koda g unset list.per_page       # remove key (reverts to built-in default)
koda g reset -f                  # delete config file without prompt
koda g edit                      # open config in $EDITOR
koda g path                      # print config file path
```

Full form and aliases:

```bash
koda config set defaults.cmd list   # long form
kd g set defaults.cmd list          # kd prefix
kg set defaults.cmd list            # two-letter alias
```

---

## Options

The following are flags that work across multiple commands, not standalone subcommands.

---

### Shortcuts (`-s` / `--shortcut`)

Assign a memorable string alias to any entry and use it instead of a numeric index.

```bash
# Save with a shortcut (-s is short for --shortcut)
koda a "kubectl rollout restart deploy/api" -t k8s -s restart

# Use the shortcut anywhere an index is accepted
koda r restart
koda x restart
koda s restart
koda d restart

# Default command — no subcommand needed (when defaults.cmd = raw)
koda restart          # → koda raw restart

# List all entries that have shortcuts
koda l -S
koda l -S --sort-by shortcut
```

To change or remove a shortcut, open the entry with `edit` — the `shortcut:` field appears in the metadata footer.

---

### Tags (`-t` / `--tag`)

Assign one or more tags to an entry at save time; use them to filter across commands.

```bash
koda a "docker compose up" -t docker,dev     # assign multiple tags at add time
koda l -t docker                             # filter list by tag substring
koda l -T archive                            # exclude entries tagged "archive"
koda d -t tmp                                # delete all entries tagged "tmp"
koda p -x -t dev                             # pick + exec, pre-filtered by tag
```

Use `tag` (subcommand) to add or remove tags on existing entries in bulk — see [Tag](#tag).

---

### Variable substitution (`-V` / `--var`)

Embed placeholders in a saved entry; fill them in at recall time with `-V`.

| Style | Placeholder | How to pass |
|---|---|---|
| Named | `${host}` | `-V KEY=VALUE` |
| Positional | `$1`, `$2`, ... | `-V value` or `-V val1,val2` (comma-separated, left-to-right) |

```bash
# Save a template with a positional placeholder
koda a "gcloud storage cp \$1 gs://my-company-analytics-prod/uploads/" -t gcloud -s upload

# Run with different values — no need to retype the bucket path
koda x upload -V ./report.csv
koda x upload -V ./summary.csv
kx upload -V ./report.csv          # two-letter alias

# Named substitution — swap one variable by name
koda a "aws s3 sync ./dist s3://acme-frontend-\${env}-us-east-1/app/" -t aws -s deploy
koda x deploy -V env=prod
koda x deploy -V env=staging
kx deploy -V env=prod              # two-letter alias

# Multiple positional values
koda a "rsync -avz \$1 \$2" -t rsync
koda r 8 -V /src/path -V user@host:/dest
koda r 8 -V '/src/path,user@host:/dest'    # same result, comma-separated

# Mix named and positional
koda r 9 -V 'admin,5432' -V host=db.example.com -V name="new york"
# → connect admin@db.example.com:5432 as new york
```

Positional values are comma-separated within a single `-V` flag. Use `"..."` inside the flag to include spaces or commas in a value: `-V '"hello world","foo,bar"'`.

---

## Recommended aliases

Two patterns are available. Choose one based on how much you want to shorten your workflow.

---

### kd prefix — minimal (`alias kd='koda'` only)

Register only `kd` as an alias for `koda`, then use koda's built-in single-letter aliases for subcommands (`kd a`, `kd x`, `kd p -x`, etc.). No risk of conflicting with other tools.

```bash
# Add to ~/.zshrc or ~/.bashrc
alias kd='koda'
```

Usage with kd prefix:

```bash
kd a "YOUR_TEXT_HERE" -t tag -s sc    # add
kd l -q docker                        # list
kd s web-srv                # show
kd e web-srv                # edit
kd r web-srv                # raw
kd x web-srv -V localhost   # exec
kd p -x -q docker           # pick + exec
kd d web-srv                # remove
kd t 1 3-5 -t archive       # tag
kd g set defaults.cmd list  # config set
```

---

### Two-letter alias — full set

Register a two-letter alias for every subcommand. Shorter to type, but check for conflicts before adding.

> **Note**: This pattern assigns `kd` to `koda remove`. If you already have `alias kd='koda'` (kd prefix), remove that line first.

```bash
# Add to ~/.zshrc or ~/.bashrc
alias ka='koda add'
alias kl='koda list'
alias ks='koda show'
alias ke='koda edit'
alias kr='koda raw'
alias kx='koda exec'
alias kp='koda pick'
alias kd='koda remove'   # ← replaces kd='koda' if you had kd prefix
alias kc='koda copy'
alias km='koda move'
alias kw='koda swap'
alias kh='koda shift'
alias kk='koda compact'
alias kt='koda tag'
alias kg='koda config'
```

**Potential conflicts — check before adding:**

| Alias | Possible conflict |
|---|---|
| `ks` | Kakoune session manager on some setups |
| `kt` | Kotlin toolchain (`ktlint`, etc.) in some environments |

Run `alias` in your shell to see what is already defined before adding these.

---

## Configuration

Koda reads `~/.config/koda/config.toml` (XDG: `$XDG_CONFIG_HOME/koda/config.toml`). All settings are optional — unset values fall back to built-in defaults.

```toml
[defaults]
cmd = "raw"       # "raw", "list", "show", or "add" — what bare `koda` does

[list]
per_page = 20     # entries per page
rows = 1          # content preview lines (0 = all)
truncate = 80     # max chars per line (0 = no truncation)
sort_by = "idx"   # default sort column
desc = false      # sort direction
columns = ["idx", "sc", "tags", "content"]   # idx is required; available: idx, uid, sc, tags, content, created_at

[db]
path = "~/.local/share/koda/koda.db"
backend = "local"   # "local" or "turso"

[turso]
url = "libsql://YOUR_DB.turso.io"    # Turso database URL
# token = "..."                       # ⚠ Omit here — use KODA_TURSO_TOKEN env var
#                                     #   (config file may be committed to version control)

[git]
sync_path = "~/koda-sync"           # local clone of the sync repository
payload_file = "koda-sync.jsonl"    # JSONL file inside sync_path
sync_format = "jsonl"               # wire format (jsonl only)

[exec]
shell = "sh"      # shell used by exec
```

Priority order: **CLI flags > environment variables > config file > built-in defaults**

## Environment variables

| Variable | Purpose |
|---|---|
| `KODA_DB_PATH` | Override database file path |
| `KODA_DEFAULT_CMD` | Override `defaults.cmd` for this session |
| `KODA_CONFIG_PATH` | Override config file path |
| `KODA_TURSO_URL`        | Turso database URL (overrides `turso.url` in config) |
| `KODA_TURSO_TOKEN`      | Turso auth token (overrides `turso.token` in config) |
| `KODA_GIT_SYNC_PATH`    | Path to the local Git sync clone (overrides `git.sync_path`) |
| `KODA_GIT_PAYLOAD_FILE` | JSONL file name inside the sync clone (overrides `git.payload_file`) |
| `KODA_GIT_SYNC_FORMAT`  | Sync wire format — `jsonl` (overrides `git.sync_format`) |
| `EDITOR`                | Editor for `add`, `edit`, and `config edit` (default: `vim`) |

## Turso (remote database)

Koda supports [Turso](https://turso.tech) as a remote backend, letting you share entries across machines.

### Setup

1. Install the optional dependency:

```bash
pip install libsql-experimental
# or with uv:
uv tool install ".[turso]"
```

2. Create a Turso database and get your URL and token from the [Turso dashboard](https://app.turso.tech) or CLI:

```bash
turso db create koda
turso db show koda --url
turso db tokens create koda
```

3. Configure Koda — use environment variables to keep the token out of the config file:

```bash
export KODA_TURSO_URL="libsql://YOUR_DB.turso.io"
export KODA_TURSO_TOKEN="YOUR_AUTH_TOKEN"
koda config set db.backend turso
```

You can also write the URL to the config file, but **omit the token** and supply it via the env var:

```bash
koda config set db.backend turso
koda config set turso.url "libsql://YOUR_DB.turso.io"
# Do NOT run: koda config set turso.token "..."
# The config file may be committed to version control — use KODA_TURSO_TOKEN instead.
```

### Switching between backends

```bash
koda config set db.backend turso   # use Turso
koda config set db.backend local   # use local SQLite (default)
```

The local SQLite database (`db.path`) and the Turso database are independent — switching backends does not migrate data.

## Git sync (multi-machine sharing via GitHub)

Koda supports syncing entries across machines using a Git repository (e.g. a private GitHub repo) as a transport. On push, Koda exports the local database as a JSON Lines file (`koda-sync.jsonl`) into a local clone, commits it, and pushes to the remote. On pull, it fetches the latest commit and merges entries into the local database by `uid` and `modified_at` — newer wins, no entry is deleted.

This works independently of the Turso backend. You can use Git sync with the default local SQLite database.

> **Security note**: `koda-sync.jsonl` contains **all entries in plaintext**.
> Any passwords, tokens, or secrets stored in Koda will be committed to the
> sync repository in plaintext. Use a **private** repository and avoid storing
> sensitive values in Koda when Git sync is enabled.

### Setup

**1. Create a sync repository on GitHub (private recommended):**

```bash
# Create the repo on GitHub, then clone it locally
git clone git@github.com:YOUR_USERNAME/koda-sync.git ~/koda-sync
```

**2. Point Koda at the clone:**

```bash
koda config set git.sync_path ~/koda-sync
```

The `koda-sync` directory is the local clone root. Koda creates `koda-sync.jsonl` inside it automatically on first push.

**3. Confirm the configuration:**

```bash
koda config get git.sync_path      # → ~/koda-sync
koda config get git.payload_file   # → koda-sync.jsonl  (default)
koda config get git.sync_format    # → jsonl            (default)
```

### Push and pull

```bash
koda push   # export DB → koda-sync.jsonl, commit, push to remote
koda pull   # git pull the clone, merge koda-sync.jsonl into local DB
```

`push` does a `git pull --rebase` before writing the payload so the branch stays linear. `pull` merges by `uid` — entries that already exist locally are updated only if the incoming `modified_at` is newer.

### Setting up a second machine

```bash
# On the new machine: clone the sync repo
git clone git@github.com:YOUR_USERNAME/koda-sync.git ~/koda-sync

# Point Koda at it
koda config set git.sync_path ~/koda-sync

# Import the shared entries
koda pull
```

### Specifying a different remote

Koda resolves the remote automatically: it picks `origin` if available, otherwise the first listed remote. To use a different remote, set it as `origin` in the clone:

```bash
git -C ~/koda-sync remote set-url origin git@github.com:YOUR_USERNAME/koda-sync.git
```

Or add a named remote and set it as the tracking upstream for your branch:

```bash
git -C ~/koda-sync remote add work git@github.com:YOUR_ORG/koda-sync.git
git -C ~/koda-sync branch --set-upstream-to=work/main main
```

When an upstream is set for the current branch, Koda uses it directly (`git pull --rebase` / `git push`). When no upstream is configured, Koda calls `git push -u <remote> <branch>` to set one automatically on the first push.

### Configuration reference

| Key | Default | Description |
|---|---|---|
| `git.sync_path` | *(empty)* | Path to the local clone of the sync repository |
| `git.payload_file` | `koda-sync.jsonl` | JSONL file path, relative to `git.sync_path` |
| `git.sync_format` | `jsonl` | Wire format — only `jsonl` is supported |

Environment variable overrides:

| Variable | Purpose |
|---|---|
| `KODA_GIT_SYNC_PATH` | Override `git.sync_path` |
| `KODA_GIT_PAYLOAD_FILE` | Override `git.payload_file` |
| `KODA_GIT_SYNC_FORMAT` | Override `git.sync_format` |

## License

MIT (c) 2026 ngt22
