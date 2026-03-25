---
name: netra-pii-and-input-guardrails
description: 'Configure PII detection and prompt-injection scanning using Netra detectors and scanners.'
argument-hint: 'Describe your sensitive fields, action policy (MASK/FLAG/BLOCK), and where scanning runs in request flow.'
---

# Netra PII And Input Guardrails

Use this skill to protect prompts and logs from sensitive data and malicious inputs.

## When To Use
- You must detect/mask/block PII before sending text to LLMs.
- You need prompt-injection scanning before agent/tool execution.
- You want explicit security actions (`MASK`, `FLAG`, `BLOCK`).

## Procedure
1. Choose a PII detector (`get_default_detector` or `RegexPIIDetector`).
2. Set action type based on policy (`MASK` for safe continuation is common).
3. Scan user input with `InputScanner` before processing.
4. Stop requests on blocked violations.
5. Send masked content downstream when allowed.

## Python Pattern
```python
from netra.pii import get_default_detector
from netra.input_scanner import InputScanner, ScannerType

pii_detector = get_default_detector(
    action_type="MASK",
    entities=["EMAIL_ADDRESS", "PHONE_NUMBER", "CREDIT_CARD"],
)
scanner = InputScanner(scanner_types=[ScannerType.PROMPT_INJECTION])


def secure_input_pipeline(user_text: str) -> str:
    scan_result = scanner.scan(user_text, is_blocked=True)
    if scan_result.has_violation:
        raise ValueError("Malicious input detected")

    pii_result = pii_detector.detect(user_text)
    return pii_result.masked_text if pii_result.has_pii else user_text
```

## TypeScript Pattern
```typescript
import { Netra } from "netra-sdk";

const EMAIL_RE = /[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}/g;
const INJECTION_RE = /ignore previous instructions|reveal system prompt/i;

export function secureInputPipeline(userText: string): string {
    const span = Netra.startSpan("security-input-scan");
    try {
        if (INJECTION_RE.test(userText)) {
            span.setError("prompt_injection_detected");
            throw new Error("Malicious input detected");
        }

        const masked = userText.replace(EMAIL_RE, "[EMAIL_ADDRESS]");
        span.setAttribute("security.masked", masked !== userText);
        span.setSuccess();
        return masked;
    } finally {
        span.end();
    }
}
```

## Checklist
- Scanner runs before model/tool calls.
- PII action type matches policy requirements.
- Blocked inputs fail fast with safe error responses.
- Security metadata is logged without storing secret values.

## References
- https://docs.getnetra.ai/Observability/Traces/overview
- https://docs.getnetra.ai/sdk-reference/sdk/overview
- https://docs.getnetra.ai/sdk-reference/sdk/typescript
