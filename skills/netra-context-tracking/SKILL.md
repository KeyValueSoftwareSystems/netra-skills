---
name: netra-context-tracking
description: 'Implement request/session/user/tenant context tracking and conversation logging with Netra.'
argument-hint: 'Describe your request flow, where user/session IDs come from, and what context attributes must be tracked.'
---

# Netra Context Tracking

Use this skill to apply consistent identity and conversation context in Netra traces.

## When To Use
- You need per-request identity (`user_id`, `session_id`, `tenant_id`) in traces.
- You want custom attributes and business events attached to spans.
- You need conversation input/output logs for multi-turn apps.

## Procedure
1. Set IDs at request entry before any downstream LLM/tool calls.
2. Attach stable custom attributes (tier, region, plan, channel).
3. Record business events for key milestones.
4. Add conversation entries for user inputs and assistant outputs.
5. Clear or replace context at the next request boundary.

## Python Pattern
```python
from netra import Netra
from netra.session_manager import ConversationType


def handle_request(request):
    Netra.set_user_id(request.user_id)
    Netra.set_session_id(request.session_id)
    Netra.set_tenant_id(request.tenant_id)

    Netra.set_custom_attributes("customer_tier", request.customer_tier)
    Netra.set_custom_attributes("region", request.region)

    Netra.add_conversation(
        conversation_type=ConversationType.INPUT,
        role="user",
        content=request.message,
    )

    response = run_agent(request.message)

    Netra.add_conversation(
        conversation_type=ConversationType.OUTPUT,
        role="assistant",
        content=response,
    )

    Netra.set_custom_event(
        event_name="request_completed",
        attributes={"status": "success"},
    )

    return response
```

## TypeScript Pattern
```typescript
import { Netra } from "netra-sdk";

type RequestLike = {
    userId: string;
    sessionId: string;
    tenantId: string;
    customerTier: string;
    region: string;
    message: string;
};

async function handleRequest(request: RequestLike): Promise<string> {
    Netra.setUserId(request.userId);
    Netra.setSessionId(request.sessionId);
    Netra.setTenantId(request.tenantId);

    Netra.setCustomAttributes({ key: "customer_tier", value: request.customerTier });
    Netra.setCustomAttributes({ key: "region", value: request.region });

    const response = await runAgent(request.message);

    Netra.setCustomEvent({
        event_name: "request_completed",
        attributes: { status: "success" },
    });

    return response;
}
```

## Checklist
- IDs are set once per request and not reused across users.
- Context is set before first traced operation.
- Event payloads avoid secrets and raw PII.
- Conversation logs are enabled only where policy allows.

## References
- https://docs.getnetra.ai/sdk-reference/sdk/overview
- https://docs.getnetra.ai/Observability/Traces/overview
- https://docs.getnetra.ai/sdk-reference/sdk/typescript
