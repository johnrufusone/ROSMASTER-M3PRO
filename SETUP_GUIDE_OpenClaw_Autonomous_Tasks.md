# Part 2: Autonomous Tasks with OpenClaw (Pick-and-Place, Navigation & More)

> **Prerequisite:** Finish [Part 1 — OpenClaw AI setup](SETUP_GUIDE_OpenClaw_OrinNano.md)
> first (robot online, on real Wi-Fi, OpenClaw model key configured, you can chat
> with it). This guide picks up from there and shows how to make the robot
> **carry out physical tasks on its own** — "pick up the orange block", "bring
> me the cube from the kitchen", scheduled patrols, and your own custom routines.

Written for the **Jetson Orin Nano 8GB** (code in `/home/jetson`, no Docker).

---

## 1. How autonomy works (the 30-second mental model)

Three layers turn a sentence into robot actions:

```
You (chat/voice)  →  OpenClaw LLM  →  Skill (workflow)  →  MCP tools  →  ROS 2  →  hardware
                       "what to do"    "the recipe"        "the verbs"
```

| Layer | What it is | Where it lives |
|---|---|---|
| **MCP tools** | ~27 low-level "verbs": `Move`, `Rotate`, `Pick`, `Place`, `SeeWhat`, `GetBbox`, `Navigation`, `TargetTrack`, `TTS`, … | `mcp_service` (a ROS 2 launch) |
| **Skills** | Task **workflows** — an ordered recipe telling the AI which tools to call, in what order, with error handling | `$HOME/.openclaw/workspace/skills/*/SKILL.md` |
| **Vision model (Dify)** | Detects/locates objects for grasping (the `GetBbox` step) | Dify service on the robot |

The AI runs the loop **perceive → plan → act → verify** by itself, guided by the
Skill. You never call tools directly — you just describe the goal.

### The 5 factory-preset Skills

| Skill | Triggers | What it does |
|---|---|---|
| 🦾 **arm-pick-and-place** | grasp / pick / place / grab / move | Observe → locate target → grasp → verify → place |
| ♻️ **waste-sorting** | sort / collect waste / classify | Recognize waste → grasp → navigate to correct bin → drop |
| 🧭 **robot-navigation** | (named locations) | Look up map → navigate to a place (up to 3 retries) |
| 👁️ **visual-functions** | observe environment / track target | Capture image → locate → visually track |
| ⚙️ **comprehensive-task** | sorting / replenishment / shelf ops | Chains the above for multi-stage jobs |

---

## 2. First autonomous task: "Pick up the object"

### 2.1 One-time prerequisites

1. **Calibrate the arm** (median + offset) — grasp accuracy depends on it.
   See `00.Configuration and Operation Guide / 3. Robotic Arm Calibration.pdf`.
2. **Dify running + a vision model key.** Object detection (`GetBbox`) runs
   through a vision model configured in Dify. Start Dify and set its model key:
   ```bash
   sh ~/bringup_dify.sh          # then browse to http://<robot-ip> to configure
   ```
   *(Refs: `12.AI Model Development / 02.Configuring API-KEY` and `/03.Introduction to Dify`.)*
3. **Enable the `arm-pick-and-place` skill:** in WebChat go to
   **Agent → Skills → Workspace Skills**, toggle it **on**, click **Save**.

### 2.2 Launch the robot-control stack

Open a few terminals on the robot (via VNC or SSH) and run, in order:

```bash
sh start_agent.sh                                   # 1. MicroROS chassis agent (auto-starts on boot)
ros2 launch m3pro_bringup car_base.launch.py        # 2. odometry, TF, arm assist, camera nodes
ros2 launch m3pro_bringup mcp_service.launch.py     # 3. MCP tools — gives OpenClaw its "body"
ros2 run multi_brains_pre openclaw_bridge           # 4. (optional) spoken replies
```

> **Why separate terminals?** These are long-running nodes. The MCP service in
> step 3 is the bridge that lets OpenClaw's tool calls reach the hardware — if
> it isn't running, OpenClaw can chat but can't move anything.

### 2.3 Give the command

Open WebChat (`http://<robot-ip>:18789`, token `yahboom`) or wake it by voice,
then say something like:

> **"Grasp the orange block in front of you."**

OpenClaw will, on its own:
1. `SeeWhat` — capture the camera view
2. `GetBbox` — ask the Dify vision model to locate the object (writes a check
   image so you can see what it found)
3. `AdjustChassisFitArmRange` — nudge the chassis so the target is reachable
4. compute a grasp point (writes a second debug image)
5. `Pick` — grasp, then verify

**Debug images** to inspect if something looks off:
```
~/M3Pro_ws/multi_brains_file/verify_image.png        # what the vision model selected
~/M3Pro_ws/multi_brains_file/grasp_point_image.png   # where the arm aimed
```

*Refs: `08.OpenClaw Development / 1. OpenClaw connects to the MCP interface`,
`/7. OpenClaw robotic arm control`; `09.OpenClaw Embodied intelligent application / 2. arm tracking and grasping`.*

---

## 3. Fetch-and-carry: "Bring me the cube from the kitchen"

