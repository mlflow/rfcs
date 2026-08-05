---

## start_date: 2026-04-23

mlflow_issue: [https://github.com/mlflow/mlflow/issues/21445](https://github.com/mlflow/mlflow/issues/21445)
rfc_pr:

# Scorer Presets for Common Evaluation Patterns


| Author(s)              | Nehanth     |
| ---------------------- | ----------- |
| **Date Last Modified** | 2026-08-03  |
| **AI Assistant(s)**    | Claude Code |


# Summary

> **Note:** This RFC is based on [mlflow/mlflow#21445](https://github.com/mlflow/mlflow/issues/21445). The motivation, proposed presets, and API examples are derived from that issue, with additional design details and implementation specifics added here.

MLflow provides 21 built-in scorers for evaluating GenAI outputs, but users have no way to select a coherent subset for a specific evaluation pattern. Today, evaluating an agent requires importing and instantiating 9+ individual scorer classes -- boilerplate that gets copy-pasted across teams and templates.

This RFC proposes a `Preset` class that packages a named collection of scorers with support for **customization** and **persistence**. MLflow ships three built-in preset subclasses (`Rag`, `Agent`, `ConversationalAgent`) as starting points. Users can create custom presets, persist them to the MLflow server, and share them across teams and sessions. Presets can be passed directly in the `scorers` list alongside individual scorers.

# Basic Example

```python
import mlflow
from mlflow.genai.scorers import Agent

# Use a built-in preset directly
result = mlflow.genai.evaluate(
    data=eval_dataset,
    predict_fn=predict_fn,
    scorers=[Agent()],
)
```

```python
# Mix presets and individual scorers
from mlflow.genai.scorers import Agent, Guidelines

result = mlflow.genai.evaluate(
    data=eval_dataset,
    predict_fn=predict_fn,
    scorers=[Agent(), Guidelines(name="tone", guidelines=["Respond professionally"])],
)
```

```python
# Define a custom preset and persist it for team sharing
from mlflow.genai.scorers import Preset, Safety, Fluency

my_preset = Preset("my_team_eval", scorers=[Safety(), Fluency(), my_custom_scorer])

# Register to MLflow server so the team can reuse it
my_preset.register()

# Later, another team member loads it
from mlflow.genai.scorers import get_scorer_preset

preset = get_scorer_preset(name="my_team_eval")
result = mlflow.genai.evaluate(data=eval_dataset, scorers=preset.scorers)
```

## Motivation

### The Problem

As described in [the original issue](https://github.com/mlflow/mlflow/issues/21445), the Databricks agent app template [evaluate_agent.py](https://github.com/databricks/app-templates/blob/main/agent-openai-agents-sdk/agent_server/evaluate_agent.py) imports and instantiates 9 separate scorers to evaluate a conversational agent:

```python
from mlflow.genai.scorers import (
    Completeness,
    ConversationalSafety,
    ConversationCompleteness,
    Fluency,
    KnowledgeRetention,
    RelevanceToQuery,
    Safety,
    ToolCallCorrectness,
    UserFrustration,
)

mlflow.genai.evaluate(
    data=simulator,
    predict_fn=predict_fn,
    scorers=[
        Completeness(),
        ConversationCompleteness(),
        ConversationalSafety(),
        KnowledgeRetention(),
        UserFrustration(),
        Fluency(),
        RelevanceToQuery(),
        Safety(),
        ToolCallCorrectness(),
    ],
)
```

Every team building agent evaluation follows this same pattern. This creates three problems (from the [original issue](https://github.com/mlflow/mlflow/issues/21445)):

1. **No built-in grouping.** `get_all_scorers()` returns all 19 default-constructible scorers. Users evaluating a RAG pipeline get `ToolCallCorrectness`; users evaluating an agent get `RetrievalGroundedness`. Each unnecessary scorer wastes an LLM API call.
2. **21 scorers to choose from.** Users must read documentation for each scorer to determine relevance. Session-level scorers (e.g., `KnowledgeRetention`) silently produce no results when passed to single-turn evaluation.
3. **Copy-paste problem.** The same scorer lists get duplicated across templates, notebooks, and tutorials. When new scorers are added, existing lists don't pick them up.
4. **No persistence or sharing.** Teams cannot save and share a curated set of scorers. Each team member independently assembles their own list, leading to drift across projects.

### Who Benefits

- **New users** get a curated starting point without reading all 21 scorer docs
- **Teams** can define, persist, and share custom presets across sessions and team members
- **Template authors** replace hardcoded scorer lists with a single preset
- **MLflow maintainers** gain a single place to update when new scorers are added

### Out of Scope

- **Third-party scorer presets.** Integrating presets for DeepEval, RAGAS, or TruLens scorers.

## Detailed Design

### The `Preset` Class

A `Preset` is a named, iterable container of scorers. It is **not** a `Scorer` subclass -- it is a grouping mechanism that gets flattened into individual scorers at validation time.

```python
class Preset:
    def __init__(self, name: str, scorers: list[Scorer]): ...
    def register(self, *, experiment_id: str | None = None): ...
    def copy(self, *, to_experiment_id: str): ...
    @property
    def name(self) -> str: ...
    @property
    def scorers(self) -> list[Scorer]: ...
    def __iter__(self): ...
    def __len__(self): ...
    def __repr__(self): ...
```

**Key design decisions:**

- **Immutable.** Scorers are stored internally and the `scorers` property returns a fresh copy each time, so mutations don't affect the preset.
- **Blocks duplicates on construction.** `__init__` raises an error if duplicate scorers (same type and name) are passed. Users who need two scorers of the same type should give them different names.
- **Not a `Scorer` subclass.** A preset doesn't produce feedback -- it's a container. The evaluation loop assumes one scorer = one result column.
- **Stores instances, not classes.** Users pass already-configured scorer instances.

### Built-in Presets as Subclasses

Each built-in preset is a subclass of `Preset` that hardcodes its scorer list. Each call creates **fresh scorer instances** (no shared mutable singletons) and supports preset-specific customization. See the Built-in Preset Summary table below for the scorers in each preset.

**Why subclasses over instances:**

- **Fresh instances every time.** `Agent()` creates new scorer instances on each call. No shared mutable state.
- **Preset-specific customization.** Each preset can accept its own parameters (e.g., `Agent(model="openai:/gpt-4o")` to set the judge model for all scorers).
- **Type checking.** `isinstance(preset, Agent)` works — code can distinguish which preset is being used.
- **Custom control flow.** Each preset can override methods for preset-specific validation or behavior.

### Customization

Users can create custom presets with any combination of scorers:

```python
my_preset = Preset("my_eval", scorers=[
    ToolCallCorrectness(),
    Safety(),
    Fluency(),
    Guidelines(name="tone", guidelines=["Be professional"]),
    my_custom_scorer,
])
```

### Persistence

Presets can be registered to the MLflow server so teams can share them across sessions. This leverages the existing scorer registration infrastructure.

**Register a preset:**

```python
my_preset = Preset("my_team_agent", scorers=[
    ToolCallCorrectness(),
    Safety(),
    Fluency(),
])

# Register to the active experiment
my_preset.register()

# Or register to a specific experiment
my_preset.register(experiment_id="123")
```

**Preset CRUD API:**

```python
from mlflow.genai.scorers import get_scorer_preset, list_scorer_presets, delete_scorer_preset

# Load a preset
preset = get_scorer_preset(name="my_team_agent")
preset = get_scorer_preset(name="my_team_agent", version="a8f3c2")

# List all presets in the current experiment
presets = list_scorer_presets()

# Update a preset (registers a new version)
updated = Preset("my_team_agent", scorers=[Safety(), Fluency(), Completeness()])
updated.register()

# Delete a preset
delete_scorer_preset(name="my_team_agent")

# Copy to another experiment
preset.copy(to_experiment_id="456")

# Use in evaluation
result = mlflow.genai.evaluate(data=eval_dataset, scorers=preset.scorers)
```

**Why persistence matters:**

- **Version stability.** Persisted presets are snapshots — they don't change when MLflow upgrades. Built-in presets serve as starting points; teams persist their own versions for stability.
- **Team sharing.** A persisted preset is available to any team member with access to the experiment.
- **Customization without code.** Teams can customize and persist presets without modifying source code or templates.

**Persistence behavior:**

- **Scope.** Presets are scoped to experiments, consistent with how scorer registration already works in MLflow. This prevents name collisions across teams and ensures presets are organized alongside the experiments they evaluate. If no `experiment_id` is provided, the active experiment is used.
- **Custom scorer portability.** If a preset contains custom scorers, those scorers must be registered first. When a teammate loads the preset, the custom scorers are resolved from the registry. If a custom scorer is not registered, `preset.register()` will raise an error.
- **Discovery.** `list_scorer_presets()` returns all registered presets for the current experiment, allowing teams to discover what presets are available. This follows the same pattern as `list_scorers()`.
- **Copying across experiments.** Since experiments typically map to a single agent, users should be able to copy a preset to another experiment via `preset.copy(to_experiment_id)`.
- **Versioning.** A preset version is an immutable snapshot of `(scorer_id, scorer_version)` pairs, identified by a hash generated from that set. A new preset version is created in two ways:
  - **Manual** — the user re-registers the preset with a different scorer list (adding, removing, or swapping scorers).
  - **Automatic** — when a member scorer gets a new version, MLflow detects that the scorer belongs to one or more presets and auto-creates a new version of each, snapshotting the updated scorer version. Users never maintain preset versions by hand.

  Identical scorer combinations produce identical hashes, so rolling a scorer back to a previous combination creates no duplicate version. Version history is ordered by creation time via the `/versions` endpoint.

  Users load the latest version by default, or pin a hash via `get_scorer_preset(name, version="a8f3c2")` for deterministic, reproducible evaluation.
- **Visibility of auto-created versions.** When registering a new scorer version triggers preset version bumps, the registration response and logs surface which presets were updated, so the side effect is never silent.
- **Telemetry.** Using MLflow's existing telemetry system, preset load calls record whether a specific version was explicitly requested or defaulted to latest. This validates how often users rely on deterministic pinning versus latest.
- **Return type.** `get_scorer_preset()` returns a `Preset` object with server-side metadata attached (experiment ID, version, created timestamp). The preset is fully functional — it can be iterated and passed to `evaluate()` like any other preset.

**REST API Endpoints:**

Following the existing scorer registration API pattern:

| Method | Endpoint | Request | Response |
|--------|----------|---------|----------|
| `POST` | `/mlflow/scorer-presets/register` | experiment_id, name, scorer_ids | version, preset_id |
| `GET` | `/mlflow/scorer-presets/get` | experiment_id, name, version? | preset |
| `GET` | `/mlflow/scorer-presets/list` | experiment_id | presets[] |
| `GET` | `/mlflow/scorer-presets/versions` | experiment_id, name | presets[] |
| `DELETE` | `/mlflow/scorer-presets/delete` | experiment_id, name, version? | — |
| `POST` | `/mlflow/scorer-presets/copy` | experiment_id, name, to_experiment_id, version? | — |

List operations (`list`, `versions`) support pagination via `page_token` and `max_results` parameters. Deleting without a version removes all versions of the preset.

**Serialization schema:**

Presets store references to registered scorers. When `preset.register()` is called, the Python API automatically registers each scorer. If a scorer is already registered in the experiment, it uses the existing scorer ID instead of creating a duplicate.

Each preset version snapshots the specific scorer versions it was created with, making versions immutable and reproducible:

```json
{
  "name": "my_team_agent",
  "version": "a8f3c2",
  "scorers": [
    {"scorer_id": "scorer_abc123", "scorer_version": 3},
    {"scorer_id": "scorer_def456", "scorer_version": 1},
    {"scorer_id": "scorer_ghi789", "scorer_version": 1}
  ]
}
```

### Built-in Preset Summary

MLflow ships three built-in preset subclasses as starting points. Each call creates fresh scorer instances. Users can customize and persist their own presets.

| Preset                 | Scorers                                                                                                                                 | Use Case                                                 |
| ---------------------- | --------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------- |
| `Rag()`                | RetrievalRelevance, RetrievalGroundedness, RelevanceToQuery, Safety, Completeness                                                       | Retrieval-augmented generation pipelines                 |
| `Agent()`              | ToolCallCorrectness, ToolCallEfficiency, RelevanceToQuery, Safety, Completeness                                                         | Single-turn tool-calling agents                          |
| `ConversationalAgent()`| All of `Agent` + UserFrustration, ConversationCompleteness, ConversationalSafety, ConversationalToolCallEfficiency, KnowledgeRetention  | Multi-turn conversational agents                         |


#### Design Rationale

- **Only three built-in presets** (Rag, Agent, ConversationalAgent) — these represent clear, distinct evaluation patterns. Users can create and persist their own groupings for specific needs.

### UI Surface

Presets are managed within the existing **Judges tab** in the experiment page.

**"New preset" button:**

A new "New preset" button is added next to the existing "New LLM judge" split button in the top-right header.

**New preset view:**

Clicking "New preset" opens a view with two paths:

- **Quickstart** — three built-in preset cards (Rag, Agent, ConversationalAgent). Each card shows the preset name, use case description, and the scorers it includes. Clicking a quickstart card registers it to the experiment immediately with the default scorers and returns to the Judges tab.

- **Create custom preset** — opens a creation form (following the same two-panel layout as the "Create LLM judge" modal):
  - **Left panel:**
    - Name field (required, cannot be changed after creation)
    - Searchable scorer list with checkboxes, grouped by type (built-in LLM judges, session-level judges, registered custom judges). Each row shows the scorer name and a short description.
  - **Right panel:**
    - Live preview of selected scorers, updating as the user checks/unchecks. Shows a count of selected scorers at the top.
  - **Footer:**
    - "Cancel" and "Create preset" buttons.

**Preset cards in the Judges tab:**

Registered presets (quickstart or custom) appear as cards in the Judges tab alongside individual judge cards. Each preset card shows:

- Preset name, scorer count, and version badge
- Collapsed view showing scorer names as tags
- Expandable view showing each scorer with its type and description
- Overflow menu with edit and delete actions

**Edit preset:**

Clicking edit opens the same creation form, pre-populated with the current scorers. The name field is read-only. Saving creates a new version.

**Delete preset:**

Clicking delete opens a confirmation dialog. Deleting removes all versions of the preset.

## Drawbacks

1. **New class in the API.** Adds `Preset` to the public surface. Mitigation: it's a simple container with persistence support.
2. **Opinionated defaults.** Not everyone will agree on which scorers belong in which preset. Mitigation: users can create and persist their own custom presets.
3. **Persistence adds scope.** Supporting preset registration and retrieval increases implementation complexity. Mitigation: leverages the existing scorer registration infrastructure.

# Alternatives

### 1. `get_scorer_preset()` function (no class)

Instead of a `Preset` class, provide a simple function that returns a plain list. Simpler to implement and can also support persistence via `register_preset()` / `get_scorer_preset()`.

### 2. Tag-based filtering

Add `categories` to each scorer class and provide `get_scorers(categories=["rag"])`. More flexible but over-engineered for 21 scorers and requires modifying every existing class.

### 3. Enum-based API

`ScorerPreset.RAG.get_scorers()`. Type-safe but heavier API surface.

### 4. Do nothing

Users keep copy-pasting scorer lists. Does not scale as the scorer count grows.

# Adoption Strategy

This is an **additive, non-breaking change**. Existing code continues to work unchanged.

- Update documentation and templates to show `Preset` usage alongside the manual import pattern.
- Update the `validate_scorers()` error message to mention presets for discoverability.
- Databricks agent templates can simplify from 9 imports + 9 instantiations to `scorers=[ConversationalAgent()]`.
- Teams can persist their customized presets and share them across projects.

# Open Questions

1. **Class-based vs function-based approach.** The class-based approach is proposed as the primary design for its ergonomics and customization support. The function-based approach is a viable alternative that may be more flexible for persistence. Both approaches were discussed during review.
