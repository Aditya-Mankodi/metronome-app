# Metronome AI Weekly Planner

## Overview

This codebase implements a desktop planning assistant that converts user-provided study and task constraints into a structured 7-day timetable. The application is packaged as a PyInstaller Windows executable and runs a PyQt5-based GUI that collects date ranges, daily activity durations, exercise blocks, and task deadlines, then submits a constraint-heavy prompt to the OpenAI GPT-3.5 Turbo chat completion API. The returned response is treated as a machine-readable JSON schedule and is rendered in a seven-day dashboard for the user.

At a systems level, the program is a hybrid of a UI orchestration layer and a prompt-driven planning engine. The real execution path is:

1. `Ui_MainWindow.setupUi()` creates the full desktop form containing start/end dates, start/end times, activity fields, exercise blocks, and task rows.
2. `Ui_MainWindow.generate()` normalizes raw text inputs into list-based data structures representing activities and tasks.
3. A large natural-language scheduling prompt is assembled from those lists, embedding time windows, task deadlines, and minimum required time constraints.
4. The prompt is sent to `openai.ChatCompletion.create(model="gpt-3.5-turbo", messages=[...])`.
5. The returned model output is sanitized with `json.loads(str(msg_))`, converted into a date-to-schedule mapping, and displayed across the UI’s day widgets.
6. Success and failure states are handled by toggling the `generated` label between `Generated!` and `Error!`.

This matters because it converts low-structure human planning input into a constrained, machine-readable weekly schedule while hiding the LLM interaction behind a desktop UI.

## Technical Architecture Summary

The code is fundamentally a Python application built around a single `Ui_MainWindow` class. The class owns the entire form, the generation function, and the UI state transitions. The runtime is not a web service or backend API; it is a local desktop application that issues outbound HTTPS calls to OpenAI and renders the result locally with Qt widgets.

The main architectural pattern is a deterministic orchestration pipeline:

- Input acquisition from Qt widgets
- Structured normalization into fixed-length activity/task vectors
- Assembly of a large prompt with explicit scheduling constraints
- Remote inference via the OpenAI API
- JSON extraction and post-processing into per-day text blocks
- GUI rendering to label widgets and a generated-result panel

## Tech Stack

### Languages

- Python 3.10
- Windows PE executable packaging (PyInstaller-generated binary)
- C++/Win32 runtime dependencies bundled with the application

### Frameworks and UI Runtime

- PyQt5
- QtCore
- QtGui
- QtWidgets
- PyInstaller 2.1+ packaging model
- Qt platform plugins and Win32 support libraries

### Libraries and SDKs

- `openai` Python SDK
- `json` for strict JSON parsing
- `numpy` (bundled runtime dependency)
- `certifi` for TLS trust material
- `requests` and supporting HTTP stack
- `aiohttp`, `asyncio`, `urllib3`, `idna`, `charset_normalizer`
- `attrs` and `dataclasses`
- `multidict`, `frozenlist`, `yarl`, `async_timeout`
- `PyQt5` Qt resource and widget modules

### Interface and Execution Environment

- Windows desktop window management via Qt5 platform plugins
- GUI form controls: `QWidget`, `QLabel`, `QFrame`, `QGridLayout`, `QLineEdit`
- Outbound HTTPS API access to OpenAI
- Local filesystem access for application resources such as `metronome.png`
- No direct hardware sensor, serial, or GPIO interfaces are present in the code path; the system is UI-driven and network-mediated

## Detailed Engineering Challenges and Solutions

### 1. Converting loosely structured GUI inputs into a strict planning schema

The most significant challenge in the app is the mismatch between free-form user input and the rigid schedule schema required by the LLM. The form exposes a variable number of user-filled values, but the scheduling engine standardizes them into fixed internal lists:

- `activities_` is assembled from three pairs: `Activity1 + ta1`, `Activity2 + ta2`, `Activity3 + ta3`
- `tasks_` is assembled from four tuples: `Task1 + tt1 + d1`, `Task2 + tt2 + d2`, etc.
- Each list entry is normalized by reading `.text()`, then filtering blank values with comparisons like `!= ''` before appending to the operational sequence.

This design prevents empty or malformed rows from polluting the model prompt, and it turns a weakly typed GUI into a deterministic data model. The code uses iterative list building and append-based normalization instead of a dynamic object model, which is stable and suitable for a compact desktop utility.

### 2. Prompt synthesis under severe scheduling constraints

The prompt-building block is a constraint-heavy planning specification. The `generate()` routine creates a single `command` string that explicitly instructs the model to:

- respect a weekly date range,
- fit activities within the provided daily schedule,
- respect task deadlines,
- allocate required minutes before the deadline,
- avoid stacking work only on the due date,
- keep daily activities on every day,
- fit within the `Start` and `End` time windows,
- return output as JSON only.

The strings are concatenated with explicit formatting markers such as:

- `"Pls create a timetable..."`
- `"Daily Activities:\n..."`
- `"Tasks:\n..."`
- `"Please provide output as JSON ... No need of any other text other than JSON output"`

This is a classic prompt-engineering solution: the app does not attempt to parse natural language in the model output; instead, it embeds a machine-readable output requirement and a dense set of temporal constraints directly into the prompt.

### 3. Recovering a strict JSON schedule from a generative model

This project explicitly solves the problem of non-deterministic LLM output by insisting on JSON and then parsing it aggressively in code. The function executes:

