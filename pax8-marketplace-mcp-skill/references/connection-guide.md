# Connection Guide — Pax8 MCP + Skill Setup

For full setup instructions across all supported AI clients, see:

**[devx.pax8.com/docs/mcp-server](https://devx.pax8.com/docs/mcp-server)**

That guide covers connecting the Pax8 MCP Server and installing the skill for Claude, Cursor, VS Code Copilot, n8n, and other compatible clients.

---

## Verifying the Connection

Once connected, ask your AI assistant:

> "Can you list my companies in Pax8?"

If the MCP is connected correctly, it will call `pax8-list-companies` and return your account's companies. If you see an error about authentication or no tools available, double-check the MCP configuration using the guide above.
