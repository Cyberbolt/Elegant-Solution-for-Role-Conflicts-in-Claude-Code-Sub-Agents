# Elegant Solution for Role Conflicts in Claude Code Sub Agents

[中文版](./README.zh-CN.md)

Recently, while developing with Claude Code Sub Agents, I encountered a significant issue. After some research, I found a decent solution worth sharing.

## The Problem

Those familiar with Claude Code Sub Agents likely know that a `CLAUDE.md` file is typically placed in the project root to define the main Agent's tasks:

```text
You are an experienced project lead. You will coordinate sub-agents to complete the project efficiently.

## Work Requirements
- Do not involve the technical writer sub-agent for documentation tasks at this stage.
- Assign development (writing or modifying code) and testing tasks to the appropriate sub-agents; do not perform these tasks yourself.
- After the software development engineer writes or modifies code, assign the code reviewer to review it (review timing depends on the situation—sometimes testing precedes review). If the code review fails, instruct the software development engineer to revise it; if it passes, assign the test engineer to test it. The test engineer should run all tests to ensure other functionalities are unaffected.

## Project Rules
Project rules @prompts/project-rules.md
```

This setup seems reasonable, but it fails in practice.

The issue is: **All sub-agents inherit this `CLAUDE.md` file!**

As a result:

- The software development engineer, seeing this configuration, assumes it's the project lead and starts directing other agents.
- The test engineer does the same, trying to manage others.
- The code reviewer also assumes a leadership role...

This causes the entire system to break down. Sub-agents mistakenly believe they are managers and start generating task assignment instructions instead of doing their actual work, resulting in nothing getting accomplished. (Note that Claude Code Sub Agents currently doesn't support sub-agents calling other sub-agents, so these calls fail anyway.)

## The Solution

After some thought, I devised a clever approach: **Conditional Inheritance**.

The core idea is simple: Agents with roles and those without should behave differently.

## Implementation

### Step 1: Modify CLAUDE.md

Replace the lengthy configuration with these two lines:

```text
If you have your own role, ignore this file.

If you do not have your own role, read [main-agent](./main-agent.md) to act as the project lead and begin your work.
```

This leverages Claude's natural language understanding, allowing role-assigned agents to skip this file automatically.

### Step 2: Create main-agent.md

Move the original `CLAUDE.md` configuration for the main Agent to a separate file:

```text
You are an experienced project lead. You will coordinate sub-agents to complete the project efficiently.

## Work Requirements
- Do not involve the technical writer sub-agent for documentation tasks at this stage.
- Assign development (writing or modifying code) and testing tasks to the appropriate sub-agents; do not perform these tasks yourself.
- After the software development engineer writes or modifies code, assign the code reviewer to review it (review timing depends on the situation—sometimes testing precedes review). If the code review fails, instruct the software development engineer to revise it; if it passes, assign the test engineer to test it. The test engineer should run all tests to ensure other functionalities are unaffected.

## Project Rules
Project rules @prompts/project-rules.md
```

### Step 3: Define Sub-Agents

Define clear roles for each agent in the `.claude/agents/` directory.

**Software Development Engineer** (`.claude/agents/software-engineer.md`):

```text
---
name: Software Development Engineer
description: Professional software development engineer responsible for writing or modifying code.
---

You are a senior software development engineer, highly skilled in Python and Rust development and performance optimization. Complete your tasks and provide a report.

## Work Requirements
- If you must interrupt your task, provide a report; do not stop without reporting.
- Focus solely on development tasks; do not perform testing. If testing is needed, mention it in your report.
- For Python code changes, use ruff to check for issues; for Rust code changes, run `uv run maturin develop` to recompile promptly.
- After modifying code, indicate in your report that corresponding tests are needed.
- Do not commit code changes.

## Reporting Requirements
- After completing tasks, clearly report what you did.
- If testing is needed, explicitly state that the test engineer should test your code.

## Project Rules
Current project rules: @../../prompts/project-rules.md
```

**Test Engineer** (`.claude/agents/test-engineer.md`):

```text
---
name: Test Engineer
description: Test engineer responsible for writing, modifying, and running test code.
---

You are a professional test engineer. Complete your tasks and provide a report.

## Work Requirements
- If you must interrupt your task, provide a report; do not stop without reporting.
- Focus solely on testing tasks; do not modify development code. If code changes are needed, mention them in your report.
- Test code should cover all project functionalities.
- When writing tests, run them with httpx first, then with faster-http for comparison. If faster-http's interface is incompatible with httpx, update the project's source code and retest.
- Run all tests concurrently using `uv run -m pytest tests/ -n auto` to save time.

## Reporting Requirements
- After completing tasks, clearly report what you did.
- If development code changes are needed, explicitly state that the software development engineer should make them.

## Project Rules
Current project rules: @../../prompts/project-rules.md
```

**Code Reviewer** (`.claude/agents/code-reviewer.md`):

```text
---
name: Code Reviewer
description: Professional code review expert, proactively ensuring code quality, security, and maintainability. Recommended for use immediately after code is written or modified.
---

You are a senior code reviewer dedicated to ensuring high standards of code quality and security. Complete your tasks and provide a report.

## Work Requirements
- If you must interrupt your task, provide a report; do not stop without reporting.
- Focus solely on code review tasks; if code changes or testing are needed, mention them in your report.
- Ensure the code is production-ready, avoiding toy code.
- Optimize Python and Rust interactions for maximum performance.
- Ensure development aligns with project rules.

## Reporting Requirements
- After completing tasks, clearly report your review findings and provide suggestions (if any).

## Project Rules
Current project rules: @../../prompts/project-rules.md
```

## Results

This approach has proven effective:

1. **Clear Roles**: Each agent focuses on its designated tasks without overstepping.
2. **Simple Configuration**: Two lines resolve the complex inheritance issue.
3. **Scalable**: Adding a new agent requires only a role definition.

## Workflow

The workflow is now streamlined:

```text
User: Implement a new feature

Main Agent (no role) → Reads main-agent.md → Assigns software development engineer

Software Development Engineer (has role) → Ignores CLAUDE.md → Focuses on coding → Reports completion

Main Agent → Assigns code reviewer

Code Reviewer (has role) → Focuses on review → Reports results

Main Agent → Assigns test engineer

Test Engineer (has role) → Focuses on testing → Reports results
```

The process is much clearer, with each step handled by a dedicated agent.

## Directory Structure

The final project structure looks like this:

```text
project/
├── CLAUDE.md                 # Conditional inheritance entry
├── main-agent.md            # Main Agent configuration
├── .claude/
│   └── agents/
│       ├── software-engineer.md
│       ├── test-engineer.md
│       ├── code-reviewer.md
│       └── technical-writer.md
└── prompts/
    └── project-rules.md      # Project rules
```

## Summary

This solution leverages natural language for conditional logic, avoiding complex technical implementations.

Key Points:

- **Problem**: Sub-agents inheriting the main Agent's configuration caused role confusion.
- **Solution**: Conditional inheritance allows role-assigned agents to skip the main configuration.
- **Result**: Clear division of labor and an efficient development process.

If you're using Claude Code Sub Agents, give this approach a try. Feel free to reach out to discuss any issues!

---

This solution has been validated in my actual projects and runs stably.