- `msg = openai.ChatCompletion.create(...)`
- `msg_ = msg.choices[0].message.get("content")`
- `events_ = json.loads(str(msg_))`
- `events = list(events_.items())`

This is a strong engineering choice: the model response is immediately coerced into a Python dictionary and flattened to a list of `(date, schedule)` pairs. That allows the UI to render the schedule in a fixed seven-day layout without ad hoc string parsing.

The code then iterates over the list and serializes each date + schedule pair into a display string:

- `str(i[0]) + "\n" + str(i[1])`

This converts a loosely structured LLM response into a stable, renderable data model while preserving the date keys to display each day’s planner block.

### 4. Guarding the system against network and LLM runtime failure

The `generate()` function is wrapped in a `try/except` block. If anything fails in the OpenAI request, JSON extraction, or list rendering, execution falls into the exception handler, which sets the UI status label to `Error!` and makes it visible. This is critical because a language model call can fail for reasons including quota limits, authentication issues, malformed responses, or transient network errors.

The result is a robust user-visible failure mode instead of a silent crash: the desktop app remains operational and clearly indicates that the generation step failed.

### 5. Data cleaning and fixed-width schedule rendering

The app uses deterministic list transformations to avoid invalid schedule text:

- blank activity rows are filtered out with `if activitiy_[i][0] != ''`
- strings are normalized via `.upper()` for activity labels
- each task/assignment is formatted as `- TaskName, Deadline: dd/mm/yy (Required Time: X min)`
- empty rows are appended as placeholder blanks so the final rendering retains the expected UI structure

This is effectively a data-cleaning layer before output to the user. It transforms free-text inputs into a compact, renderable, and safe representation before the schedule is transmitted to the LLM and then back to the UI.

### 6. GUI construction as a static declarative layout engine

The `setupUi()` routine programmatically builds the complete interface instead of relying on a design file. It instantiates widgets in sequence:

- `QWidget` as the central container
- `QLabel` for the title and UI prompts
- `QFrame` objects such as `StartTimes`
- `QGridLayout` for field placement
- `QLineEdit` and other form elements for each time/date/task field

This is a direct implementation of declarative UI assembly in code. It ensures that the front-end is self-contained and not subject to external designer files or dynamic layout serialization.

## Key Programmatic Features

### 1. Constraint-driven prompt synthesis

The app does not simply ask the model to “generate a timetable.” Instead, it generates a highly constrained prompt that includes:

- time windows,
- activity durations,
- task deadlines,
- required workload before deadline,
- rules against deadline-only work,
- daily activity requirements,
- strict JSON-only output requirements.

This is the most important algorithmic part of the project: prompt conditioning is what turns a loosely guided generative model into a usable schedule planner.

### 2. Fixed-dimension planning vector normalization

The `generate()` function normalizes input into:

- 3 daily activity slots
- 4 task slots
- 7-day display slots

These fixed dimensions are enforced by loops over `range(0, 3)` and `range(0, 4)`, then displayed to the UI through `day1` to `day7`. This is a lightweight but important scheduler model: it prevents unbounded growth and keeps the output within a consistent weekly container.

### 3. State-machine-like execution flow

The generation phase behaves like a finite-state pipeline:

- parse user inputs,
- normalize and validate strings,
- assemble the schedule prompt,
- call the model,
- parse JSON,
- render schedule to UI,
- transition to success/failure marker state.

This state transition is explicitly visible in the code: the `generated` label is updated to `Generated!` or `Error!` and made visible based on the execution path.

### 4. Seven-day rendering logic

Once the JSON response is parsed into `events_`, the code converts it to `events = list(events_.items())` and then iterates to populate the labels for the weekly planning view. The daily schedule is not stored as a complex object graph; instead, it is flattened into a compact list of display strings and rendered to fixed widgets. This minimizes memory overhead and preserves predictable UI behavior.

### 5. Desktop orchestration with a single window lifecycle

The top-level module creates a `QApplication`, instantiates `QMainWindow`, loads the UI, and calls `sys.exit(app.exec_())`. This is the execution lifecycle of the product: GUI startup, UI layout initialization, and event loop management. The app is cleanly structured around a single main window and a single generation entry point.

## Application Behavior Summary

The system is best understood as a desktop AI scheduling assistant that:

- accepts study and task data from a form,
- converts that input into a highly constrained planning prompt,
- relies on GPT-3.5 Turbo to generate a weekly schedule,
- enforces JSON output and parses it aggressively,
- renders the resulting per-day schedule into a seven-day planner UI.

This is a focused but technically meaningful integration of PyQt5, prompt engineering, and LLM-based scheduling logic in a compact desktop package.

## Security and Risk Notes

- The code embeds a hardcoded API key at module initialization: `API = 'sk-...'`.
- This is a production concern because the key is stored in the distributed executable and is not rotated or externalized through a secure secret manager.
- The application performs network calls to the OpenAI API and therefore depends on valid credentials, network access, and successful model responses.
- There is no direct secret injection or secure credential workflow in the code path; all configuration is static and compiled into the application artifact.

## Conclusion

Metronome is not a generic UI or sample app; it is a compact AI scheduling service embedded into a desktop shell. Its engineering value lies in the precise handling of heterogeneous planning inputs, the prompt-constrained generation of a specific schedule schema, the conversion of model output into strict JSON, and the transformation of machine-generated schedule data into a readable daily planner. The code demonstrates a practical pattern for turning a language model into a structured scheduling assistant behind a local GUI.
