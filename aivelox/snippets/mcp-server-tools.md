# MCP Server Tool Registration — Code Pattern

How aiVelox exposes tools to AI assistants via the Model Context Protocol.

## What MCP Tools Look Like

Each tool is a self-describing function that AI assistants discover automatically. The assistant reads the tool definition (name, description, parameters) and decides when and how to call it.

```typescript
// Tool: Get tasks for a project
// The AI assistant sees this description and knows when to call it
const tools = [
  {
    name: 'aivelox_get_tasks',
    description: 'Fetch tasks for a project. Optionally filter by column name.',
    parameters: {
      projectId: { type: 'string', required: true },
      columnName: { type: 'string', required: false },
    },
    handler: async ({ projectId, columnName }) => {
      const tasks = await getTasksForProject(projectId, columnName);
      return formatTasksForAgent(tasks);
    },
  },

  {
    name: 'aivelox_update_status',
    description: 'Update the AI work status of a task. Use "working" when actively processing, "waiting" when blocked, "done" when complete.',
    parameters: {
      taskId: { type: 'string', required: true },
      status: { type: 'string', enum: ['working', 'waiting', 'done'] },
    },
    handler: async ({ taskId, status }, context) => {
      // Verify agent has access to this task's project
      await verifyAgentAccess(context.agentId, taskId);

      await updateTaskAiStatus(taskId, status);
      await auditLog('ai_status_update', {
        taskId,
        status,
        agentId: context.agentId,
        timestamp: new Date(),
      });

      return { success: true, taskId, newStatus: status };
    },
  },

  {
    name: 'aivelox_add_task',
    description: 'Create a new task. Agents can only add to backlog or rework columns.',
    parameters: {
      projectId: { type: 'string', required: true },
      title: { type: 'string', required: true },
      description: { type: 'string', required: false },
      columnName: { type: 'string', enum: ['backlog', 'rework'] },
    },
    handler: async ({ projectId, title, description, columnName }, context) => {
      // TRUST BOUNDARY: Agents cannot add to sprint, approved, or deployed
      if (!['backlog', 'rework'].includes(columnName)) {
        return { error: 'Agents can only add tasks to backlog or rework' };
      }

      const task = await createTask({
        projectId,
        title,
        description,
        column: columnName,
        source: 'ai-suggested',  // Auto-tagged for tracking
        createdBy: context.agentId,
      });

      await auditLog('task_created_by_agent', {
        taskId: task.id,
        agentId: context.agentId,
      });

      return { success: true, taskId: task.id };
    },
  },
];
```

## The Trust Boundary Pattern

```
AI Agent Capabilities:
  CAN:    read tasks, update own status, add notes, create in backlog/rework
  CANNOT: move to approved, move to deployed, delete tasks, modify other agents' work

Human (PM) Capabilities:
  CAN:    everything above + approve, deploy, delete, reassign
```

This isn't just access control — it's a design philosophy. AI agents are powerful at execution but shouldn't make judgment calls about production readiness. The PM reviews and promotes. The agent reports and suggests.

## Webhook vs MCP: Same Pipeline, Different Entry Points

```
MCP Tool Call ──┐
                ├──→ Auth Layer ──→ RBAC Check ──→ Business Logic ──→ Audit Log ──→ Database
Webhook POST ───┘
```

Both integration points converge at the auth layer. The business logic, RBAC checks, and audit logging are shared. This means:
- No duplicate code for different integration methods
- Consistent security regardless of entry point
- One audit trail for all agent activity
