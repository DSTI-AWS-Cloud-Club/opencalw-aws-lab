# Phase 1 – OpenClaw Skills Setup (Ubuntu 24.04 EC2)

Goal: set up a lean OpenClaw skill stack on Ubuntu 24.04 EC2 using a single GEMINI_API_KEY (no coding-agent, no Gemini CLI). You will install and configure: clawdhub, goplaces, nano-banana-pro, nano-pdf, weather. Trello is optional (requires key/token). Obsidian is optional (requires sudo + Homebrew); skip if you cannot install it.

## Skills overview
- clawdhub: skills registry/installer used to pull and list skills.
- goplaces: Google Places search; needs `GOOGLE_PLACES_API_KEY`.
- nano-banana-pro: Gemini 3 Pro image generation/edit; needs `GEMINI_API_KEY` and `uv`.
- nano-pdf: natural-language PDF edits; uses `uv`.
- weather: basic weather lookup (no extra env required).
- trello: board/list/card management via Trello REST API; needs `TRELLO_API_KEY` and `TRELLO_TOKEN`.
- obsidian (optional): local markdown vault access; requires `obsidian-cli` and your vault.

## 0) System prep
```bash
sudo apt-get update
sudo apt-get install -y curl git build-essential python3 python3-venv pkg-config libssl-dev unzip jq
```

## 1) Node + pnpm (for OpenClaw ecosystem)
```bash
# Enable corepack and pin pnpm 10.23.0
sudo npm install -g corepack@latest
corepack enable
corepack prepare pnpm@10.23.0 --activate
```

## 2) Go toolchain (for goplaces)
```bash
sudo apt-get install -y golang
mkdir -p "$HOME/.local/bin"
export GOBIN="$HOME/.local/bin"
export PATH="$GOBIN:$PATH"
```

## 3) uv (for nano-banana-pro and nano-pdf)
```bash
curl -Ls https://astral.sh/uv/install.sh | sh
# Add to shell profile if not already present
export PATH="$HOME/.local/bin:$PATH"
```

## 4) Skill-specific installs
```bash
# ClawdHub CLI (skills registry)
npm install -g clawdhub

# goplaces CLI (Google Places)
GOBIN="$HOME/.local/bin" go install github.com/steipete/goplaces/cmd/goplaces@latest

# nano-pdf (natural-language PDF edits) – installs as a uv tool
uv tool install nano-pdf

# nano-banana-pro (Gemini 3 Pro Image)
# Uses bundled script; ensure uv is present, then register the skill
clawdhub install nano-banana-pro

# Obsidian CLI (for local markdown vault notes)
# Requires passwordless sudo for Homebrew. If you don't have sudo, skip Obsidian and use other skills.
# To install (optional):
cd ~
mkdir homebrew && curl -L https://github.com/Homebrew/brew/tarball/master | tar xz --strip 1 -C homebrew
# add homebrew bin to path
export PATH="$HOME/homebrew/bin/:$PATH"
# /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
# echo 'eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"' >> ~/.bashrc
# eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"
# brew install yakitrak/yakitrak/obsidian-cli

clawdhub install github
# GitHub CLI (gh) – already included with clawdhub
# No separate installation needed; gh comes with clawdhub
# Verify: which gh && gh --version

# access to a mail
clawdhub install himalaya
brew install himalaya


```

nano-banana-pro uses the bundled script; just ensure uv is installed (already done).

### Pass env vars into sandboxes
Sandboxed skills don’t see host env. Add keys to the sandbox Docker env so skills can read them:

```json5
{
  "agents": {
    "defaults": {
      "sandbox": {
        "docker": {
          "env": {
            "GEMINI_API_KEY": "<your_gemini_key>",
            "GOOGLE_PLACES_API_KEY": "<your_places_key>",
            "TRELLO_API_KEY": "<your_trello_api_key>",
            "TRELLO_TOKEN": "<your_trello_token>"
          }
        }
      }
    }
  }
}
```

Per-agent example:

```json5
{
  "agents": {
    "list": [
      {
        "name": "bot",
        "sandbox": {
          "docker": {
            "env": {
              "GEMINI_API_KEY": "<your_gemini_key>",
              "TRELLO_API_KEY": "<your_trello_api_key>",
              "TRELLO_TOKEN": "<your_trello_token>"
            }
          }
        }
      }
    ]
  }
}
```

### Bind skills into sandboxes
If your sandbox image supports `binds`, expose your workspace/skills so sandboxed skills can see them:

