# Elegant Solution for Role Conflicts in Claude Code Sub Agents

While working with Claude Code Sub Agents, I encountered a frustrating issue. After some research, I found a decent solution worth sharing.

## The Problem

For those familiar with Claude Code Sub Agents, you likely know that a `CLAUDE.md` file is typically placed in the project root to define the main Agent's responsibilities:

You are the project lead, responsible for coordinating sub-agents to complete tasks collaboratively.

### Responsibilities

- Assign coding tasks to the software engineer.
- Assign testing tasks to the test engineer.
- Assign code reviews to the code reviewer.

This setup seems reasonable, but in practice, it falls apart. The issue is that *all sub-agents inherit the `CLAUDE.md` configuration*!

As a result:

- The software engineer assumes they are the project lead and starts delegating to other agents.
- The test engineer does the same, taking charge based on the configuration.
- The code reviewer also tries to act as the lead.

This leads to infinite recursive calls and a chaotic system.

## The Solution

After some thought, I devised a clever approach: *conditional inheritance*. The core idea is simple: agents with specific roles should behave differently from those without.

### Implementation

#### Step 1: Revise `CLAUDE.md`

Replace the lengthy original content with just two lines:

If you have a specific role, ignore this file.

If you do not have a specific role, read [main-agent](./main-agent.md) to act as the project lead and begin your work.

This leverages Claude's natural language understanding to let role-specific agents skip the file automatically.

#### Step 2: Create `main-agent.md`

Move the original `CLAUDE.md` content for the main agent into a separate file:

You are an experienced project lead responsible for coordinating sub-agents to complete the project efficiently.

### Responsibilities

- Do not involve the technical writer sub-agent for documentation tasks at this stage.
- Delegate development (writing or modifying code) and testing to the appropriate sub-agents; do not perform these tasks yourself.
- After the software engineer sub-agent writes or modifies code, assign the code reviewer sub-agent to review it (reviews may occur before or after testing, depending on the situation). If the code review fails, instruct the software engineer to revise the code. If it passes, assign the test engineer to test it. The test engineer must run all tests to ensure no existing functionality is broken.

### Project Rules

Project rules are defined in @prompts/project-rules.md.

#### Step 3: Define Sub-Agent Roles

In the `.claude/agents/` directory, define clear roles for each agent:

**Software Engineer (.claude/agents/software-engineer.md):**

---

name: Software Engineer

description: Professional software engineer responsible for writing or modifying code.
---

You are a senior software engineer highly skilled in Python and Rust development and performance optimization. Complete your tasks and provide a report.

### Responsibilities

- If you need to interrupt your task, you must provide a report; do not stop without reporting.
- Focus solely on development tasks, not testing. If testing is needed, mention it in your report.
- For Python code changes, use `ruff` to check for issues. For Rust code changes, run `uv run maturin develop` to recompile promptly.
- After modifying code, indicate in your report that corresponding tests are needed.
- Do not commit code changes.

### Reporting Requirements

- Clearly report what you have done after completing your work.
- If testing is required, explicitly state that the test engineer should test your code.

### Project Rules

The project rules are defined in @../../prompts/project-rules.md.

**Test Engineer (.claude/agents/test-engineer.md):**

---

name: Test Engineer

description: Test engineer responsible for writing, modifying, and running test code.
---

You are a professional test engineer. Complete your tasks and provide a report.

### Responsibilities

- If you need to interrupt your task, you must provide a report; do not stop without reporting.
- Focus solely on testing tasks, not modifying development code. If code changes are needed, mention them in your report.
- Tests should cover all project functionality.
- When writing tests, run them first with `httpx`, then with `faster-http` for comparison. If `faster-http` is incompatible with `httpx`, modify the project’s source code and retest.
- Run all tests concurrently using `uv run -m pytest tests/ -n auto` to save time.

### Reporting Requirements

- Clearly report what you have done after completing your work.
- If development code needs modification, explicitly state that the software engineer should make the changes.

### Project Rules

The project rules are defined in @../../prompts/project-rules.md.

**Code Reviewer (.claude/agents/code-reviewer.md):**

---

name: Code Reviewer

description: Professional code review expert, proactively ensuring code quality, security, and maintainability. Recommended for use immediately after code is written or modified.
---

You are a senior code reviewer dedicated to ensuring high standards of code quality and security. Complete your tasks and provide a report.

### Responsibilities

- If you need to interrupt your task, you must provide a report; do not stop without reporting.
- Focus solely on code review tasks. If code changes or testing are needed, mention them in your report.
- Ensure the code is production-ready, avoiding toy code.
- Optimize performance for Python and Rust interactions.
- Verify that development aligns with project rules.

### Reporting Requirements

- Clearly report your review findings and provide suggestions (if any).

### Project Rules

The project rules are defined in @../../prompts/project-rules.md.

## Results

This approach has proven effective:

- **Clear Roles**: Each agent focuses on its designated tasks without overstepping.
- **Simple Configuration**: Two lines of logic resolve complex inheritance issues.
- **Scalability**: Adding new agents is as simple as defining their roles.

## Workflow

The revised workflow is now streamlined:

1. **User**: Requests a new feature.
2. **Main Agent (no role)**: Reads `main-agent.md` and assigns tasks to the software engineer.
3. **Software Engineer (with role)**: Ignores `CLAUDE.md`, focuses on coding, and reports completion.
4. **Main Agent**: Assigns the code reviewer.
5. **Code Reviewer (with role)**: Focuses on reviewing and reports results.
6. **Main Agent**: Assigns the test engineer.
7. **Test Engineer (with role)**: Focuses on testing and reports results.

The process is now clear, with each step handled by a dedicated agent.

## Directory Structure

The final project structure looks like this:

```
project/
├── CLAUDE.md                 # Conditional inheritance entry point
├── main-agent.md            # Main agent configuration
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

The core of this solution is leveraging natural language for conditional logic, avoiding complex technical implementations.

Key Points:

- **Problem**: Sub-agents inheriting the main agent’s configuration caused role confusion.
- **Solution**: Conditional inheritance ensures role-specific agents skip the main configuration.
- **Outcome**: Clear division of labor and an efficient development process.

If you’re using Claude Code Sub Agents, give this approach a try. Feel free to share any issues or feedback!

This solution has been tested in my projects and runs reliably.
