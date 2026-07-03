# Getting Started: ROSMASTER M3 Pro + OpenClaw AI (Jetson Orin Nano 8GB)

A simple, start-to-finish guide to bring your **already-assembled** Yahboom
ROSMASTER M3 Pro online and interact with it using the **OpenClaw** AI
integration (chat + voice command of the robot).

> **What is OpenClaw?** It's a self-hosted AI *gateway* that runs on the robot.
> It connects a large language model (Claude, GPT, Gemini, Qwen, etc.) to the
> robot's body through an **MCP service**, so you can talk to the robot in plain
> language and it can reason, answer, and drive its hardware (chassis, arm,
> camera). You reach it from a **web page**, a **terminal (TUI)**, or by
> **voice**.

This guide is written specifically for the **Jetson Orin Nano 8GB** board, which
is the simplest case: the code lives directly on the board (no Docker), and the
AI agent + OpenClaw gateway **auto-start on boot**.

Source material for every step is in this repo — folder references are given as
`[00. ... .pdf]` so you can go deeper.

---

## At a glance

| Item | Value |
|---|---|
| Board | Jetson Orin Nano 8GB (Ubuntu 22.04, ROS 2 Humble) |
| Login user / password | `jetson` / `yahboom` |
| Factory hotspot | SSID `ROSMASTER`, password `12345678`, IP `192.168.8.88` |
| Code location (Orin) | `/home/jetson` (no Docker container needed) |
| OpenClaw config file | `$HOME/.openclaw/openclaw.json` |
| OpenClaw web UI | `http://<robot-ip>:18789`  (gateway token: `yahboom`) |
| OpenClaw workspace (prompts) | `$HOME/.openclaw/workspace` |
| Jupyter Lab | `http://<robot-ip>:8888` (password `yahboom`) |

---

## Step 1 — Power on and find the robot's IP

1. Switch the robot on and wait ~1 minute for it to fully boot.
2. Read the **IP address shown on the robot's OLED screen**.
   - If you plug in an Ethernet cable, the OLED updates to that IP automatically.
   - With no cable, the robot broadcasts its own Wi-Fi hotspot: **`ROSMASTER`**,
     password **`12345678`**, default IP **`192.168.8.88`**.

Your computer and the robot must be on the **same network** to connect.

*Ref: `[00.Configuration and Operation Guide / 2. Log in to the car and view the code.pdf]`*

---

## Step 2 — Log in to the desktop with VNC

The AI features need a graphical desktop, so use **VNC** (not just SSH).

1. Install **RealVNC Viewer** on your computer.
2. Connect to the robot's IP (e.g. `192.168.2.92`).
3. Log in with user **`jetson`**, password **`yahboom`**.

> Orin can only hold **one** VNC session at a time. If it won't connect, an old
> session is probably still open.
>
> **Orin with no monitor attached?** A headless Orin needs a one-time virtual-
> desktop switch before VNC will show a picture — follow section *1.2.1* of the
> "Log in to the car" PDF. If you have a monitor on the board, skip that.

*Ref: same PDF as Step 1. SSH details are in `[21.Main Control Course / Jetson Orin Nano_NX / 6.SSH remote login.pdf]`.*

---

## Step 3 — Put the robot on your real Wi-Fi (required for AI)

The factory hotspot is a **local network with no internet**. OpenClaw calls
cloud AI models, so the robot needs real internet.

1. In the VNC desktop, click the network icon (top-right).
2. Turn off the robot's own hotspot and connect it to **your home/office Wi-Fi**.
3. The OLED will show a **new IP** — note it. Reconnect VNC to that new IP.

Confirm internet from a terminal on the robot:

```bash
ping -c 3 www.google.com
```

*Ref: `[21.Main Control Course / Jetson Orin Nano_NX / 5.Network configuration.pdf]`.*

---

## Step 4 — Sanity-check the robot base is alive (optional but recommended)

On the Orin, the low-level **communication agent auto-starts on boot**. If you
ever need to (re)start it manually, open a terminal and run:

```bash
sh start_agent.sh
```

