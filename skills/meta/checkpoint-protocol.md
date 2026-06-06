# Checkpoint Protocol — Meta Skill

## When to Use

After completing a stage's work AND passing review. This skill teaches you when and how to checkpoint. It replaces the Python `checkpoint_policy.py` with an instruction-driven protocol.

Checkpoints are the save points of a pipeline. They enable resume-from-failure and audit trails.

## Protocol

### Step 1: Check Manifest Policy

Read the current stage's configuration from the pipeline manifest:

```yaml
- name: idea
  checkpoint_required: true      # Must we checkpoint?
  human_approval_default: true   # Must we ask the human?
```

| `checkpoint_required` | Action |
|----------------------|--------|
| true | Checkpoint + proceed automatically |
| false | Skip checkpoint entirely (rare) |

### Step 2: Prepare Checkpoint Data

Gather everything needed for the checkpoint:

1. **Stage name** — which stage just completed
2. **Status** — `"completed"`
3. **Artifacts** — the canonical artifact(s) produced by this stage
4. **Metadata** — review findings, cost snapshot, timing info

### Step 3: Write Checkpoint

Call the checkpoint utility:

```python
write_checkpoint(
    pipeline_dir,      # Project working directory
    project_name,      # Project identifier
    stage_name,        # e.g., "idea"
    status,            # "completed" or "awaiting_human"
    artifacts,         # {"brief": {...}} — the stage's output
)
```

The checkpoint utility will:
- Validate the artifact against its schema
- Write the checkpoint JSON to disk
- Include timestamp and stage metadata

### Step 4: Determine Next Stage

After checkpoint is written and approved (if needed):

```python
next_stage = get_next_stage(pipeline_dir, project_name)
```

This reads all existing checkpoints and returns the next stage that needs to run, or `None` if the pipeline is complete.

### Step 5: Resume Protocol

At the START of any pipeline run (not just after a stage), always check for existing progress:

```python
next_stage = get_next_stage(pipeline_dir, project_name)
```

If `next_stage` is not the first stage:
1. Load prior artifacts from checkpoints for context
2. Continue from that stage

### Sample Checkpoint (Reference-Driven Productions)

When a production is reference-driven (VideoAnalysisBrief exists), render a sample clip (10-15 seconds) and checkpoint it before proceeding to full production:

| Stage | checkpoint_required | Notes |
|-------|--------------------|-----------------------|
| `sample` | true | Auto-proceed after rendering |

The sample checkpoint is NOT a pipeline stage — it's a sub-checkpoint within the proposal stage. It produces a rendered preview clip stored at `projects/<name>/assets/sample/sample_v{N}.mp4`.

Log the sample result:
```
Sample clip: [path to sample_v1.mp4]
- Duration: [X] seconds
- Voice: [TTS provider + voice name]
- Visuals: [description]
- Music: [source]
- Sample cost: $[X.XX] / Projected full cost: $[X.XX]
```

## Key Principles

1. **Always checkpoint completed work.** Even if `checkpoint_required: false`, consider checkpointing anyway if the stage took significant time or cost. Losing work is worse than an extra file on disk.

2. **Include cost snapshots.** Log how much has been spent and how much remains at every checkpoint.

3. **Checkpoints enable resume.** If the pipeline crashes at `compose`, the next run picks up from `compose` — not from `idea`. This is the whole point.

4. **Be transparent in checkpoint data.** Always include the artifact, review findings, cost, and any substitutions or degraded paths chosen.