```json5
{
  "agents": {
    "defaults": {
      "sandbox": {
        "docker": {
          "binds": [
            "/home/ubuntu/openclaw:/workspace/openclaw:rw"
          ],
          "env": {
            "GEMINI_API_KEY": "<your_gemini_key>",
            "GOOGLE_PLACES_API_KEY": "<your_places_key>",
            "TRELLO_API_KEY": "<your_trello_api_key>",
            "TRELLO_TOKEN": "<your_trello_token>"
          }
        }
      }
    },
    "list": [
      {
        "id": "build",
        "sandbox": {
          "docker": {
            "binds": [
              "/home/ubuntu/<source-folder>:/<target_folder_in_sandbox>:rw", // target is generally /source:rw
            ]
          }
        }
      }
    ]
  }
}
```

Inside the sandbox, use the target path (e.g., `/workspace/openclaw/skills/...`). The `mounts` key is not supported; use `binds` as shown.

### Rebuild sandbox images

`openclaw sandbox recreate --all`

## 5) Environment variables (put in ~/.bashrc or ~/.profile)
```bash
export PATH="$HOME/homebrew/bin/:$PATH"

export GEMINI_API_KEY="<your_gemini_key>"          # for nano-banana-pro, summarize-like flows
export GOOGLE_PLACES_API_KEY="<your_places_key>"  # for goplaces
export TRELLO_API_KEY="<your_trello_api_key>"     # for trello
export TRELLO_TOKEN="<your_trello_token>"
export GH_TOKEN="<your_github_pat>"               # for GitHub CLI (gh)
export EDITOR="nano"                              # default editor for CLI tools (himalaya, etc.)
export PATH="$HOME/.local/bin:$PATH"
```

To persist across reboots/logins, append the exports to your shell profile (example for Bash):

```bash
cat <<'EOF' >> ~/.bashrc
export GEMINI_API_KEY="<your_gemini_key>"
export GOOGLE_PLACES_API_KEY="<your_places_key>"
export TRELLO_API_KEY="<your_trello_api_key>"
export TRELLO_TOKEN="<your_trello_token>"
export GH_TOKEN="<your_github_pat>"
export EDITOR="nano"
export PATH="$HOME/.local/bin:$PATH"
EOF
source ~/.bashrc
```

## 6) Notes skill choice
- Using `obsidian` (local markdown vault). Ensure `obsidian-cli` is installed and your vault exists; point the CLI to your default vault or specify path when invoking.

## 7) Quick validation
```bash
clawdhub --version
which goplaces && goplaces --help
uv --version
nano-pdf --help
clawdhub list | grep nano-banana-pro
which obsidian-cli && obsidian-cli --help   # only if you installed it
echo "$TRELLO_API_KEY" | head -c 4 && echo "..."   # quick presence check
command -v uv
```

## 8) Enable skills in OpenClaw config (~/.openclaw/openclaw.json)
```json5
{
  "skills": {
    "entries": {
      "goplaces": { "enabled": true, "env": { "GOOGLE_PLACES_API_KEY": "${GOOGLE_PLACES_API_KEY}" } },
      "trello": { "enabled": true, "env": { "TRELLO_API_KEY": "${TRELLO_API_KEY}", "TRELLO_TOKEN": "${TRELLO_TOKEN}" } },
      "nano-banana-pro": { "enabled": true, "env": { "GEMINI_API_KEY": "${GEMINI_API_KEY}" } },
      "nano-pdf": { "enabled": true },
      "weather": { "enabled": true },
      "clawdhub": { "enabled": true },
      "github": { "enabled": true, "env": { "GH_TOKEN": "${GH_TOKEN}" } },
      "nano-banana-pro": {
        "apiKey": "${GEMINI_API_KEY}" ,
        "env": {
          "GEMINI_API_KEY": "${GEMINI_API_KEY}" 
        }
      }
      // enable obsidian only if obsidian-cli is installed
      // "obsidian": { enabled: true }
    }
  }
}
```

Restart the gateway or start a new session to pick up env/config changes.

If any problem arises, try to do a `openclaw doctor --fix`

## 13) Himalaya email (Gmail setup)
Himalaya works with Gmail via IMAP/SMTP using an App Password. Steps:
- Create a dedicated Gmail account for the bot; enable 2-Step Verification.
- Enable IMAP: Settings → See all settings → Forwarding and POP/IMAP → Enable IMAP.
- Generate an App Password: Security → App passwords → choose “Mail” → copy the 16-char password once.
- Configure `~/.config/himalaya/config.toml` (or point to a custom path with `himalaya --config /path/to/config.toml`):

