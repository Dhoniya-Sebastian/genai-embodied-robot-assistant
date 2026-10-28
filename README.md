# Generative-AI Embodied Robot Assistant

A simulated service robot that understands spoken requests and carries them out safely. An **e-puck robot in Webots** listens to an instruction such as *"Find the red ball, and if it isn't in the lab, check Office A"*. A **large language model** turns the request into a symbolic plan, a **deterministic validator** checks the plan, and the robot executes it with its **GPS, compass, proximity sensors and camera**.

The core task is search: find a red ball hidden in one of four rooms, ignore look-alike distractors (a red box, a green ball, coloured cylinders), handle obstacles, and report back.

---

## How it works

```text
speech / text goal
      │
      ▼
SpeechIO ─► DialogueManager ─► LLMInterface ─► PlanValidator ─► TaskExecutive ─► RobotSkills ─► Webots devices
 (worker thread)  (clarify)     (symbolic plan)  (deterministic    (one small step   (navigate,      (motors, GPS,
                                                  safety gate)      per timestep)     search, ...)    compass, camera,
                                                                         │                             proximity)
                                                                         ▼
                                                                    WorldState  ◄── read for explanations & replanning
```

Design principles:

- **The language model proposes; it never acts.** It may output only symbolic actions from a fixed vocabulary. It never writes wheel speeds or code. Every plan, including revised plans after a failure, must pass the validator.
- **The simulation loop is never blocked.** Speech recognition and model inference run in a worker thread. Only the main loop touches robot devices.
- **The robot perceives; it doesn't cheat.** The ball is found with the camera alone. The scene Supervisor that places the ball never shares its position with the robot controller.
- **Execution is bounded.** Skills advance one small step per timestep, with timeouts, retry limits and bounded replanning.

### Modules

All in `controllers/project3_student/`.

| Module | File | Responsibility |
|---|---|---|
| `SpeechIO` | `speech_io.py` | Non-blocking speech-to-text and text-to-speech |
| `DialogueManager` | `nlu_dialogue.py` | Ambiguity detection, clarification questions, status queries |
| `LLMInterface` | `llm_interface.py` | Prompting the model for a symbolic plan; replanning after failures |
| `PlanValidator` | `plan_validator.py` | Deterministic schema and safety checks |
| `WorldState` | `world_state.py` | Goal, current step, search results, blocked routes, counters |
| `TaskExecutive` | `task_executive.py` | Non-blocking plan execution, conditional branches |
| `Navigator` | `navigation.py` | Waypoint navigation using GPS and compass, with obstacle safety |
| `ObjectPerception` | `perception.py` | Camera-based object detection and bounded search |
| `RobotSkills` | `robot_skills.py` | `start / update / cancel` for each symbolic action |
| — | `plan_schema.py` | Shared plan vocabulary and limits |
| — | `semantic_memory.py` | Remembers where objects were seen |
| — | `project3_student.py` | Main controller loop |

---

## Symbolic plan format (v1.0)

```json
{
  "version": "1.0",
  "goal": "Find the red ball in the laboratory and report.",
  "steps": [
    {"action": "navigate", "location": "laboratory"},
    {"action": "search", "object": "ball", "colour": "red"},
    {"action": "report", "message": "Search complete."}
  ]
}
```

| Item | Allowed values |
|---|---|
| Actions | `navigate` (location), `search` (object, optional colour), `observe`, `ask` (question), `report` (message), `return_home`, `stop`, plus `conditional` (`condition`, `then`, `else`) |
| Locations | `reception`, `main_corridor`, `office_a`, `office_b`, `laboratory`, `meeting_room` |
| Objects / colours | `ball`, `box`, `cylinder` / `red`, `green`, `blue`, `yellow` |
| Condition sources / operators | `last_search_result`, `route_status` / `found`, `equals` |
| Limits | 20 steps in total, nesting depth 2, 240 characters per text field |

---

## The environment

`worlds/project_genai_embodied_assistant.wbt` (Webots R2025a): a 10 m × 8 m office floor with x/y on the floor and z up.

- **Robot:** e-puck with GPS and Compass, a 320×240 camera and 8 infrared proximity sensors (`ps0`–`ps7`). It starts at reception.
- **Named locations,** each marked by a coloured floor pad:

  | Location | (x, y) |
  |---|---|
  | reception | (-0.8, -2.8) |
  | main_corridor | (0.0, 2.4) |
  | office_a | (-3.5, 2.0) |
  | office_b | (3.5, 2.0) |
  | laboratory | (-3.5, -2.0) |
  | meeting_room | (3.5, -2.0) |

- **Target:** a red ball (radius 0.09 m). On each restart, the scene Supervisor (`controllers/scene_randomizer`) places it at one of 8 spawn points, two per room. The choice is printed to the console as ground truth for evaluating runs. Set `PROJECT_RANDOM_SEED` to make a placement reproducible.
- **Distractors:** green ball, red box, blue box, green box, yellow cylinder, blue cylinder.
- **Furniture:** desks in both offices, a lab bench and a meeting table.
- **Obstacle:** a black block in the central hall for testing obstacle handling and replanning.

---

## Project layout

```
.
├── controllers/
│   ├── project3_student/        robot controller and modules
│   └── scene_randomizer/        Supervisor that places the red ball
├── config/
│   ├── locations.json           named destinations
│   ├── navigation_graph.json    waypoint graph between locations
│   ├── objects.json             supported object types and colours
│   ├── spawn_points.json        ball spawn points (Supervisor only)
│   └── llm_config.example.json  template for the language-model backend
├── prompts/
│   └── planning_prompt_template.txt
├── tests/
│   └── test_plan_validator.py
├── worlds/
│   ├── project_genai_embodied_assistant.wbt
│   └── carpet_texture.png
└── requirements.txt
```

---

## Getting started

1. Install **Webots R2025a** and a compatible Python 3.
2. Optionally create a virtual environment and point Webots at it: **Preferences → General → Python command**, e.g. `/Users/<you>/webots-venv/bin/python3`.
3. Install the packages your implementation uses and pin them in `requirements.txt`. Webots supplies the `controller` module, so don't install the unrelated PyPI package of that name.
4. To use a language model, copy `config/llm_config.example.json` to `config/llm_config.json` and fill it in. A local GGUF model (e.g. via `llama-cpp-python`) or an online API both work. Credentials come from environment variables, and the real config file and model weights stay out of version control.
5. Open `worlds/project_genai_embodied_assistant.wbt` in Webots and press Play.
6. Run the controller-independent tests:
   ```bash
   python -m unittest discover -s tests -v
   ```

---

## Tech stack

Webots R2025a · Python 3 · OpenCV / NumPy (vision) · a local or hosted LLM (planning) · Vosk / pyttsx3 or similar (speech)
