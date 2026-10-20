---
name: review-chain
description: Run a four-role review chain on a code change. The doubter finds problems. The arbiter decides and writes a plan. The producer implements the plan. The ghost logs and evaluates the work. Use for a non-trivial pull request or diff. Do not use for small edits.
disable-model-invocation: true
---

# Review Chain (orchestrator)

## Purpose

This skill checks a code change and fixes the errors in it.
You are the orchestrator. You run in the main session.
You start four subagents: `doubter`, `arbiter`, `producer`, and `ghost`.
Each subagent has its own file in `.claude/agents/`.
Each subagent runs in a clean context.
The subagents share data through files in `.review-chain/` only.

## When to use

Use this skill when the change is non-trivial.
Examples: a pull request, a refactor, a new feature, a risky fix.

Do not use this skill for a typo, a rename, or a one-line fix.
The chain uses many tokens.

## Files

```
.review-chain/
  change.diff       the diff to check
  goal.md           the goal of the change
  findings.md       doubter output
  plan.md           arbiter output
  report.md         producer output
  ghost-log.md      ghost output
  change-r2.diff    diff for round 2 (optional)
  findings-r2.md    doubter output for round 2 (optional)
```

## Rules for the orchestrator

1. Start each role with the Agent tool. Use the agent name as `subagent_type`.
2. Do not do the work of a role yourself.
3. Give each subagent file paths. Do not give it a summary of earlier roles.
4. Do not change the output of a role. Pass the file as it is.
5. Run the stages in order: doubter, arbiter, producer.
6. Start the ghost after each stage. You can run it in parallel with the next stage.
7. Tell the ghost the stage number in the prompt.
8. Do not give `ghost-log.md` to the doubter, the arbiter, or the producer.
9. Do the checks after each stage (see "Checks").

## Procedure

### Stage 0: Prepare

1. Create the directory `.review-chain/`.
2. Find the diff. Use `git diff`, `git diff --staged`, or a commit range that the user gives.
3. Save the diff to `.review-chain/change.diff`.
4. Write the goal of the change to `.review-chain/goal.md`. Use two to five sentences.
5. If the diff is empty, stop. Tell the user.

### Stage 1: Doubter

1. Start the agent `doubter`.
   Prompt: "Review the change. Input is in .review-chain/. Write findings.md."
2. Wait for the agent to finish.
3. If `findings.md` has no finding with severity HIGH or MEDIUM, stop. Tell the user "No problems found".
4. Start the agent `ghost` with the prompt "Stage 1".

### Stage 2: Arbiter

1. Start the agent `arbiter`.
   Prompt: "Decide on the findings. Input is in .review-chain/. Write plan.md."
2. Wait for the agent to finish.
3. If `plan.md` has no FIX task, stop. Show the REJECT list to the user.
4. Start the agent `ghost` with the prompt "Stage 2".

### Stage 3: Producer

1. Start the agent `producer`.
   Prompt: "Do the FIX tasks. Input is .review-chain/plan.md. Write report.md."
2. Wait for the agent to finish.
3. If `report.md` has the status BLOCKED, show the question to the user. Stop.
4. Start the agent `ghost` with the prompt "Stage 3".

### Stage 4: Second round (optional)

1. Make a new diff. Save it to `.review-chain/change-r2.diff`.
2. Run Stage 1 again. Tell the doubter to read `change-r2.diff` and write `findings-r2.md`.
3. Do not run more than two rounds.
4. Show the remaining findings to the user. Do not loop again.

### Stage 5: Final report

1. Start the agent `ghost` with the prompt "Final stage".
2. Show the user a short summary:
   - number of findings, FIX tasks, and REJECT items
   - tasks done and tasks blocked
   - the main points from the ghost evaluation

## Parallel start example

Send one message with two Agent calls:

```
Agent(description: "Producer implement", subagent_type: "producer",
      prompt: "Do the FIX tasks. Input is .review-chain/plan.md. Write report.md.")
Agent(description: "Ghost stage 2", subagent_type: "ghost",
      prompt: "Stage 2")
```

## Checks

Do these checks after each stage:

- After `doubter` and `arbiter`: `git status` must show no source change.
- After `producer`: `git diff` must show changes in the planned files only.
- After `ghost`: only `.review-chain/ghost-log.md` has changed.

If a role broke its write limit, stop the chain. Tell the user.

## Notes

- A subagent returns only its final message. The real result is in the file. Read the file.
- Use the `model` field in an agent file to give a role a different model.
  A different model for the doubter gives more variety.
- A subagent cannot start other subagents. Only the main session starts them.