```toml
[accounts.gmail]
email = "yourbot@gmail.com"
display-name = "Your Bot"
default = true

backend.type = "imap"
backend.host = "imap.gmail.com"
backend.port = 993
backend.encryption.type = "tls"
backend.login = "yourbot@gmail.com"
backend.auth.type = "password"
backend.auth.cmd = "echo YOUR_APP_PASSWORD"

message.send.backend.type = "smtp"
message.send.backend.host = "smtp.gmail.com"
message.send.backend.port = 587
message.send.backend.encryption.type = "start-tls"
message.send.backend.login = "yourbot@gmail.com"
message.send.backend.auth.type = "password"
message.send.backend.auth.cmd = "echo YOUR_APP_PASSWORD"
```

Verify: `himalaya --version` then `himalaya envelope list`. Keep the App Password secret; rotate if exposed.

List unread quickly: `himalaya envelope list not flag seen` (add `folder INBOX` if needed). 
Then open one by ID: `himalaya message read <id-of-unread-message>`.

Workflow that worked for this bot:
- List unread: `himalaya envelope list not flag seen`.
- Read content by ID: `himalaya message read <id>`.
- Reply via piped template (headers in the body):
  ```bash
  cat <<'EOF' | /home/ubuntu/homebrew/bin/himalaya template send
  From: yourbot@gmail.com
  To: recipient@example.com
  Subject: Re: <subject>

  <reply body>
  EOF
  ```

## 11) Cron: daily agentic lesson (use isolated session)
When scheduling cron jobs that should run reliably without heartbeat context, use `--session isolated`. Example (daily at 09:00 UTC, delivered to Telegram):

```bash
openclaw cron add \
  --name "daily-agentic-lesson" \
  --cron "0 9 * * *" \
  --tz "UTC" \
  --session isolated \
  --message "Daily reminder: Generate and send agentic ML lesson to zBotta's Telegram (5552552764). Pick an unused topic from memory/daily-lessons.md. Format: 300-400 words." \
  --deliver \
  --channel telegram \
  --to "5552552764"
```

Note: main-session cron can be skipped if the heartbeat prompt is missing/empty; isolated avoids that.