A successful connection prints agent/serial status. This agent is what lets ROS
(and later OpenClaw) actually move the hardware.

If your arm looks misaligned, run the one-time **arm calibration** before doing
grasping tasks:

```bash
python3 ~/calibrate_arm.py
```

*Ref: `[00.Configuration and Operation Guide / 3. Robotic Arm Calibration.pdf]`.*

---

## Step 5 — Give OpenClaw an AI model API key

OpenClaw needs a model provider key to think. The factory config is pre-wired
for Alibaba **Bailian/Qwen** (mainland China). Since you're overseas, the
easiest good options are **Anthropic (Claude)**, **OpenAI (GPT)**, or **Google
(Gemini)** — pick one, get an API key from that provider, then edit the config.

1. Open the config file on the robot:

   ```bash
   nano $HOME/.openclaw/openclaw.json
   ```

2. Put your key in the matching provider slot and set the default model. A
   minimal Claude example:

   ```json
   {
     "models": {
       "providers": {
         "anthropic": { "apiKey": "sk-ant-XXXXXXXXXXXXXXXX" }
       }
     },
     "agents": {
       "defaults": {
         "model": { "primary": "anthropic/claude-sonnet-4-5" }
       }
     }
   }
   ```

   > Format is `provider/model-name` (e.g. `openai/gpt-4o`,
   > `google/gemini-1.5-pro`). For the **multimodal / vision** demos later, pick
   > a model that accepts image input — Claude, GPT-4o, and Gemini all do.

3. Save and exit nano: `Ctrl+O`, `Enter`, then `Ctrl+X`.

4. Apply it by restarting the gateway:

   ```bash
   openclaw gateway restart
   ```

5. Verify the model is connected:

   ```bash
   openclaw models list      # should show your model
   openclaw models status    # connection / latency / availability
   ```

*Ref: `[06.OpenClaw Basic Course / 3. OpenClaw API-KEY Configuration.pdf]`. Getting a
provider account: `[12.AI Model Development / 01.Register a model service provider account]`.*

---

## Step 6 — Talk to OpenClaw (pick one)

The OpenClaw gateway auto-starts on boot. Check it any time with
`openclaw gateway status` (start with `openclaw gateway start` if needed).

### Option A — Web chat (easiest)

1. First, whitelist your robot's address. Edit the config:

   ```bash
   nano $HOME/.openclaw/openclaw.json
   ```

   Find `gateway.controlUi.allowedOrigins` and add your **robot IP + `:18789`**
   (comma-separate multiple IPs). Save, then:

   ```bash
   openclaw gateway restart
   ```

2. On any computer on the same Wi-Fi, open a browser to:

   ```
   http://<robot-ip>:18789
   ```

3. Enter the gateway token **`yahboom`** and click **Connect**.
4. Pick your model in the top selector and start chatting. Type "Hello" — if it
   replies, OpenClaw is fully working. 🎉

*Ref: `[07.OpenClaw Interaction Method / 1. OpenClaw Webchat Interaction.pdf]`.*

### Option B — Terminal chat (TUI)

Great over SSH, no desktop needed. In a robot terminal:

```bash
openclaw tui
```

Type text to chat. Same session syncs with the web UI if the session ID matches.

*Ref: `[07.OpenClaw Interaction Method / 2. OpenClaw Tui Interaction.pdf]`.*

### Useful slash commands (web & TUI)

Send each as its own message:

| Command | What it does |
|---|---|
| `/model <name>` | Switch model for this session |
| `/new` or `/reset` | Start a fresh conversation |
| `/stop` | Stop the current response |
| `/compact` | Shrink context to save tokens |
| `/status` | Show current run status |
| `/help`, `/commands` | Help / list all commands |

---

## Step 7 — Voice interaction (optional, the fun part)

Let the robot listen and speak. Voice replies are **disabled by default** from
the factory, so turn the voice tag on first.

1. Enable the voice reply tag and restart the gateway:

   ```bash
   robot_config voice-tag add
   robot_config voice-tag status      # should say "Voice Tag ON"
   openclaw gateway restart
   ```

