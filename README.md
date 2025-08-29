# Nomie
Deploy AI agents in enterprise environments without getting fired

Nomie is contextual access control for AI agents. It lets developers build agents that can safely access user data across systems while giving enterprises the granular permission controls they need to approve deployments.

---

## The Problem

AI agents are powerful, but enterprise deployment gets blocked by a simple question: "How do we know this agent won't access data it shouldn't?"

Current solutions force a binary choice:
- **Full access**: Agent can read everything (security team says no)
- **No access**: Agent is useless (business team says no)

Nomie solves this with **contextual permissions** that understand what an agent is trying to do, not just what data it's touching.

## Why Existing Solutions Fall Short

**Basic RBAC**: "This agent can read calendars" doesn't differentiate between reading your availability vs. reading confidential meeting notes.

**Memory platforms**: Store preferences but can't enforce that a travel agent shouldn't see your salary data when booking flights.

**Enterprise identity**: Designed for humans, not for AI agents that need task-specific, temporary access patterns.

## What Nomie Provides

**Intent-aware permissions**: "Book travel" gets calendar availability and hotel preferences, but not confidential meeting details or financial information.

**Cross-system context**: Works across Google Workspace, Slack, CRM systems, and internal tools without requiring agents to understand each system's permission model.

**Developer-friendly deployment**: Single integration that handles enterprise security requirements so developers can focus on agent functionality.

**Audit and compliance**: Every permission request and data access is logged with business context, not just technical metadata.

---

## How It Works

### For Agent Developers
```typescript
import { nomie } from "nomie-sdk";

// Initialize with your agent's capabilities
await nomie.init({
  agentId: "travel-booking-agent",
  capabilities: ["calendar:availability", "user:travel-preferences", "expenses:create"]
});

// Request access with business context
const access = await nomie.requestAccess({
  intent: "book_business_travel",
  userEmail: "user@company.com", 
  dataNeeded: {
    calendar: "availability_only",
    profile: "travel_preferences", 
    expenses: "create_travel_expense"
  }
});

if (access.granted) {
  // Agent gets exactly what it needs, nothing more
  const availability = await access.calendar.getAvailability("2025-01-15", "2025-01-20");
  const preferences = await access.profile.getTravelPreferences();
} else {
  // Clear feedback on what's blocked and why
  console.log(`Access denied: ${access.reason}`);
  console.log(`Required approvals: ${access.pendingApprovals}`);
}
```

### For Enterprise Administrators
```yaml
# Policy configuration
travel_booking_agent:
  intents:
    book_business_travel:
      auto_approve: true
      permissions:
        calendar: ["availability", "create_events"]
        profile: ["travel_preferences", "expense_account"]
      restrictions:
        - no_personal_calendar_details
        - no_salary_information
        - require_manager_approval_over_$5000
      audit_level: "detailed"
```

---

## Core Architecture

### Intent Engine
Maps agent requests to specific data needs:
- "Book travel" → calendar availability + travel preferences
- "Schedule meeting" → calendar write + attendee contact info  
- "Generate report" → read-only access to specified data sources

### Context Broker
Understands data relationships across systems:
- Calendar events can contain confidential information
- User profiles have both business and personal elements
- Financial data requires different controls than preferences

### Permission Resolver
Translates business intent into technical permissions:
- Temporary access tokens for specific operations
- Scoped API calls that filter sensitive information
- Cross-system coordination without exposing credentials

---

## Integration Examples

### Travel Booking Agent
**Before Nomie**: Agent needs full Google Workspace access, security team rejects deployment

**With Nomie**: Agent requests "travel booking" intent, gets calendar availability and travel preferences, security team approves

```typescript
const travelAccess = await nomie.requestAccess({
  intent: "book_business_travel",
  duration: "2_hours", // Access expires automatically
  auditTrail: true
});

// Agent can check availability but not read meeting contents
const availability = await travelAccess.calendar.getAvailability();
// Agent gets travel preferences but not personal information
const preferences = await travelAccess.profile.getTravelPreferences();
```

### Meeting Scheduler
**Before Nomie**: Needs contacts access, calendar write, email sending - too broad for security approval