## 10) Trello API checks
- Key/token must come from the same Trello account (https://trello.com/app-key). Regenerate both if unsure.

- Create a Trello connector/app first: log in to Trello → open https://trello.com/app-key → click “Token” to authorize; this creates the app (often saved as “trello-connector”) and gives you the matching key/token pair you must use together. When Trello asks for allowed origins, set it to the origin hosting your connector page , not the EC2 public IP unless you serve the connector there over HTTPS with a valid cert. If you need a minimal connector page, publish a GitHub Pages repo `trello-connector` (e.g., GitHub Pages `https://<user>.github.io`) with `index.html`:

```html
<!doctype html>
<html>
<head><meta charset="utf-8"><title>Trello Connector</title></head>
<body>
<script src="https://p.trellocdn.com/power-up.min.js"></script>
<script>
  window.TrelloPowerUp.initialize({});
</script>
</body>
</html>
```

Use the resulting Pages URL (e.g., `https://<user>.github.io/trello-connector/`) as the allowed origin.
- Ensure env is loaded before calling: `echo "$TRELLO_API_KEY" | head -c 4 && echo ...` and `echo "$TRELLO_TOKEN" | head -c 4 && echo ...`.
- Basic test:
```bash
curl -v "https://api.trello.com/1/members/me/boards?key=$TRELLO_API_KEY&token=$TRELLO_TOKEN" | jq '.[] | {name,id}'
```
- If you see "invalid key", the key/token pair is wrong or not exported; regenerate and re-source your shell, then retry.

## 9) nano-banana-pro usage (Gemini 3 Pro Image)
- Env: ensure `GEMINI_API_KEY` is exported **in the same shell** that runs OpenClaw (or pass `--api-key`). Verify with `env | grep GEMINI`.
- PATH: `command -v uv` should succeed; the script lives under `~/.openclaw/skills/nano-banana-pro/scripts/generate_image.py`.
- Example (edit with one input image):

```bash
uv run ~/.openclaw/workspace/skills/nano-banana-pro/scripts/generate_image.py \
  --prompt "Take this picture and generate ..." \
  --filename "OUTPUT_IMAGE_PATH" \
  --input-image "INPUT_IMAGE_PATH" \
  --resolution 2K
  --api-key $GEMINI_API_KEY
```

- Fallback if env isn’t seen (sandbox/agent): add `--api-key "<your_gemini_key>"` to the command.
- The script prints `MEDIA: <path>`; OpenClaw auto-attaches that file on supported channels.
## 12) GitHub CLI setup (for repository management)

## 12.1 install GH cli
# 1) Install deps
sudo apt-get update
sudo apt-get install -y curl git ca-certificates

# 2) Add GitHub CLI repo
type -p curl >/dev/null || sudo apt-get install -y curl
curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg \
  | sudo tee /usr/share/keyrings/githubcli-archive-keyring.gpg >/dev/null
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" \
  | sudo tee /etc/apt/sources.list.d/github-cli.list >/dev/null

# 3) Install gh
sudo apt-get update
sudo apt-get install -y gh

# 4) Verify
gh --version

## 12.2 Create a GitHub Personal Access Token (PAT)
1. Log in to GitHub with your bot's account (or the account you want the bot to use)
2. Go to: https://github.com/settings/tokens/new
3. Create a new **fine-grained** token (recommended) scoped to the single repo the bot needs. Minimum permissions:
  - Contents: **Read** (add **Write** only if the bot must push commits)
  - Pull requests: **Write** (if the bot opens/edits PRs)
  - Issues: **Write** (if the bot comments/labels issues)
  - Actions: **Read** (if the bot needs CI status); **Write** only if it must dispatch workflows
  Avoid broader scopes (Admin/Security/Org) unless strictly required.
  If you must use a classic token, keep it minimal: `repo` plus `workflow` only if needed.
4. Click "Generate token"
5. **Copy the token immediately** (you won't see it again)
6. If `github` was installed with `clawdhub` ask the Agent to clone your repository and do a modification
7. Give the PAT if asked for having modification access to repo



### Authenticate gh CLI with your token (MANUALLY IF MOLTBOT FAILS ON SETTING IT UP)
```bash
# Method 1: Interactive authentication (paste token when prompted)
gh auth login
# Select: GitHub.com → HTTPS → Paste an authentication token → [paste token]

# Method 2: Non-interactive (pipe token directly)
echo "YOUR_TOKEN_HERE" | gh auth login --with-token

# Method 3: Use environment variable (already set in step 5)
# Just ensure GH_TOKEN is exported and gh will use it automatically
```

### Configure git identity for commits
```bash
git config --global user.name "Your Bot Name"
git config --global user.email "bot@example.com"
```

### Test GitHub access
```bash
# Verify authentication
gh auth status

# List your repositories
gh repo list

# Test API access
gh api user --jq '.login'

# Clone a repository (example)
gh repo clone YOUR-USERNAME/YOUR-REPO
```

### What your bot can now do
- Clone, commit, and push to repositories
- Create, update, and merge pull requests
- Manage issues and discussions
- Run GitHub Actions workflows
- Use `gh api` for advanced GitHub API operations

### Example workflow: Update a portfolio repository
```bash
# Clone your portfolio
cd ~
gh repo clone YOUR-USERNAME/portfolio
cd portfolio

# Create a new branch
git checkout -b bot-updates

# Make changes (bot edits files)
echo "Updated by OpenClaw" >> README.md

# Commit and push
git add .
git commit -m "Update portfolio via OpenClaw"
git push origin bot-updates

# Create a pull request
gh pr create --title "Portfolio updates" --body "Automated updates from OpenClaw"

# Merge when ready (optional)
gh pr merge --squash
```
# Phase 1 – OpenClaw Skills Setup (Ubuntu 24.04 EC2)

Goal: set up a lean OpenClaw skill stack on Ubuntu 24.04 EC2 using a single GEMINI_API_KEY (no coding-agent, no Gemini CLI). You will install and configure: clawdhub, goplaces, nano-banana-pro, nano-pdf, weather. Trello is optional (requires key/token). Obsidian is optional (requires sudo + Homebrew); skip if you cannot install it.

## Skills overview
- clawdhub: skills registry/installer used to pull and list skills.
- goplaces: Google Places search; needs `GOOGLE_PLACES_API_KEY`.
- nano-banana-pro: Gemini 3 Pro image generation/edit; needs `GEMINI_API_KEY` and `uv`.
- nano-pdf: natural-language PDF edits; uses `uv`.
- weather: basic weather lookup (no extra env required).
- trello: board/list/card management via Trello REST API; needs `TRELLO_API_KEY` and `TRELLO_TOKEN`.
- obsidian (optional): local markdown vault access; requires `obsidian-cli` and your vault.

## 0) System prep
```bash
sudo apt-get update
sudo apt-get install -y curl git build-essential python3 python3-venv pkg-config libssl-dev unzip jq
```

## 1) Node + pnpm (for OpenClaw ecosystem)
```bash
# Enable corepack and pin pnpm 10.23.0
sudo npm install -g corepack@latest
corepack enable
corepack prepare pnpm@10.23.0 --activate
```

## 2) Go toolchain (for goplaces)
```bash
sudo apt-get install -y golang
mkdir -p "$HOME/.local/bin"
export GOBIN="$HOME/.local/bin"
export PATH="$GOBIN:$PATH"
```

## 3) uv (for nano-banana-pro and nano-pdf)
```bash
curl -Ls https://astral.sh/uv/install.sh | sh
# Add to shell profile if not already present
export PATH="$HOME/.local/bin:$PATH"
```

## 4) Skill-specific installs
```bash
# ClawdHub CLI (skills registry)
npm install -g clawdhub

# goplaces CLI (Google Places)
GOBIN="$HOME/.local/bin" go install github.com/steipete/goplaces/cmd/goplaces@latest

# nano-pdf (natural-language PDF edits) – installs as a uv tool
uv tool install nano-pdf

# nano-banana-pro (Gemini 3 Pro Image)
# Uses bundled script; ensure uv is present, then register the skill
clawdhub install nano-banana-pro

# Obsidian CLI (for local markdown vault notes)
# Requires passwordless sudo for Homebrew. If you don't have sudo, skip Obsidian and use other skills.
# To install (optional):
cd ~
mkdir homebrew && curl -L https://github.com/Homebrew/brew/tarball/master | tar xz --strip 1 -C homebrew
# add homebrew bin to path
export PATH="$HOME/homebrew/bin/:$PATH"
# /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
# echo 'eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"' >> ~/.bashrc
# eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"
# brew install yakitrak/yakitrak/obsidian-cli

clawdhub install github
# GitHub CLI (gh) – already included with clawdhub
# No separate installation needed; gh comes with clawdhub
# Verify: which gh && gh --version

# access to a mail
clawdhub install himalaya
brew install himalaya


```

nano-banana-pro uses the bundled script; just ensure uv is installed (already done).

### Pass env vars into sandboxes
Sandboxed skills don’t see host env. Add keys to the sandbox Docker env so skills can read them:

```json5
{
  "agents": {
    "defaults": {
      "sandbox": {
        "docker": {
          "env": {
            "GEMINI_API_KEY": "<your_gemini_key>",
            "GOOGLE_PLACES_API_KEY": "<your_places_key>",
            "TRELLO_API_KEY": "<your_trello_api_key>",
            "TRELLO_TOKEN": "<your_trello_token>"
          }
        }
      }
    }
  }
}
```

Per-agent example:

```json5
{
  "agents": {
    "list": [
      {
        "name": "bot",
        "sandbox": {
          "docker": {
            "env": {
              "GEMINI_API_KEY": "<your_gemini_key>",
              "TRELLO_API_KEY": "<your_trello_api_key>",
              "TRELLO_TOKEN": "<your_trello_token>"
            }
          }
        }
      }
    ]
  }
}
```

### Bind skills into sandboxes
If your sandbox image supports `binds`, expose your workspace/skills so sandboxed skills can see them:

```json5
{
  "agents": {
    "defaults": {
      "sandbox": {
        "docker": {
          "binds": [
            "/home/ubuntu/openclaw:/workspace/openclaw:rw"
          ],
          "env": {
            "GEMINI_API_KEY": "<your_gemini_key>",
            "GOOGLE_PLACES_API_KEY": "<your_places_key>",
            "TRELLO_API_KEY": "<your_trello_api_key>",
            "TRELLO_TOKEN": "<your_trello_token>"
          }
        }
      }
    },
    "list": [
      {
        "id": "build",
        "sandbox": {
          "docker": {
            "binds": [
              "/home/ubuntu/<source-folder>:/<target_folder_in_sandbox>:rw", // target is generally /source:rw
            ]
          }
        }
      }
    ]
  }
}
```

Inside the sandbox, use the target path (e.g., `/workspace/openclaw/skills/...`). The `mounts` key is not supported; use `binds` as shown.

### Rebuild sandbox images

`openclaw sandbox recreate --all`

## 5) Environment variables (put in ~/.bashrc or ~/.profile)
```bash
export PATH="$HOME/homebrew/bin/:$PATH"

export GEMINI_API_KEY="<your_gemini_key>"          # for nano-banana-pro, summarize-like flows
export GOOGLE_PLACES_API_KEY="<your_places_key>"  # for goplaces
export TRELLO_API_KEY="<your_trello_api_key>"     # for trello
export TRELLO_TOKEN="<your_trello_token>"
export GH_TOKEN="<your_github_pat>"               # for GitHub CLI (gh)
export EDITOR="nano"                              # default editor for CLI tools (himalaya, etc.)
export PATH="$HOME/.local/bin:$PATH"
```

To persist across reboots/logins, append the exports to your shell profile (example for Bash):

```bash
cat <<'EOF' >> ~/.bashrc
export GEMINI_API_KEY="<your_gemini_key>"
export GOOGLE_PLACES_API_KEY="<your_places_key>"
export TRELLO_API_KEY="<your_trello_api_key>"
export TRELLO_TOKEN="<your_trello_token>"
export GH_TOKEN="<your_github_pat>"
export EDITOR="nano"
export PATH="$HOME/.local/bin:$PATH"
EOF
source ~/.bashrc
```

## 6) Notes skill choice
- Using `obsidian` (local markdown vault). Ensure `obsidian-cli` is installed and your vault exists; point the CLI to your default vault or specify path when invoking.

## 7) Quick validation
```bash
clawdhub --version
which goplaces && goplaces --help
uv --version
nano-pdf --help
clawdhub list | grep nano-banana-pro
which obsidian-cli && obsidian-cli --help   # only if you installed it
echo "$TRELLO_API_KEY" | head -c 4 && echo "..."   # quick presence check
command -v uv
```

## 8) Enable skills in OpenClaw config (~/.openclaw/openclaw.json)
```json5
{
  "skills": {
    "entries": {
      "goplaces": { "enabled": true, "env": { "GOOGLE_PLACES_API_KEY": "${GOOGLE_PLACES_API_KEY}" } },
      "trello": { "enabled": true, "env": { "TRELLO_API_KEY": "${TRELLO_API_KEY}", "TRELLO_TOKEN": "${TRELLO_TOKEN}" } },
      "nano-banana-pro": { "enabled": true, "env": { "GEMINI_API_KEY": "${GEMINI_API_KEY}" } },
      "nano-pdf": { "enabled": true },
      "weather": { "enabled": true },
      "clawdhub": { "enabled": true },
      "github": { "enabled": true, "env": { "GH_TOKEN": "${GH_TOKEN}" } },
      "nano-banana-pro": {
        "apiKey": "${GEMINI_API_KEY}",
        "env": {
          "GEMINI_API_KEY": "${GEMINI_API_KEY}"

        }
      }
      // enable obsidian only if obsidian-cli is installed
      // "obsidian": { enabled: true }
    }
  }
}
```

Restart the gateway or start a new session to pick up env/config changes.

If any problem arises, try to do a `openclaw doctor --fix`

## 13) Himalaya email (Gmail setup)
Himalaya works with Gmail via IMAP/SMTP using an App Password. Steps:
- Create a dedicated Gmail account for the bot; enable 2-Step Verification.
- Enable IMAP: Settings → See all settings → Forwarding and POP/IMAP → Enable IMAP.
- Generate an App Password: Security → App passwords → choose “Mail” → copy the 16-char password once.
- Configure `~/.config/himalaya/config.toml` (or point to a custom path with `himalaya --config /path/to/config.toml`):

```toml
[accounts.gmail]
email = "yourbot@gmail.com"
display-name = "Your Bot"
default = true

backend.type = "imap"
backend.host = "imap.gmail.com"
backend.port = 993
backend.encryption.type = "tls"
backend.login = "yourbot@gmail.com"
backend.auth.type = "password"
backend.auth.cmd = "echo YOUR_APP_PASSWORD"

message.send.backend.type = "smtp"
message.send.backend.host = "smtp.gmail.com"
message.send.backend.port = 587
message.send.backend.encryption.type = "start-tls"
message.send.backend.login = "yourbot@gmail.com"
message.send.backend.auth.type = "password"
message.send.backend.auth.cmd = "echo YOUR_APP_PASSWORD"
```

Verify: `himalaya --version` then `himalaya envelope list`. Keep the App Password secret; rotate if exposed.

List unread quickly: `himalaya envelope list not flag seen` (add `folder INBOX` if needed). 
Then open one by ID: `himalaya message read <id-of-unread-message>`.

Workflow that worked for this bot:
- List unread: `himalaya envelope list not flag seen`.
- Read content by ID: `himalaya message read <id>`.
- Reply via piped template (headers in the body):
  ```bash
  cat <<'EOF' | /home/ubuntu/homebrew/bin/himalaya template send
  From: yourbot@gmail.com
  To: recipient@example.com
  Subject: Re: <subject>

  <reply body>
  EOF
  ```

## 11) Cron: daily agentic lesson (use isolated session)
When scheduling cron jobs that should run reliably without heartbeat context, use `--session isolated`. Example (daily at 09:00 UTC, delivered to Telegram):

```bash
openclaw cron add \
  --name "daily-agentic-lesson" \
  --cron "0 9 * * *" \
  --tz "UTC" \
  --session isolated \
  --message "Daily reminder: Generate and send agentic ML lesson to zBotta's Telegram (5552552764). Pick an unused topic from memory/daily-lessons.md. Format: 300-400 words." \
  --deliver \
  --channel telegram \
  --to "5552552764"
```

Note: main-session cron can be skipped if the heartbeat prompt is missing/empty; isolated avoids that.

## 10) Trello API checks
- Key/token must come from the same Trello account (https://trello.com/app-key). Regenerate both if unsure.

- Create a Trello connector/app first: log in to Trello → open https://trello.com/app-key → click “Token” to authorize; this creates the app (often saved as “trello-connector”) and gives you the matching key/token pair you must use together. When Trello asks for allowed origins, set it to the origin hosting your connector page , not the EC2 public IP unless you serve the connector there over HTTPS with a valid cert. If you need a minimal connector page, publish a GitHub Pages repo `trello-connector` (e.g., GitHub Pages `https://<user>.github.io`) with `index.html`:

```html
<!doctype html>
<html>
<head><meta charset="utf-8"><title>Trello Connector</title></head>
<body>
<script src="https://p.trellocdn.com/power-up.min.js"></script>
<script>
  window.TrelloPowerUp.initialize({});
</script>
</body>
</html>
```

Use the resulting Pages URL (e.g., `https://<user>.github.io/trello-connector/`) as the allowed origin.
- Ensure env is loaded before calling: `echo "$TRELLO_API_KEY" | head -c 4 && echo ...` and `echo "$TRELLO_TOKEN" | head -c 4 && echo ...`.
- Basic test:
```bash
curl -v "https://api.trello.com/1/members/me/boards?key=$TRELLO_API_KEY&token=$TRELLO_TOKEN" | jq '.[] | {name,id}'
```
- If you see "invalid key", the key/token pair is wrong or not exported; regenerate and re-source your shell, then retry.

## 9) nano-banana-pro usage (Gemini 3 Pro Image)
- Env: ensure `GEMINI_API_KEY` is exported **in the same shell** that runs OpenClaw (or pass `--api-key`). Verify with `env | grep GEMINI`.
- PATH: `command -v uv` should succeed; the script lives under `~/.openclaw/skills/nano-banana-pro/scripts/generate_image.py`.
- Example (edit with one input image):

```bash
uv run ~/.openclaw/workspace/skills/nano-banana-pro/scripts/generate_image.py \
  --prompt "Take this picture and generate ..." \
  --filename "OUTPUT_IMAGE_PATH" \
  --input-image "INPUT_IMAGE_PATH" \
  --resolution 2K
  --api-key $GEMINI_API_KEY
```

- Fallback if env isn’t seen (sandbox/agent): add `--api-key "<your_gemini_key>"` to the command.
- The script prints `MEDIA: <path>`; OpenClaw auto-attaches that file on supported channels.
## 12) GitHub CLI setup (for repository management)

## 12.1 install GH cli
# 1) Install deps
sudo apt-get update
sudo apt-get install -y curl git ca-certificates

# 2) Add GitHub CLI repo
type -p curl >/dev/null || sudo apt-get install -y curl
curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg \
  | sudo tee /usr/share/keyrings/githubcli-archive-keyring.gpg >/dev/null
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" \
  | sudo tee /etc/apt/sources.list.d/github-cli.list >/dev/null

# 3) Install gh
sudo apt-get update
sudo apt-get install -y gh

# 4) Verify
gh --version

## 12.2 Create a GitHub Personal Access Token (PAT)
1. Log in to GitHub with your bot's account (or the account you want the bot to use)
2. Go to: https://github.com/settings/tokens/new
3. Create a new **fine-grained** token (recommended) scoped to the single repo the bot needs. Minimum permissions:
  - Contents: **Read** (add **Write** only if the bot must push commits)
  - Pull requests: **Write** (if the bot opens/edits PRs)
  - Issues: **Write** (if the bot comments/labels issues)
  - Actions: **Read** (if the bot needs CI status); **Write** only if it must dispatch workflows
  Avoid broader scopes (Admin/Security/Org) unless strictly required.
  If you must use a classic token, keep it minimal: `repo` plus `workflow` only if needed.
4. Click "Generate token"
5. **Copy the token immediately** (you won't see it again)
6. If `github` was installed with `clawdhub` ask the Agent to clone your repository and do a modification
7. Give the PAT if asked for having modification access to repo



### Authenticate gh CLI with your token (MANUALLY IF MOLTBOT FAILS ON SETTING IT UP)
```bash
# Method 1: Interactive authentication (paste token when prompted)
gh auth login
# Select: GitHub.com → HTTPS → Paste an authentication token → [paste token]

# Method 2: Non-interactive (pipe token directly)
echo "YOUR_TOKEN_HERE" | gh auth login --with-token

# Method 3: Use environment variable (already set in step 5)
# Just ensure GH_TOKEN is exported and gh will use it automatically
```

### Configure git identity for commits
```bash
git config --global user.name "Your Bot Name"
git config --global user.email "bot@example.com"
```

### Test GitHub access
```bash
# Verify authentication
gh auth status

# List your repositories
gh repo list

# Test API access
gh api user --jq '.login'

# Clone a repository (example)
gh repo clone YOUR-USERNAME/YOUR-REPO
```

### What your bot can now do
- Clone, commit, and push to repositories
- Create, update, and merge pull requests
- Manage issues and discussions
- Run GitHub Actions workflows
- Use `gh api` for advanced GitHub API operations

### Example workflow: Update a portfolio repository
```bash
# Clone your portfolio
cd ~
gh repo clone YOUR-USERNAME/portfolio
cd portfolio

# Create a new branch
git checkout -b bot-updates

# Make changes (bot edits files)
echo "Updated by OpenClaw" >> README.md

# Commit and push
git add .
git commit -m "Update portfolio via OpenClaw"
git push origin bot-updates

# Create a pull request
gh pr create --title "Portfolio updates" --body "Automated updates from OpenClaw"

# Merge when ready (optional)
gh pr merge --squash
```

# EXTRA

## NVIDIA API KEY -> USE Nvidia build service to call many models !
### Define API KEY in env variables
1. Create an account in NVIDIA build and create an API KEY.
2. Create env variable in `~/.bashrc` file, edit it and add it to the env variables:

Add this line
```bash
export NVIDIA_API_KEY="YOUR_NVIDIA_API_KEY"
```
3. Restart env variables
` source ~/.bashrc`

### Create the provider in OpenClaw

1. **Define the NVIDIA LLM Provider**:  We first needed to tell OpenClaw about the NVIDIA API and how to access it. This was done by adding an entry under `models.providers` in `openclaw.json` config file.
  • The provider was named `nvidia`.
  • It was configured as an `openai_compatible` type.
  • We specified the `baseUrl` as `https://integrate.api.nvidia.com/v1`.
  • We indicated that the apiKey should be sourced from an environment variable named `NVIDIA_API_KEY`.
  • The specific model within this provider was moonshotai/kimi-k2.5.
  • We also provided details for the model itself, like its contextWindow and maxTokens, along with its id and name.
The exact JSON patch applied for this step was:

```bash
{
  "models": {
    "providers": {
      "nvidia": {
        "baseUrl": "https://integrate.api.nvidia.com/v1",
        "apiKey": "NVIDIA_API_KEY",
        "api": "openai-completions",
        "models": [
          {
            "id": "moonshotai/kimi-k2.5",
            "name": "Kimi K2.5",
            "reasoning": false,
            "input": ["text"],
            "cost": {"input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0},
            "contextWindow": 16384,
            "maxTokens": 16384
          }
        ]
      }
    }
  }
}

```

Note: This patch used apiKey to reference the environment variable. If you were to provide the key directly (not recommended for security), it would be a string value.
2. **Register the Model for Agent Use:** After defining the provider, we needed to make sure agents could discover and use the `moonshotai/kimi-k2.5` model specifically. This involves updating the agents.defaults.models configuration.
  • We registered the model using the format `provider_name/model_id`, which in this case is nvidia/moonshotai/kimi-k2.5.
The JSON patch for this step was:

3. **Restart the Gateway**:  The `config.patch` action automatically triggers a gateway restart to apply these changes. You'll see a "GatewayRestart: ok" confirmation after each successful patch.

4. **Test and Verify**: load model into Openclaw tui and chat with it

5. **Define Kimi for managing sub-agents**: Kimi K2.5 is great for sub-agents management and optimal parallelisation.
To set nvidia/moonshotai/kimi-k2.5 as the default model for managing sub-agents, the bot will need to update the `primary model` setting within the agent configuration (`openclaw.json` file).

Or ask your bot the following, he'll do the rest.

```text
I want you to use the NVIDA_API_KEY to set a Kimi K2.5 (moonshotai/Kimi-K2.5) as default model for managing sub-agents.
``` 