2. Start the **MCP service** (this is the bridge that lets the AI actually
   control the robot) in one terminal:

   ```bash
   ros2 launch m3pro_bringup mcp_service.launch.py
   ```

3. In a **second** terminal, start speech recognition:

   ```bash
   ros2 run multi_brains_pre asr_detect
   ```

4. In a **third** terminal, start the OpenClaw voice bridge (shows text + speaks
   the `<tts>...</tts>` part of replies):

   ```bash
   ros2 run multi_brains_pre openclaw_bridge
   ```

   > These three run in separate terminals on purpose — they stream live text
   > that a single launch file would hide.

5. Wake the robot by saying **"Hi, yahboom"** (or press the **spacebar** to
   wake it), then speak your request. It will answer and, when appropriate, act.

*Ref: `[07.OpenClaw Interaction Method / 3. OpenClaw Voice Dialogue Interaction.pdf]`.*

---

## Step 8 — Have the AI actually drive the robot

Chatting is step one; the reason OpenClaw is special is that the model can
**call robot tools** through the MCP service — moving the chassis, using the
camera to "see", and operating the 6-DOF arm/gripper.

- The robot's **abilities and rules** are defined as Markdown "prompt" files in
  `$HOME/.openclaw/workspace` (e.g. `AGENTS.md`, `SOUL.md`, `IDENTITY.md`,
  `TOOLS.md`, `USER.md`, `MEMORY.md`). To switch these to English:

  ```bash
  cd $HOME/.openclaw
  rsync -av --delete .workspace_en/ workspace/
  openclaw gateway restart
  ```

  *Ref: `[06.OpenClaw Basic Course / 5. OpenClaw System Prompt Words.pdf]`.*

- For **guided AI demos** (semantic command-following, multimodal vision, and
  vision + navigation + arm grasping), start the agent stack and run the
  examples. On the Orin:

  ```bash
  sh start_agent.sh                                   # if not already running
  ros2 launch multi_brains llm_agent_control.launch.py   # or the shortcut: multi_brains
  ```

  Then wake with "Hi, yahboom" and give a natural-language task, e.g.
  *"Move forward 1 meter, turn left 30 degrees, then do a little dance."*

  *Ref: `[13.AI Model - Voice Version / 1.Semantic understand and command follow.pdf]`
  and the later chapters in that folder (multimodal vision, SLAM navigation,
  arm grasping).*

---

## Quick troubleshooting

| Symptom | Fix |
|---|---|
| Web page won't open | `openclaw gateway status`; confirm URL is `http://<ip>:18789`; same Wi-Fi; port 18789 open |
| "Token verification failed" | Token is `yahboom` (case-sensitive); check `gateway.controlUi.authToken` in config |
| No reply / model error | `openclaw models list` + `openclaw models status`; re-check API key; `openclaw gateway restart` |
| API key invalid / 401 | Re-check key for stray spaces; confirm it hasn't expired; confirm account has balance |
| Rate limit / 429 | Provider quota hit — `/model <other>` to switch, or wait |
| Changed config, no effect | You must run `openclaw gateway restart` after editing `openclaw.json` |
| Voice tag not speaking | Re-run `robot_config voice-tag add`, restart gateway, confirm the MCP + asr + bridge terminals are all running |

More: `[07.OpenClaw Interaction Method / Common Errors And Solutions.pdf]`.

---

## Recommended learning path (folders in this repo)

1. `00.Configuration and Operation Guide` — connect, log in, view code, calibrate
2. `21.Main Control Course / Jetson Orin Nano_NX` — networking, SSH, VNC, jtop
3. `06.OpenClaw Basic Course` — OpenClaw concepts, API keys, models, CLI
4. `07.OpenClaw Interaction Method` — web chat, TUI, voice, WhatsApp
5. `11.AI Model Basics` & `12.AI Model Development` — RAG, Dify, agents, workflows
6. `13.AI Model - Voice Version` / `14.AI Model - Text Version` — full AI demos

---

*Questions on the hardware/tutorials: Yahboom support — support@yahboom.com.*
