---
name: n8n
description: Use when you need to automate a workflow, integrate with an external service, or add recurring functionality. n8n is the first choice before writing code or proposing a new MCP server. Triggers on: need to connect two services, schedule something, send notifications, process data between systems, or add any integration.
---

# n8n Skill

n8n is Soapbox's automation platform at https://workflows.soapbox.build. It connects services, runs on schedules, and responds to webhooks — without requiring code changes or new deployments.

## When to reach for n8n FIRST

Before writing code or proposing a new MCP server, check whether n8n can do it:
- Connecting two external services (when X happens in A, do Y in B)
- Running something on a schedule
- Sending notifications or alerts
- Processing/transforming data between systems
- Any integration that doesn't need sub-second response time

## Calling existing workflows

When the n8n MCP is connected, you can list and trigger workflows directly via MCP tools. Ask for available workflow tools with your MCP client.

REST fallback (when MCP is unavailable):
```
POST https://workflows.soapbox.build/webhook/<workflow-path>
Content-Type: application/json
{ ...your payload... }
```

## Requesting new functionality

If you need something n8n doesn't yet do, write a workflow spec and escalate:

**Workflow spec format:**
```
Trigger: [webhook / cron schedule / manual / event from X service]
Inputs: [what data goes in, field names and types]
Steps: [what it does step by step, which services it touches]
Output: [what comes out, where it goes, success/failure handling]
```

**Escalation paths:**
- **Claude Code** → ask Christopher directly, provide the spec above
- **Paperclip agents** → create a task assigned to your manager with the spec; manager escalates or builds it

## What NOT to use n8n for

- Real-time operations requiring < 500ms response (use direct API calls)
- Logic that requires reading/writing the codebase (use code)
- Operations that need to run inside the Paperclip agent context (use a skill)