**With Nomie**: Gets exactly what it needs for scheduling, nothing more

```typescript
const scheduleAccess = await nomie.requestAccess({
  intent: "schedule_internal_meeting",
  participants: ["alice@company.com", "bob@company.com"],
  constraints: {
    timeRange: "business_hours_only",
    duration: "max_2_hours"
  }
});
```

---

## Security Model

### Principle of Least Context
Agents get minimum data needed for their specific task, not broad system access.

### Temporal Permissions  
Access tokens expire automatically when the task completes or after a time limit.

### Intent Verification
Every data request must map to a declared business intent that's been approved.

### Audit by Default
All access requests, approvals, and data usage logged with business context for compliance teams.

### Revocation
Instant revocation of agent access across all systems when policies change or threats detected.

---

## Enterprise Features

### Policy Management
- Visual policy builder for non-technical administrators
- Template policies for common agent use cases  
- Integration with existing identity providers and RBAC systems

### Compliance Reporting
- Detailed audit logs with business context
- Compliance dashboard showing agent activity across systems
- Automated reporting for security reviews

### Risk Management
- Real-time monitoring of unusual agent behavior
- Automatic access revocation on policy violations
- Integration with enterprise security tools (SIEM, DLP)

---

## Getting Started

### Quick Start for Developers
```bash
npm install nomie-sdk
```

```typescript
import { nomie } from "nomie-sdk";

// 1. Register your agent's capabilities
await nomie.registerAgent({
  name: "My Business Agent",
  intents: ["process_invoices", "schedule_meetings"],
  dataRequirements: {
    process_invoices: ["documents:invoices", "accounting:write"],
    schedule_meetings: ["calendar:availability", "contacts:business"]
  }
});

// 2. Request access for specific tasks
const access = await nomie.requestAccess({
  intent: "process_invoices",
  userContext: "finance_team_member"
});

// 3. Use scoped access safely
if (access.granted) {
  const invoices = await access.documents.getInvoices({ status: "pending" });
  await access.accounting.createEntries(processedInvoices);
}
```

### Enterprise Setup
1. **Connect identity systems**: Integrate with existing SSO and user directories
2. **Define policies**: Set up approval workflows and access rules
3. **Deploy agents**: Developers can now build agents that pass security review
4. **Monitor and audit**: Track all agent activity through compliance dashboard

---

## Why This Approach Works

### For Security Teams
- **Granular control**: Approve specific business use cases, not broad system access
- **Clear audit trail**: Every agent action tied to business intent and user context
- **Risk reduction**: Agents can't accidentally access sensitive data outside their scope
- **Compliance ready**: Built-in logging and reporting for regulatory requirements

### For Developers  
- **Faster approvals**: Security teams understand and can approve specific business use cases
- **Simpler integration**: One SDK handles complex enterprise permission requirements
- **Better reliability**: Clear permission boundaries prevent runtime access failures
- **Easier debugging**: Know exactly what data your agent can and cannot access

### For Business Teams
- **Faster deployment**: Agents pass security review without months of back-and-forth
- **Better functionality**: Agents can access the data they need to be useful
- **Trust and adoption**: Users confident that agents follow appropriate access controls
- **Measurable ROI**: Deploy AI agents that solve business problems without security bottlenecks

---

## Competitive Advantages

**vs. Memory platforms**: We don't just store preferences—we enforce contextual access across all enterprise systems

**vs. Basic RBAC**: We understand business intent, not just technical permissions

**vs. Enterprise identity**: Purpose-built for AI agents' unique access patterns and temporary permission needs

**vs. Building in-house**: We handle the complex enterprise security requirements so you can focus on agent functionality

---

## Roadmap

**Q1 2025**: Core platform with Google Workspace, Slack, and Salesforce integrations  
**Q2 2025**: Advanced policy engine with conditional access and risk scoring  
**Q3 2025**: Multi-tenant SaaS with self-service policy management  
**Q4 2025**: AI-powered policy recommendations and automated compliance reporting

---

*Deploy AI agents that enterprises actually approve*
