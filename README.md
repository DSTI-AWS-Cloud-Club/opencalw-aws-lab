# moltbot-aws-lab
This lab deploys MoltBot in an EC2 instance and sets up Telegram as an assistant Bot.

We will ask the bot to give us the last trends in Data, AI agents and Machine Learning and we will message him through our Telegram account! :grin: 

**NB**: MoltBot was initially called ClawdBot but Anthropic had issues with the name due to its LLM Claude. That is why the CLI is `clawdbot` for the moment (jan 2026).

### Pre-requisites
- Having an AWS Free Tier account
- Have Telegram installed on your phone
- Having a Claude API key (you MUST pay at least $5 to have one) or a Gemini API key ($300 for FREE during 90 days in AI Studio). 
    - You can create one [in Claude Platform](https://platform.claude.com/).
    - You can create your Gemini API key in [AI Studio](https://aistudio.google.com/api-keys)

## Resources
### Clawdbot/MoltBot
- If you want to know how people use MoltBot visit its [website](https://clawd.bot/)
- Video explaining how to install MoltBot in an EC2 instance. [Video from AJ](https://x.com/techfrenAJ/status/2014934471095812547/video/1)
- To configure a protection against injection attacks follow up the [tutorial](https://github.com/Dicklesworthstone/acip/tree/main/integrations/clawdbot) by Dicklesworthstone.
- If you want to install a local Ollama model to be used by Moltbot go and visit this [video](https://www.youtube.com/watch?v=Idkkl6InPbU)

### About MPC
- A [video](https://www.youtube.com/watch?v=w-rXEUTOIas) for providing MPCs to your Vibe tool: Codex, Cursor, Claude Code or Droid.


### Gemini
- Gemini API, create an account in [AI Studio](https://aistudio.google.com/api-keys) and receive $300 in credits **for FREE** for *90 days*. In the **Overview** page you will be able to monitor the use of your $300 :grin:
- Gemini API [Prices](https://ai.google.dev/gemini-api/docs/pricing)


# Part 1. Creating an EC2 instance and installing MoltBot

## 1.1 Creating an EC2 instance in AWS Free Tier account

1. In AWS Dashboard, navigate to **EC2 Console** 
2. Select *Instances* and *Launch Instances*
3. Name: MoltBot
4. OS: Select *Ubuntu*, with *Ubuntu Server 24.04*
5. Instance type: **t2.micro** or **t3.micro** (Free Tier eligible)
    You can check the eligibility of your Free tier account by executing this in a Terminal:
    ```bash
    aws ec2 describe-instance-types \
    --filters Name=free-tier-eligible,Values=true \
    --query "InstanceTypes[*].[InstanceType]" \
    --output text | sort
    ```
6. Key pair: Generate a .pem key pair (we will use it later for the ssh tunnel)
7. Storage: Since we are not going to install local LLMs, `8 GB` is enough. If we were to install *Ollama*, we would need up to `60 GB` depending on the model we want to install locally.
8. Launch Instance
9. Go to **Instances**. Select the instance and click **Connect**
10. Then click **Connect** and a web terminal will be open with the Ubuntu instance.

## 1.2 Installing MoltBot

1. Inside the EC2 instance, execute the line:

`curl -fsSL https://molt.bot/install.sh | bash`

2. This will download MoltBot, wait a few minutes for the installer to pop up

![moltbot-onboard](/assets/moltbot-onboard.JPG)

3. Before setting everything up, say **No** first and reboot the instance, so we can have access to `clawdbot` CLI.

4. Reboot the instance `sudo reboot`. Wait for some seconds and refresh the connection
5. Check that clawdbot is installed `clawdbot help`

**Before going any further. Stop the bot `ctrl + C` and do the security installs**

# Part 2. SECURITY INSTALLS
**Protecting your personal information**

As we give rights to connect to our personal data, mail, phone, etc the security is vital.

We should go through several security checks.

See the X message from MoltBot creator Peter Steinberg.

![security-checks](/assets/security-clawdbot.JPG)

## 2.1 Injection attacks
One of the first things to configure in MoltBot is a protection against injection

Go to this [tutorial](https://github.com/Dicklesworthstone/acip/tree/main/integrations/clawdbot) by Dicklesworthstone.

## 2.2 Add a sandbox configuration (Skip for now)

You can isolate each session inside a sandbox. The sessions will run in a docker container, isolating each session.

```bash
nano ~/.clawdbot/clawdbot.json
```

Paste **before the compaction part**

```json
----- COPY THIS PART ONLY --------
"sandbox": {
       "mode": "off", // you can turn it to all if you want full sandbox isolation
       "scope": "session",
       "workspaceAccess": "none"
       },
------------------
 # DO NOT COPY THIS
      "compaction": {
        "mode": "safeguard"
``` 

#### 2.2.1 Install docker in Ubuntu (skip if no sandbox used)
We need Docker installed in the machine to be able to activate the Sandbox option. The sandbox option will run a container per-session so it will isolate the machine from access by an intruder. 

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

#### 2.2.2 Finding problems with docker.
Maybe a solution is to add the user to the docker group.

```bash
# 1) Make sure Docker is installed and running
sudo systemctl enable --now docker
sudo systemctl status docker --no-pager

# 2) Add your current user to docker group
sudo usermod -aG docker $USER

# 3) Apply new group membership (choose ONE)
newgrp docker
# or log out and log back in (SSH: disconnect/reconnect)

# 4) Test
docker ps
```

#### 2.2.3 Restart 
To apply changes:

`clawdbot gateway restart`

And restart the instance:

`sudo reboot`

## 2.3 Run an audit
Every time you add a new feature (e.g. connection to Telegram), you can run a security audit to ask MoltBot if everything is safe.

Read the details in [here](https://docs.molt.bot/cli/security)

`clawdbot security audit`

`clawdbot security audit --deep` (to have a deeper insight in the audit)

`clawdbot security audit --fix` (to fix the security problems)

## 2.4 Security Notes

To have more insights on security issues in MoltBot, watch this [video](https://www.youtube.com/watch?v=AbCHaAeqC_c) and implement the step by step.

:warning: Be careful about what you permit your bot to have access to! If you suffer sensitive data leakage, it is at your own responsibility!

To avoid any problems with your sensitive data:
- Read the [documentation](https://docs.molt.bot/)
- Research about how to best configure your new feature before deploying it
- Be aware of what the bot will have access to and also other agents interacting with the bot.

# Part 3. Configuring MoltBot

## 3.1 Creating an Anthropic API KEY
1. Go to [Claude Platform](https://platform.claude.com/).
2. Create an account and add a billing payment method
3. Go to API keys -> **Create key**
4. Give it a name and copy it. 
5. Save it in secure place.

## 3.2 Setting up MoltBot

1. Run the onboard interface
`clawdbot onboard`
2. Select `Yes` 
3. Select `Quick Start`
4. Select 
    - if you have an Anthropic API key: `Anthropic -> Anthropic API KEY`
    - if you have a Google Gemini API key: `Google -> Gemini API KEY`
5. Paste your API KEY here and tap ENTER
6. Select `Haiku 4.5 (latest --k reasoning)`

## 3.3 Setting up Telegram
7. In the onboard interface
8. Select `Telegram`
  - Open Telegram and chat with @BotFather             
  - Run /newbot (or /mybots)  
  - Send a unique name for your bot: `MyAwesomeAssistant`                          
  - Copy the token (looks like 123456:ABC...)       
9. Paste your token id and tap ENTER
    9.1 Now open your Telegram App in your phone and search for `@YOUR_BOT_NAME`. Click `START` and you will receive a message like this:

        ```
        Clawdbot: access not configured.

        Your Telegram user id: ________

        Pairing code: __________

        Ask the bot owner to approve with:
        clawdbot pairing approve telegram <code>
        ```

    9.2 Copy the entire message and paste it into the EC2 instance terminal.
    9.3 You can now chat with your Bot! :alien:

10. For the rest of the questions, say **No / Skip for now**.

## 3.4 Giving a duty and instructions to the Bot

11. :warning: Stop when it asks you about **Hatch in TUI** 
12. Select **Hatch in TUI**
13. Now we are starting to chat to our bot
14. When he asks you your name, give it e.g. `DSTI`
15. Then he will ask you about his personality. Here you define the task of the bot.

Since we are going to use this bot to look for trends in Tech, we will define the following:

First, give the bot a name:
```
Your name is <YOUR_BOT_NAME>.
``` 

Then, define its soul and duty.

```text
You are “PySprint”, a daily Python coaching bot for a Data Scientist / Data Engineer.

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
Avoid long lectures, too many new concepts/day, and unnecessary math.

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
   - If the user asks anything that benefits from external info (library behavior, new syntax, docs, best practice claims),
     call the appropriate MCP tool to fetch minimal relevant snippets FIRST.
   - Only then write the answer using those snippets.
   - Never paste long pages. Never “guess” when a snippet can be retrieved.

2) Aggressive dedupe:
   - If multiple sources repeat the same content, keep the single best source (+ 1 secondary max).
   - Avoid reporting the same idea twice in one digest.

3) Hard caps on context:
   - Cap tool outputs to the smallest useful sections (top 1–3 passages, shortest relevant chunk).
   - Prefer bullet summaries of tool results.

4) Cache to avoid re-sending:
   - Maintain and reuse compact memory: user level estimates, recurring pitfalls, “already covered concepts”.
   - If storage MCP exists, store:
     - daily concept IDs, exercise results, common mistakes,
     - source → short abstract (so repeated topics cost near-zero).

5) Execute instead of over-explaining:
   - When debugging or validating, prefer running tests/code via E2B (if available) and report only:
     pass/fail + minimal diff + 1–3 insights.
   - Do not produce long speculative debugging narratives.

6) Keep outputs short by design:
   - The digest must stay <5 minutes to read.
   - Solutions/feedback: focus on the single highest-impact fix, not exhaustive rewrites.

TOOL USAGE RULES
- Use tools only when they reduce tokens or increase accuracy.
- Never fabricate sources, outputs, or “tool results”.
- If tools are unavailable, state assumptions briefly and keep guidance conservative.

SAFETY / INTEGRITY
- Don’t recommend unethical data use or bypassing paywalls/access controls.
- Don’t present speculation as fact.

If the user specifies constraints (pandas-heavy, backend-heavy, interview prep, preferred stack), adapt the content while keeping the same digest structure and the token-saving pattern.

```

If you have any doubts go to the [video from AJ](https://x.com/techfrenAJ/status/2014934471095812547/video/1) where he explains it step-by-step.

# Part 4. Connecting to Moltbot's Control UI
1. Take the `.pem` key of your EC2 instance
2. Open a linux/WSL terminal in your local machine
3. type:

`sudo ssh -L 18789:127.0.0.1:18789 -i <your .pem file path> ubuntu@<YOUR_INSTANCE_PUBLIC DNS>`

This will create an SSH tunnel listening on Clawdbot's dashboard address at `0.0.0.0:18789`

4. Open a browser in your local machine and type `127.0.0.1:18789`. You should see the Dashboard!

However, it throws an error...we should configure the token.

5. In the EC2 instance, copy the **token** from the config file:

You can obtain it by typing: `cat ~/.clawdbot/clawdbot.json`

This *gateway token* permits you to connect to the dashboard. :warning: **Save this token as a password in a safe place**

6. Rapid solution (not the safest): 
  - Open a **private browser** :warning: It is important to be a private browser as we are adding the token on the address
  - In your browser type the following address `127.0.0.1:18789/?token=<PASTE_YOUR_AUTH_TOKEN>` 
  - be careful to bookmark or save this address

You should have access now to Clawdbot's Dashboard :grin:

## Installing MCP servers in our bot

### Exa MCP
Before starting to chat, we must optimize the web searches of the bot. A good idea is to connect it to Exa MCP server:

- Install MCP Exa (web searches). More info [here](https://mcpmarket.com/server/exa)

Tell the bot to:

```text
Create a skill by wrapping this MCP:
https://mcp.exa.ai/mcp?tools=web_search_exa,web_search_advanced_exa,get_code_context_exa,deep_search_exa,crawling_exa,company_research_exa,linkedin_search_exa,deep_researcher_start,deep_researcher_check
```

### Git MCP
With GitMCP we can convert any repo into an MCP server.

To do that, we have selected three.
Send this to the chat bot:

```text
GIT MCP SOURCES (COURSE PACK)
You have access to a Git MCP server endpoint that exposes these repositories:

1) Python for Data Engineering (practice + DE patterns)
- https://gitmcp.io/darshilparmar/python-for-data-engineering

2) Python Data Science Handbook (NumPy/Pandas/ML notebooks)
- https://gitmcp.io/jakevdp/PythonDataScienceHandbook

3) Python Mastery (advanced Python exercises/curriculum)
- https://gitmcp.io/dabeaz-course/python-mastery

HOW TO USE GIT MCP (TOKEN-SAVING RETRIEVAL RULES)
- Goal: build the daily digest using minimal repo slices (never load large notebooks end-to-end).
- Every day, choose exactly ONE “primary source repo” based on the user’s current focus:
  - If user focus = Data Engineering → prefer python-for-data-engineering
  - If user focus = pandas/numpy/ML tooling → prefer PythonDataScienceHandbook
  - If user focus = core/advanced Python fluency → prefer python-mastery
- Retrieve only what you need:
  1) List directories (high-level) to locate the best small lesson/exercise file.
  2) Read at most ONE lesson/exercise file (or an index + one exercise).
  3) If a file is long, extract only the relevant section and summarize it.
- Hard caps:
  - Max 1–2 file reads per daily digest.
  - Prefer short markdown/text files over notebooks; avoid big .ipynb reads unless absolutely necessary.
  - Do not include large code dumps in the digest. Keep examples 6–12 lines.
- Deduplicate:
  - If multiple files cover the same concept, pick the smallest/clearest one.
- Cache what you used:
  - Track (repo, path, concept tag, date) so you do not reuse the same file too soon unless it’s a review day.

DAILY CONTENT CONSTRUCTION USING GIT MCP
- Step 1: Decide today’s focus (one of):
  A) Clean Python fundamentals & idioms
  B) DS tooling (NumPy/pandas)
  C) DE patterns (IO, data formats, pipeline-like code)
  D) Reliability (tests, error handling, logging)
  E) Performance (profiling mindset, vectorization, algorithmic improvements)
- Step 2: Use Git MCP to fetch minimal context from the best repo for that focus.
- Step 3: Generate the PySprint Daily digest:
  - Micro-concept must be grounded in the retrieved content.
  - The micro-exercise should be original but aligned with the retrieved lesson theme.
  - Add a short “Source” link to the specific file(s) you used.

SOURCE CITATION RULES (IMPORTANT)
- Always cite the exact Git MCP URL(s) for the file(s) you read.
- Never claim you used repo content unless you actually retrieved it.
- If Git MCP retrieval fails/unavailable, proceed with a conservative, standard-library based lesson and label it as “no source today”.

ROTATION STRATEGY (TO AVOID REPETITION)
- Prefer a weekly rotation unless the user requests otherwise:
  - Mon/Wed/Fri: python-mastery (core/advanced Python)
  - Tue/Thu: PythonDataScienceHandbook (NumPy/pandas patterns)
  - Sat: python-for-data-engineering (DE practice)
  - Sun: Review day (repeat a weak area with a new exercise)
- Adapt rotation automatically based on user performance and preferences.

OUTPUT MUST REMAIN <5 MIN READ
- Do not include more than:
  - 5 bullets in TL;DR (if present)
  - 12 lines of code in the example
  - 4 self-check bullets/asserts
  - 1 stretch item
```

### Other MCP

Check out other MCP servers that could be useful for another use of your bot [here](https://mcpmarket.com/leaderboards). 
 :warning: Beware! These MCP tools can lead to personal data leakages, use them wisely!



# Recommended models

For low-cost reasoning and tool calling, **Claude Haiku 4.5** is your best option.

**Haiku 4.5 offers**:

- Full tool calling support - can use functions/tools just like the other models
- Strong reasoning abilities - significantly improved over previous Haiku versions
- Lowest cost in the Claude 4.5 family
- Fast response times - ideal for high-volume applications

It's designed specifically for use cases where you need reliable performance on tasks like tool use, data extraction, and structured reasoning, but want to optimize costs. For many common agentic workflows and tool-calling tasks, Haiku 4.5 performs very well while being substantially cheaper than Sonnet or Opus.
That said, if your tasks require **more complex reasoning**, nuanced judgment, or handling particularly challenging tool-calling scenarios, you might want to test Haiku against **Claude Sonnet 4.5** to see if the cost-performance tradeoff makes sense for your specific use case.
I'd recommend starting with Haiku 4.5 and only upgrading if you hit limitations.


