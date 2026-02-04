# OpenClaw AWS Lab
This lab deploys OpenClaw in an EC2 instance and sets up Telegram as an assistant bot.

We will ask the bot to give us the latest trends in Data, AI agents, and Machine Learning while we message it through our Telegram account! :grin:

**NB**: OpenClaw is the refreshed name for MoltBot (which itself started life as ClawdBot). The CLI binary is now `openclaw`, but some installers, screenshots, and docs still show the older branding until upstream assets are updated.

## Overview
- Launch an AWS Free Tier Ubuntu instance and install the OpenClaw CLI.
- Harden the deployment with injection protection, sandboxing, and audits.
- Configure API keys, Telegram, and a tailored "soul" for your bot.
- Expose the Control UI locally, connect MCP servers, and schedule cron jobs.
- Extend the setup with recommended models and sandbox container builds.

## Table of Contents
- [Overview](#overview)
- [Pre-requisites](#pre-requisites)
- [Resources](#resources)
- [Part 1. Creating an EC2 instance and installing OpenClaw](#part-1-creating-an-ec2-instance-and-installing-openclaw)
- [Part 2. Security installs](#part-2-security-installs)
- [Part 3. Configuring OpenClaw](#part-3-configuring-openclaw)
- [Part 4. Connecting to OpenClaw's Control UI](#part-4-connecting-to-openclaws-control-ui)
- [Part 5. Installing MCP servers in OpenClaw](#part-5-installing-mcp-servers-in-openclaw)
- [Part 6. Setting up a cron job](#part-6-setting-up-a-cron-job)
- [Part 7. Recommended models](#part-7-recommended-models)
- [Part 8. Advanced topics](#part-8-advanced-topics)

## Pre-requisites
- AWS Free Tier account
- Telegram installed on your phone
- Either a Claude API key (you must add $5 of credit) or a Gemini API key ($300 in credits for free during 90 days in AI Studio)
  - Create a Claude key on the [Claude Platform](https://platform.claude.com/)
  - Create a Gemini key in [AI Studio](https://aistudio.google.com/api-keys)

## Resources

### OpenClaw resources (legacy links still valid)
- :bulb: **Amazin resource!**. Use the [Deep wiki](https://deepwiki.com/openclaw/openclaw) to ask questions about everything you need to know about OpenClaw. e.g how to configure Openclaw regarding security and skills... 
It has an AI to assist you on complex questions !
- Learn how people use OpenClaw on its [website](https://clawd.bot/)
- Walkthrough video for installing OpenClaw in an EC2 instance: [Video from AJ](https://x.com/techfrenAJ/status/2014934471095812547/video/1)
- Injection protection tutorial by Dicklesworthstone (legacy repo path still uses the previous name): [tutorial link](https://github.com/Dicklesworthstone/acip/tree/main/integrations/clawdbot)


### Local models
- Install a local Ollama model for OpenClaw with this [video tutorial](https://www.youtube.com/watch?v=Idkkl6InPbU)

### About MCP
- Overview video on providing MCPs to your Vibe tool (Codex, Cursor, Claude Code, or Droid): [YouTube link](https://www.youtube.com/watch?v=w-rXEUTOIas)

### Gemini
- Create a Gemini API account in [AI Studio](https://aistudio.google.com/api-keys) and receive $300 in credits for 90 days. Use the **Overview** page to monitor spending :grin:
- Review the [Gemini API pricing](https://ai.google.dev/gemini-api/docs/pricing)

## Part 1. Creating an EC2 instance and installing OpenClaw

### 1.1 Creating an EC2 instance in an AWS Free Tier account
1. In the AWS Dashboard, navigate to **EC2 Console**.
2. Select **Instances** and **Launch Instances**.
3. Name the instance `OpenClaw`.
4. OS: select *Ubuntu* with *Ubuntu Server 24.04*.
5. Instance type: choose **t2.micro** or **t3.micro** (Free Tier eligible). You can check Free Tier eligibility with:
    ```bash
    aws ec2 describe-instance-types \
    --filters Name=free-tier-eligible,Values=true \
    --query "InstanceTypes[*].[InstanceType]" \
    --output text | sort
    ```
6. Key pair: generate a `.pem` key pair (needed later for the SSH tunnel).
7. Storage: `16 GB` is enough if you are not installing local LLMs. Installing Ollama can require up to `60 GB` depending on the model. Without a GPU there is little value in provisioning 60 GB.
8. Launch the instance.
9. Go to **Instances**, select the instance, and click **Connect**.
10. Click **Connect** again to open the browser-based Ubuntu terminal.

### 1.2 Installing OpenClaw
1. Inside the EC2 instance, run:

   `curl -fsSL https://molt.bot/install.sh | bash`

2. This downloads OpenClaw; wait a few minutes for the installer to pop up (installer URL still lives under the molt.bot domain for now).

![OpenClaw onboard wizard (legacy splash)](/assets/moltbot-onboard.JPG)

3. Before configuring anything, answer **No** and reboot the instance so you can access the `openclaw` CLI.
4. Reboot with `sudo reboot`. Wait a few seconds and reconnect.
5. Confirm the CLI is installed: `openclaw help`.

**Before going any further, stop the bot with `Ctrl+C` and complete the security installs.**

## Part 2. Security installs
**Protecting your personal information**

Granting the bot access to email, phone, or files requires careful security. Walk through each hardening step and review the X post from OpenClaw creator Peter Steinberg.

![Security checklist shared by Peter Steinberg](/assets/security-clawdbot.JPG)

### 2.1 Injection attacks
Configuring injection protection should be your first security task. Follow this [tutorial](https://github.com/Dicklesworthstone/acip/tree/main/integrations/clawdbot) by Dicklesworthstone (legacy repo name, but the guidance still applies to OpenClaw).

### 2.2 Add a sandbox configuration (optional)
You can isolate each session inside a Docker-based sandbox to protect the host.

```bash
nano ~/.openclaw/openclaw.json
```

Paste **before the `compaction` block**:

```json
----- COPY THIS PART ONLY --------
"sandbox": {
       "mode": "off", // set to "all" for full sandbox isolation
       "scope": "session",
       "workspaceAccess": "none"
       },
------------------
 # DO NOT COPY THIS
      "compaction": {
        "mode": "safeguard"
```

#### 2.2.1 Install Docker in Ubuntu (skip if no sandbox)
Docker must be installed to enable sandboxing. Each session runs inside its own container to limit exposure.

```bash
# Add Docker's official GPG key:
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update

sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

sudo systemctl status docker

sudo systemctl start docker
```

#### 2.2.2 Troubleshooting Docker
If Docker commands fail, add your user to the `docker` group.

```bash
# 1) Ensure Docker is installed and running
sudo systemctl enable --now docker
sudo systemctl status docker --no-pager

# 2) Add your current user to the docker group
sudo usermod -aG docker $USER

# 3) Apply the new group membership (choose ONE)
newgrp docker
# or log out and log back in (SSH: disconnect/reconnect)

# 4) Test
docker ps
```

#### 2.2.3 Restart services
Apply the sandbox changes:

`openclaw gateway restart`

Then reboot the instance:

`sudo reboot`

### 2.3 Run an audit
Whenever you add a new feature (for example, a Telegram connection), run a security audit to confirm the configuration is safe. Read the details [here](https://docs.molt.bot/cli/security).

`openclaw security audit`

`openclaw security audit --deep` (deeper insight)

`openclaw security audit --fix` (auto-remediate detected issues)

### 2.4 Security notes
For broader security context, watch this [video](https://www.youtube.com/watch?v=AbCHaAeqC_c) and follow the steps.

:warning: Be careful about the data you permit your bot to access. Sensitive data leakage is your responsibility.

To minimize risk:
- Read the [documentation](https://docs.molt.bot/)
- Research each feature before deploying it
- Understand what the bot and external agents can access

## Part 3. Configuring OpenClaw

### 3.1 Creating an Anthropic API key
1. Visit the [Claude Platform](https://platform.claude.com/).
2. Create an account and add a billing method.
3. Navigate to **API keys** and click **Create key**.
4. Name and copy the key.
5. Store it securely.

:pen: **If you want to test without paying yet, request a Gemini API key instead. It is free for 90 days with $300 in credits.**

### 3.2 Setting up OpenClaw
1. Run the onboard interface:

   `openclaw onboard`

2. Select **Yes**.
3. Choose **Quick Start**.
4. Pick your provider:
   - Anthropic API key: `Anthropic -> Anthropic API KEY`
   - Google Gemini API key: `Google -> Gemini API KEY`
5. Paste your API key and press Enter.
6. Select `Haiku 4.5 (latest --k reasoning)`.

### 3.3 Setting up Telegram
7. From the onboard interface, choose **Telegram**.
8. Follow @BotFather in Telegram:
   - Run `/newbot` (or `/mybots`).
   - Provide a unique name for your bot, e.g., `MyAwesomeAssistant`.
   - Copy the token (format `123456:ABC...`).
9. Paste the token into the onboard prompt and press Enter.
   9.1. In Telegram, search for `@YOUR_BOT_NAME`, click **START**, and you will receive a message like:

        ```
        OpenClaw: access not configured.

        Your Telegram user id: ________

        Pairing code: __________

        Ask the bot owner to approve with:
        openclaw pairing approve telegram <code>
        ```

   9.2. Copy the entire message and paste it into the EC2 terminal.
   9.3. You can now chat with your bot! :alien:

10. For the remaining prompts, choose **No / Skip for now**.

### 3.4 Giving a duty and instructions to the bot
11. :warning: Stop when prompted about **Hatch in TUI**.
12. Select **Hatch in TUI**.
13. Start chatting with the bot.
14. When it asks for your name, provide one (e.g., `DSTI`).
15. When asked about its personality, define the bot's duty.

Give the bot a name:
```
Your name is <YOUR_BOT_NAME>.
```

Then define its soul and duty, for example:

```text
You are an AI assistant designed to be diligent, organized, and proactive. Your mission is to help me grow as an AI Engineer while keeping my day-to-day life structured and under control. Prioritize: (1) clear next actions, (2) correctness, (3) time-saving. Communicate concisely, ask clarifying questions only when necessary, and propose sensible defaults when information is missing.

You are my learning partner for Python, Linux, Data Engineering, Data Science, and AI/ML systems. When teaching, use step-by-step explanations, small examples, and quick checks for understanding. When coding, prefer maintainable solutions: readable structure, comments, tests where useful, and safe handling of credentials. When troubleshooting, start with hypotheses, then minimal commands to confirm.

You can work with my tools (Trello, GitHub, and an email inbox). Use them to: track tasks, break work into checklists, draft and review code, summarize threads, extract action items, and prepare replies. Always protect privacy and security: never expose secrets, request tokens/keys through secure means, and confirm before performing any irreversible actions.
```

### 3.4.1 Asking for a format of the Python lessons

```text

MISSION
- Help the user improve Python coding skill every day with a short, practical digest they can read on a phone.
- Adapt difficulty and topic selection to the user’s level based on their responses and performance over time.
- Keep the daily reading time under 5 minutes.

PRIMARY DUTY (EVERY DAY)
Deliver a “Daily Python Practice Digest” that includes:
1) a tiny concept (high leverage),
2) a short example,
3) a micro-exercise,
4) a fast self-check,
5) one optional stretch task,
6) one quick calibration question for tomorrow.

DEFAULT USER PROFILE
- The user is a Data Scientist / Data Engineer.
- They want practical Python improvements for real DS/DE work: data handling, pipelines, APIs, notebooks, testing, performance.

ADAPTIVE LEARNING RULES
- Maintain an internal estimate of level across:
  A) Python fundamentals
  B) Data tooling (pandas/numpy, IO)
  C) Software engineering (structure, readability, typing)
  D) Reliability (exceptions, logging, testing)
  E) Performance (profiling, algorithms, memory)
- Difficulty adjustment:
  - If user solved quickly and accurately → slightly harder tomorrow (one extra edge case/constraint).
  - If user struggled → simplify tomorrow and reteach with a different angle.
  - If user requests a focus area → prioritize it for the next 3–5 days.

CONTENT POLICY (WHAT TO TEACH)
Prioritize DS/DE-relevant Python:
- Clean functions/modules, good naming, decomposition
- Iterables, generators, context managers
- Built-ins mastery (sorted, any/all, enumerate, zip, itertools)
- Data parsing + IO patterns (CSV/JSON/Parquet)
- Pandas best practices (loc, avoiding chained indexing, vectorization mindset)
- Testing/debugging/logging essentials
- Type hints + dataclasses for maintainability
- Error handling + defensive coding
- Performance wins (profiling mindset, reducing Python-level loops when possible)
Avoid long lectures, too many new concepts per day, and unnecessary math.

DAILY DIGEST FORMAT (MOBILE-FIRST, <5 MIN READ)
Return exactly this structure:

[HEADER]
PySprint Daily — <YYYY-MM-DD> (timezone: Europe/Paris)
Goal: <one sentence>
Time: ~3–5 min

[1) TODAY’S MICRO-CONCEPT]
- Name: <concept>
- Why it matters (1–2 lines, DS/DE context)

[2) TINY EXAMPLE]
- 6–12 lines of Python max
- Include 1 short comment explaining the key idea

[3) MICRO-EXERCISE (3–7 MIN CODING)]
- Task: <instructions>
- Input/Output: <tiny spec>
- Constraints: <0–2 constraints max>
- Tip: <optional hint>

[4) SELF-CHECK]
- 2–4 expected behaviors/tests (plain English or minimal asserts)
- Include at least one edge case

[5) STRETCH (OPTIONAL)]
- One harder variation (1–2 lines)

[6) QUICK QUESTION FOR ADAPTATION]
Ask exactly ONE question (A/B/C/D or 1–10 confidence).

INTERACTION RULES
- Be concise and direct.
- Encourage the user to reply with either:
  - their solution code, or
  - a short description of what they tried, or
  - the answer to the quick question.
- When the user replies with code, give feedback in this order:
  correctness → readability → robustness → performance
- Provide exactly one improved version of their code.
- End with a 1-line “tomorrow’s adjustment note”.

────────────────────────────────────────────────────────────
TOKEN-SAVING DUTY (CRITICAL)
You must minimize token usage while maintaining usefulness.

YOU HAVE EXTERNAL TOOLS (MCP SERVERS)
You may have access to MCP servers such as:
- EXA MCP (web search + tight snippet retrieval)
- ref.tools MCP (documentation lookup)
- (optional) Firecrawl/Playwright (extraction from dynamic pages)
- (optional) Git MCP (selective repo/file reading)
- (optional) E2B MCP (execute code/tests and return short outputs)
- (optional) Redis/Postgres MCP (cache/user progress + summaries)

TOKEN-SAVING OPERATING PATTERN
1) Retrieve-first, then reason:
   - If the user asks anything that benefits from external info (library behavior, new syntax, docs, best practice claims), call the appropriate MCP tool to fetch minimal relevant snippets FIRST.
   - Only then write the answer using those snippets.
   - Never paste long pages. Never "guess" when a snippet can be retrieved.

2) Aggressive dedupe:
   - If multiple sources repeat the same content, keep the single best source (+ 1 secondary max).
   - Avoid repeating the same idea twice in one digest.

3) Hard caps on context:
   - Cap tool outputs to the smallest useful sections (top 1–3 passages, shortest relevant chunk).
   - Prefer bullet summaries of tool results.

4) Cache to avoid re-sending:
   - Maintain and reuse compact memory: user level estimates, recurring pitfalls, "already covered concepts".
   - If storage MCP exists, store:
     - daily concept IDs, exercise results, common mistakes,
     - source → short abstract (so repeated topics cost near-zero).

5) Execute instead of over-explaining:
   - When debugging or validating, prefer running tests/code via E2B (if available) and report only pass/fail + minimal diff + 1–3 insights.
   - Do not produce long speculative debugging narratives.

6) Keep outputs short by design:
   - The digest must stay <5 minutes to read.
   - Solutions/feedback: focus on the single highest-impact fix, not exhaustive rewrites.

TOOL USAGE RULES
- Use tools only when they reduce tokens or increase accuracy.
- Never fabricate sources, outputs, or tool results.
- If tools are unavailable, state assumptions briefly and keep guidance conservative.

SAFETY / INTEGRITY
- Don’t recommend unethical data use or bypassing paywalls/access controls.
- Don’t present speculation as fact.

If the user specifies constraints (pandas-heavy, backend-heavy, interview prep, preferred stack), adapt the content while keeping the same digest structure and the token-saving pattern.

```

If you have doubts, watch [AJ's video](https://x.com/techfrenAJ/status/2014934471095812547/video/1) for a full walkthrough.

## Part 4. Connecting to OpenClaw's Control UI
1. Locate the `.pem` key of your EC2 instance.
2. Open a Linux/WSL terminal on your local machine.
3. Run:

   `sudo ssh -L 18789:127.0.0.1:18789 -i <your .pem file path> ubuntu@<YOUR_INSTANCE_PUBLIC DNS>`

This creates an SSH tunnel that exposes OpenClaw's dashboard at `0.0.0.0:18789`.

4. Open a browser on your local machine and go to `127.0.0.1:18789`. The dashboard should load.

If you see an error, configure the token.

5. On the EC2 instance, copy the **token** from `~/.openclaw/openclaw.json`.
6. Rapid (less secure) solution:
   - Use a private/incognito browser window because you will pass the token in the URL.
   - Browse to `127.0.0.1:18789/?token=<PASTE_YOUR_AUTH_TOKEN>`.
   - Bookmark or save the address carefully.

You should now have access to the OpenClaw dashboard :grin:

## Part 5. Installing MCP servers in OpenClaw
Install OpenClaw's `mcporter` skill to connect to MCP servers easily.

We recommend you to install OpenClaw's *mcporter* skill, to be able to connect to MCP servers easily with OpenClaw.

We recommend you to install OpenClaw's *mcporter* skill, to be able to connect to MCP servers easily with OpenClaw.

In your EC2 instance terminal, run:

`npm install -g mcporter`


# Part 5. Installing MCP servers in OpenClaw

We recommend you to install OpenClaw's *mcporter* skill, to be able to connect to MCP servers easily with OpenClaw.

In your EC2 instance terminal, run:

`npm install -g mcporter`

### 5.1 Exa MCP
Before starting to chat, we must optimize the web searches of the bot. A good idea is to connect it to Exa MCP server:
Before starting to chat, we must optimize the web searches of the bot. A good idea is to connect it to Exa MCP server:


Tell the bot:

```text
Create a skill by wrapping this MCP:
https://mcp.exa.ai/mcp?tools=web_search_exa,web_search_advanced_exa,get_code_context_exa,deep_search_exa,crawling_exa,company_research_exa,linkedin_search_exa,deep_researcher_start,deep_researcher_check
```

### 5.2 Git MCP

GitMCP can expose any GitHub repo as an MCP server. Visit [https://gitmcp.io/](https://gitmcp.io/) to convert repositories.

#### 5.2.1 Example: Give access to github repos about python on Data

```text
I want you to add a skill and wrap it around the following MCP Git repos with Python courses [https://gitmcp.io/darshilparmar/python-for-data-engineering https://gitmcp.io/jakevdp/PythonDataScienceHandbook https://gitmcp.io/dabeaz-course/python-mastery].

Here is some information for your SKILL.md file:
[

GIT MCP SOURCES (COURSE PACK)
You have access to a Git MCP server endpoint that exposes these repositories:

1) Python for Data Engineering (practice + DE patterns)
- https://gitmcp.io/darshilparmar/python-for-data-engineering

2) Python Data Science Handbook (NumPy/Pandas/ML notebooks)
- https://gitmcp.io/jakevdp/PythonDataScienceHandbook

3) Python Mastery (advanced Python exercises/curriculum)
- https://gitmcp.io/dabeaz-course/python-mastery

### MCP Resources (Python)

Added via `mcporter` for daily lessons and on-demand lookups:

- **python-data-eng**: Darshil Parmar's Python for Data Engineering
- **python-ds-handbook**: Jake VanderPlas's Data Science Handbook
- **python-mastery**: David Beazley's Advanced Python Mastery

**Usage Examples:**
- `mcporter call python-mastery.search_python_mastery_docs query="metaclasses"`
- `mcporter call python-ds-handbook.search_repo_code query="pandas join"`
- `mcporter call python-data-eng.fetch_repo_docs`



]
```

# Part 6. Setting up a cron job

We can set up a cron job in OpenClaw by two ways:

- Manually in the Dashboard UI (http://127.0.0.1 -> go to **Cron jobs**) or with the openclaw CLI






- Manually in the Dashboard UI (http://127.0.0.1 -> go to **Cron jobs**) or with the openclaw CLI


```bash
openclaw cron add <json-string>
```

The json must look like:

The json must look like:


```json
openclaw cron add '{
  "name": "hourly-stretch",
  "schedule": {
    "kind": "cron",

    "expr": "0 * * * *", 


    "expr": "0 * * * *", 

    "tz": "UTC"
  },
  "sessionTarget": "isolated",
  "payload": {
    "kind": "agentTurn",

    "to": "<SET_TG_USER_ID>", 


    "to": "<SET_TG_USER_ID>", 

    "channel": "telegram",
    "deliver": true,
    "message": "Send me a quick message reminding me to stretch and drink water."
  }
}'




```
- Asking the bot to do it for you!
- Asking the bot to do it for you!


```
Create a cron job every day at 9 AM. Use ONLY the MCP references I gave you regarding python courses on ML, Data engineering and advanced coding. Reference the source and change the lesson content each day.
```


An example of my daily python lesson cron job description is this (generated by a gemini-3-pro model :grin:):


An example of my daily python lesson cron job description is this (generated by a gemini-3-pro model :grin:):


```md
Generate an INTERMEDIATE-LEVEL technical lesson for <YOUR_USER_NAME> on **Python Development**. **📚 SOURCE MATERIAL REQUIREMENT:** You **MUST** use the `mcporter` tool to search or fetch content from one of the following configured servers: - `python-mastery` (David Beazley's course) - `python-ds-handbook` (Jake VanderPlas) - `python-data-eng` (Darshil Parmar) **Find a specific, practical concept or code pattern in these repos and base the lesson on it.** Do not generate generic content from memory. **Lesson Content:** Focus on practical, real-world concepts like decorators, context managers, effective use of stdlib, testing strategies, or data engineering pipelines. Use a Senior Engineer mentor persona. **Structure:** - **SOURCE:** (Link or ref to the specific file/repo you used) - **THE CONCEPT** - **PRACTICAL EXAMPLE** - **THE CODE** (Adapted from the source) - **PRO TIP** Compact the lesson to ~250 words. Format for Telegram (bold caps headers, bullet points, code blocks).
```


This is the result :wink:. As you can see, we have a reference to a real document in case we want to deep dive into the concept within the course :smiley:

This is the result :wink:. As you can see, we have a reference to a real document in case we want to deep dive into the concept within the course :smiley:

![python-lesson-output](/assets/python-lesson-output.png)


### NOTES - more MCP courses

I have added to my daily tailored digests, these courses about Linux systems and AI.







### NOTES - more MCP courses

I have added to my daily tailored digests, these courses about Linux systems and AI.


#### About Linux systems

```

Now to add to the lessons, about Linux systems, wrapp up these MCPs (https://gitmcp.io/0xAX/linux-insides https://gitmcp.io/bootlin/training-materials https://gitmcp.io/koansoftware/lkmpg)

Now to add to the lessons, about Linux systems, wrapp up these MCPs (https://gitmcp.io/0xAX/linux-insides https://gitmcp.io/bootlin/training-materials https://gitmcp.io/koansoftware/lkmpg)
```

Now to add to the lessons, about Linux systems, wrapp up these MCPs (https://gitmcp.io/0xAX/linux-insides https://gitmcp.io/bootlin/training-materials https://gitmcp.io/koansoftware/lkmpg)
```

Now to add to the lessons, about Linux systems, wrapp up these MCPs (https://gitmcp.io/0xAX/linux-insides https://gitmcp.io/bootlin/training-materials https://gitmcp.io/koansoftware/lkmpg)
```

Now to add to the lessons, about Linux systems, wrapp up these MCPs (https://gitmcp.io/0xAX/linux-insides https://gitmcp.io/bootlin/training-materials https://gitmcp.io/koansoftware/lkmpg)
```

Now to add to the lessons, about Linux systems, wrapp up these MCPs (https://gitmcp.io/0xAX/linux-insides https://gitmcp.io/bootlin/training-materials https://gitmcp.io/koansoftware/lkmpg)
```

Now to add to the lessons, about Linux systems, wrapp up these MCPs (https://gitmcp.io/0xAX/linux-insides https://gitmcp.io/bootlin/training-materials https://gitmcp.io/koansoftware/lkmpg)
```
#### About AI

RAG, Agents, MCP, etc

```
Now to add to the lessons, about AI, wrapp up these MCPs
(https://gitmcp.io/decodingai-magazine/second-brain-ai-assistant-course https://gitmcp.io/nvidia-ai-technology-center/multi-scale-agentic-rag-playbook https://gitmcp.io/towardsai/ai-tutor-rag-system) 
```

### Insight :bulb:

Here is the token economy of using **MCP** (called *mcporter* skill in OpenClaw) versus the alternatives:

1. Search (Cheap): The agent sends a small query (e.g., search_repo_docs("decorators")). The MCP server does the heavy lifting via GitHub's API and returns only titles and small snippets. This costs very few tokens.
2. Fetch (Targeted): Based on the search, the agent requests one specific file (e.g., Exercise 7.1 in logcall.py). It reads only that file, not the entire repository or course.
3. Generation: The model uses that single file as context to write the lesson.


### Other MCP
Browse additional MCP servers on [mcpmarket.com/leaderboards](https://mcpmarket.com/leaderboards). :warning: Use caution—MCP tools can introduce data leakage risks.

## Part 7. Recommended models
Smarter models handle complex tasks better, but choose the most cost-effective option for your use case. For Telegram chatting plus MCP lookups, inexpensive reasoning models usually suffice.

### Local models
Try `Ollama` and match the model size to your GPU. In this [video](https://www.youtube.com/watch?v=Idkkl6InPbU), a 30B parameter `GLM-4.7-flash` model from zAI-org is used with a `48 GB NVIDIA A6000 GPU`; scale down if your hardware is smaller.

### Gemini
`Gemini 2.5 Flash Lite` works very well :grin:. Use `Gemini 3 Pro` if you need more reasoning power (for example, heavy MCP tool calling or token optimization).

### Gemini
`Gemini 2.5 Flash Lite` works very well :grin:. Use `Gemini 3 Pro` if you need more reasoning power (for example, heavy MCP tool calling or token optimization).

### Anthropic
For low-cost reasoning and tool calling, **Claude Haiku 4.5** is an excellent option.

## Local models

Try with `Ollama` and select models as big as possible regarding your GPU capabilities. For instance, on this [video](https://www.youtube.com/watch?v=Idkkl6InPbU) he selected a 30B parameters `GLM-4.7-flash` model from zAI-org. However, he has a `48 GB NVIDIA A6000 GPU`, so it is not a good reference if you have less than that on your local machine.

## Gemini

I have tried `Gemini 2.5 Flash lite` and it works like a charm! :grin:
If you want a smarter (but more expensive) model you can go for `Gemini 3 Pro`. I highly recommend the smarter model if you are trying to do a complex task (such as using mcporter for calling a tool from an MCP server and optimize your tokens.)

## Anthropic

For low-cost reasoning and tool calling, **Claude Haiku 4.5** is your best option.

**Haiku 4.5 offers**:

- Full tool calling support - can use functions/tools just like the other models
- Strong reasoning abilities - significantly improved over previous Haiku versions

**Haiku 4.5 offers**:

- Full tool calling support - can use functions/tools just like the other models
- Strong reasoning abilities - significantly improved over previous Haiku versions

- Lowest cost in the Claude 4.5 family
- Fast responses for high-volume workloads

If you need deeper reasoning or more complex tool flows, test **Claude Sonnet 4.5** and compare cost/performance. Start with Haiku 4.5 and upgrade only if you hit limitations.


# Part 8. Advanced topics

# Part 8. Advanced topics

## 8.1 Building Docker image in OpenClaw sandbox

# Part 8. Advanced topics



## 8.1 Building Docker image in OpenClaw sandbox

In the EC2 instance, copy the `/config` folder to `$HOME`.

In the EC2 instance, copy the `/config` folder to `$HOME`.

IMPORTANT:  copy the `/config` folder in this repo to `$HOME`

1. In `$HOME` directory, execute: 

```bash
docker build docker -t <YOUR_IMAGE_NAME> -f ~/.openclaw/sandbox/Dockerfile.sandbox.skills ~/.openclaw/sandbox
```

2. Update `openclaw.json` in `~/.openclaw/openclaw.json`.
3. Rebuild the sandbox image: `openclaw sandbox recreate --all`.



## 8.2 Testing sandbox environment








## 8.2 Testing sandbox environment


```bash
openclaw agent --to agent:main:test --message "Run: which obsidian-cli && brew --version"
```

If the dependencies are installed and set, they should be available within the sandbox environment.


If the dependencies are installed and set, they should be available within the sandbox environment.
