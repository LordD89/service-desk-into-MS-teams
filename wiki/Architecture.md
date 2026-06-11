# Architecture

Overview of the Service Desk MS Teams Integration system architecture.

## System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Microsoft Teams                         │
│                    (User Interface)                         │
└────────────────────────┬────────────────────────────────────┘
                         │
                         │ REST API
                         │
┌────────────────────────▼────────────────────────────────────┐
│              Restify HTTP Server                            │
│           (Port 3978 / HTTPS on Azure)                      │
├─────────────────────────────────────────────────────────────┤
│  POST /api/messages     - Handle bot messages              │
│  GET  /health          - Health check endpoint             │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────┐
│           Microsoft Bot Framework                           │
│        (CloudAdapter & Authentication)                      │
├─────────────────────────────────────────────────────────────┤
│  - Message routing                                          │
│  - Activity handling                                        │
│  - Authentication & verification                           │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────┐
│                  TeamsBot Class                             │
│          (ActivityHandler, State Management)                │
├─────────────────────────────────────────────────────────────┤
│  - onMessage()        - Handle incoming messages            │
│  - onMembersAdded()   - Welcome new members                │
│  - Conversation state persistence                          │
└────────────────────────┬────────────────────────────────────┘
                         │
        ┌────────────────┴────────────────┐
        │                                 │
┌───────▼──────────────┐    ┌────────────▼─────────────┐
│  CommandHandler      │    │  ServiceDeskConnector    │
│                      │    │                          │
│ - parseCommand()     │    │ - getRecentTickets()     │
│ - createTicket()     │    │ - searchTickets()        │
│ - listTickets()      │    │ - getTicket()            │
│ - searchTickets()    │    │ - createTicket()         │
│ - getTicketStatus()  │    │                          │
│ - showHelp()         │    │ Adapters:                │
│                      │    │ - JiraConnector          │
│ Adaptive Cards       │    │ - AzureDevOpsConnector   │
│ - Rich formatting    │    │ - ServiceNowConnector    │
│ - Interactive UI     │    │                          │
└──────────────────────┘    └────────────┬─────────────┘
                                         │
                                    Axios HTTP
                                         │
                    ┌────────────────────┼────────────────────┐
                    │                    │                    │
            ┌───────▼────────┐  ┌────────▼─────────┐  ┌──────▼──────────┐
            │  Jira Cloud    │  │ Azure DevOps    │  │  ServiceNow     │
            │  REST API      │  │  REST API       │  │  REST API       │
            ├────────────────┤  ├─────────────────┤  ├─────────────────┤
            │ - Issues       │  │ - Work Items    │  │ - Incidents     │
            │ - Projects     │  │ - Projects      │  │ - Changes       │
            │ - Search       │  │ - Iterations    │  │ - Problems      │
            └────────────────┘  └─────────────────┘  └─────────────────┘
```

## Component Details

### 1. Restify HTTP Server
**Location:** `src/index.ts`

Handles:
- HTTP request routing
- Message endpoints
- Health checks
- Error handling

```typescript
server.post('/api/messages', async (req, res) => {
  await adapter.process(req, res, async (context) => {
    await bot.run(context);
  });
});
```

### 2. Bot Framework Adapter
**Library:** `botbuilder`

Handles:
- Bot authentication
- Activity validation
- Token verification
- Message encryption/decryption

### 3. TeamsBot Class
**Location:** `src/bot/TeamsBot.ts`

Extends: `ActivityHandler`

**Responsibilities:**
- Message reception
- Activity routing
- Conversation state management
- Member addition handling

```typescript
export class TeamsBot extends ActivityHandler {
  onMessage() - Handle user messages
  onMembersAdded() - Welcome new team members
  run() - Execute bot logic
}
```

### 4. CommandHandler
**Location:** `src/bot/CommandHandler.ts`

**Functions:**
- `handleCommand()` - Parse and route commands
- `createTicket()` - Handle create command
- `listTickets()` - Handle list command
- `searchTickets()` - Handle search command
- `getTicketStatus()` - Handle status command
- `showHelp()` - Handle help command

**Output Format:** Adaptive Cards

### 5. ServiceDeskConnector
**Location:** `src/connectors/ServiceDeskConnector.ts`

**Abstract Methods:**
```typescript
getRecentTickets(limit: number): Promise<Ticket[]>
searchTickets(query: string): Promise<Ticket[]>
getTicket(ticketId: string): Promise<Ticket>
createTicket(summary: string, description: string): Promise<string>
```

**Implementations:**
- Jira Service Management
- Azure DevOps
- ServiceNow
- Extensible for custom systems

## Data Flow

### User sends message

```
1. User: "@bot create ticket" (Teams)
   │
2. ──→ Teams Client sends to Teams Service
   │
3. ──→ Teams Service routes to Bot Endpoint
   │
4. ──→ POST /api/messages (Restify Server)
   │
5. ──→ CloudAdapter processes & authenticates
   │
6. ──→ TeamsBot.onMessage() called
   │
7. ──→ CommandHandler.handleCommand()
   │
8. ──→ createTicket() method
   │
9. ──→ Adaptive Card displayed
   │
10. ←── Response sent to Teams
```

### Command with service desk call

```
1. User: "@bot list tickets"
   │
2. ──→ Bot receives message
   │
3. ──→ CommandHandler.listTickets()
   │
4. ──→ ServiceDeskConnector.getRecentTickets()
   │
5. ──→ Jira API call (HTTP GET)
   │
6. ──→ Parse response
   │
7. ──→ Format Adaptive Card
   │
8. ←── Display in Teams
```

## State Management

### Memory Storage
```typescript
storage = new MemoryStorage()
conversationState = new ConversationState(storage)
```

**Note:** For production, use persistent storage (Cosmos DB, SQL, etc.)

## Security Flow

```
User Message
    │
    ├─→ Verify HMAC signature
    │
    ├─→ Check Bot ID
    │
    ├─→ Validate App ID
    │
    └─→ Authenticate service desk credentials
        │
        └─→ Execute command
```

## Deployment Architecture

### Local Development
```
localhost:3978
    ↑
    │ 
Restify Server
    │
    ├─→ .env (local config)
    │
    └─→ Teams Emulator (for testing)
```

### Azure Production
```
https://{app-name}.azurewebsites.net
    ↑
    │
Azure App Service
    │
    ├─→ Application Settings (env vars)
    ├─→ App Insights (monitoring)
    ├─→ Key Vault (secrets)
    │
    └─→ Teams Service
```

## Scalability Considerations

1. **Horizontal Scaling:**
   - Stateless design allows multiple instances
   - Use Azure App Service Plan for auto-scaling

2. **Performance:**
   - Cache recent tickets
   - Rate limit commands
   - Implement command queuing

3. **Storage:**
   - Move from MemoryStorage to persistent DB
   - Use Azure Cosmos DB or SQL Database

4. **Monitoring:**
   - Application Insights integration
   - Log analysis and alerting
   - Performance metrics

---

**Want to extend?** → [Development](Development)  
**Deployment ready?** → [Deployment](Deployment)