This combines **arm-pick-and-place** with the **robot-navigation** skill, so it
needs a map and named locations first.

1. **Build a SLAM map** of your space and save it.
   *Ref: `09.OpenClaw Embodied intelligent application / 5. slam mapping and navigation`.*
   For a tabletop / sand-table setup, see folder `10.OpenClaw scenario sandbox`.
2. **Name locations** so the AI can navigate to them by word. You can do this by
   voice/chat — the `RecordMapLocation` tool saves the robot's current spot:
   > "Remember this spot as the kitchen."
   Retrieve/inspect mappings with `GetMapMapping`.
3. With the same control stack running (Section 2.2) **plus** navigation started,
   give a multi-step goal:
   > "Go to the kitchen, pick up the red cube, and bring it back to me."

   The AI plans the sequence (navigate → observe → grasp → navigate → place) and
   executes each step, retrying navigation up to 3 times if needed.

*Ref: `09.OpenClaw Embodied intelligent application / 6. navigation transfer`;
skill details in `08.OpenClaw Development / 8. skills development`.*

---

## 4. Unattended / scheduled tasks

OpenClaw can run tasks automatically on a schedule — no one at the keyboard.
Just describe it in chat:

> "Create a scheduled task: every 5 minutes, move forward 0.5 meters and observe the environment."

Manage them in **WebChat → Scheduled Tasks**:
- **Enable/Disable** toggle, **Run Now** (test immediately), **Delete**
- **Run History** shows each execution's time/status/result
- **Open Run Chat** replays the AI's full reasoning + tool calls for that run
  (great for debugging)

Supports both periodic ("every N minutes") and one-shot execution, wrapping any
sequence of robot commands.

*Ref: `08.OpenClaw Development / 9. OpenClaw timed tasks`.*

---

## 5. Teach it your own routine (custom Skills)

Autonomy is fully extensible — a Skill is just a Markdown file.

1. Create a folder (its name is the skill name; use `lowercase-with-hyphens`):
   ```bash
   mkdir -p $HOME/.openclaw/workspace/skills/my-task
   nano $HOME/.openclaw/workspace/skills/my-task/SKILL.md
   ```
2. In `SKILL.md`, define: **trigger keywords**, **tool dependencies**, a
   **coordinate/scene description**, the **step-by-step workflow**, and
   **exception handling**. (Copy the structure of an existing skill in the same
   directory as a template.)
3. Apply it:
   ```bash
   openclaw gateway restart
   ```
4. **Enable it** in WebChat → **Agent → Skills**.

*Ref: `08.OpenClaw Development / 8. OpenClaw skills development` (section 5).*

---

## 6. Tuning & honest limitations

| Topic | What to know |
|---|---|
| **No torque feedback** | Gripper closure is set by parameter, not force sensing. If it grabs too loose/tight or misses, tune `grasp_offset` / `place_offset`. |
| **Grasp accuracy** | Depends on arm calibration **and** the vision model actually detecting the object. Check the debug images (Section 2.3). |
| **Object suitability** | Demos use ~4×4×4 cm blocks; good lighting helps the vision model. Very small, shiny, or cluttered objects are harder. |
| **Arm workspace** | The chassis auto-adjusts to bring targets in range, but objects far outside the arm's reach can't be grasped in one move. |
| **Vision model required** | Grasping/observation needs Dify + a vision-capable model configured; a chat-only key won't detect objects. |
| **Init/hold poses** | Arm initial and grasp-hold poses are configurable in `m3pro_bringup/config/common.yaml` (`InitArmPose`, `GraspKeepPose`). |

*More: `09.OpenClaw Embodied intelligent application / 2. arm tracking and grasping`
(parameter debugging + FAQ), and each folder's "Common Errors and Solutions" PDF.*

---

## 7. Quick reference — the control stack

```bash
# Always-needed base (Sections 2–5)
sh start_agent.sh                                   # chassis agent
ros2 launch m3pro_bringup car_base.launch.py        # sensors + arm assist + camera
ros2 launch m3pro_bringup mcp_service.launch.py     # MCP tools (OpenClaw's body)
ros2 run multi_brains_pre openclaw_bridge           # optional: voice replies

# For fetch-and-carry, also start navigation (needs a saved map):
ros2 launch M3Pro_navigation base_bringup.launch.py
ros2 launch M3Pro_navigation navigation2.launch.py
ros2 launch M3Pro_navigation nav_rviz.launch.py     # then set 2D Pose Estimate in RViz

# Dify (object detection / vision):
sh ~/bringup_dify.sh                                # browse to http://<robot-ip>
```

Interact at `http://<robot-ip>:18789` (token `yahboom`) or by voice ("Hi, yahboom").

---

## Where to go deeper (repo folders)

- `08.OpenClaw Development` — MCP interface, arm/chassis control, **skills development**, timed tasks, multi-agent, vector memory
- `09.OpenClaw Embodied intelligent application` — target tracking, **grasping**, fixed-point placement, garbage sorting, **SLAM + navigation**, line patrol, distance measurement
- `10.OpenClaw scenario sandbox` — tabletop map building, visual relocation markers, comprehensive case

*Hardware/tutorial questions: Yahboom support — support@yahboom.com.*
