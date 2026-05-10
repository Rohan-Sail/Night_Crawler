
          ╔══════════════════════════════════════════════════════════════════════════════════════════════════════════╗
          ║   ███╗   ██╗██╗ ██████╗ ██╗  ██╗████████╗    ██████╗██████╗  █████╗ ██╗    ██╗██╗     ███████╗██████╗    ║
          ║   ████╗  ██║██║██╔════╝ ██║  ██║╚══██╔══╝   ██╔════╝██╔══██╗██╔══██╗██║    ██║██║     ██╔════╝██╔══██╗   ║
          ║   ██╔██╗ ██║██║██║  ███╗███████║   ██║      ██║     ██████╔╝███████║██║ █╗ ██║██║     █████╗  ██████╔╝   ║
          ║   ██║╚██╗██║██║██║   ██║██╔══██║   ██║      ██║     ██╔══██╗██╔══██║██║███╗██║██║     ██╔══╝  ██╔══██╗   ║
          ║   ██║ ╚████║██║╚██████╔╝██║  ██║   ██║      ╚██████╗██║  ██║██║  ██║╚███╔███╔╝███████╗███████╗██║  ██║   ║
          ║   ╚═╝  ╚═══╝╚═╝ ╚═════╝ ╚═╝  ╚═╝   ╚═╝       ╚═════╝╚═╝  ╚═╝╚═╝  ╚═╝ ╚══╝╚══╝ ╚══════╝╚══════╝╚═╝  ╚═╝   ║
          ║                                                                                                          ║
          ║                                         N I G H T C R A W L E R                                          ║
          ╚══════════════════════════════════════════════════════════════════════════════════════════════════════════╝
                                          [ Autonomous AI Agent for Ethical Hacking ]

---
<div align="center">
  <img width="672" height="960" alt="Nightcrawler Architecture"
       src="https://github.com/user-attachments/assets/5a53f786-8eac-4d57-b99b-e3543b1b7d28" />
</div>

---

# Full Architecture Breakdown

## 1. Orchestrator / LLM Brain

This is the agent's **CEO layer**.

It receives the target scope, breaks it into sub-tasks, selects which tools to invoke, retrieves relevant memory, and decides what to do next.

Use a capable model such as:

- GPT-4o
- Claude
- Llama 3

Recommended reasoning frameworks:

- ReAct
- Plan-and-Execute

The orchestrator must support multi-step chaining workflows such as:

```text
Recon → Identify Tech Stack → Match Known CVEs → Attempt Exploit → Verify Impact → Report
```

---

## 2. Core Modules

| Module | Responsibility | Key Tools Wrapped |
|---|---|---|
| Recon | Asset discovery, port scanning, fingerprinting | Nmap, Amass, Subfinder, Shodan API |
| Vulnerability Scanner | CVE matching, misconfiguration detection, web scanning | Nuclei, OWASP ZAP, Nikto |
| Exploit Engine | Payload generation, chaining, post-exploitation | Metasploit RPC, SQLMap, custom scripts |
| Fuzzer | Input mutation, crash detection, edge-case exploration | ffuf, Boofuzz, custom LLM-generated inputs |
| Reporter | Triage findings, CVSS scoring, deduplication, report generation | LLM-powered markdown/PDF generation |

---

## 3. Tool Integration Layer

Every security tool should be wrapped as a callable agent tool with a defined schema:

- Accepted inputs
- Expected outputs
- Error handling behavior
- Structured return format

Think of this as an **API adapter layer**.

All tools should communicate results back to the orchestrator in structured JSON so the LLM can reason over them safely and consistently.

Example:

```json
{
  "tool": "nuclei",
  "target": "api.example.com",
  "severity": "high",
  "evidence": {
    "template": "CVE-2024-XXXX",
    "response_code": 200
  }
}
```

---

## 4. Memory & Context System

Three categories of memory are essential.

### Short-Term Memory (Working Memory)

Tracks:

- Current task context
- Recent tool outputs
- Active hypotheses
- Temporary execution state

### Long-Term Memory (Vector Database)

Stores:

- Past findings
- Known-good payloads
- Program-specific notes
- Historical exploit paths

Suggested vector databases:

- Chroma
- Weaviate
- Pinecone

### Knowledge Graph

Tracks relationships between discovered assets:

```text
Domain → Subdomain → IP → Open Port → Service → CVE
```

This allows contextual reasoning and attack-path mapping.

---

## 5. Safety & Ethics Layer (Non-Negotiable)

### Scope Enforcement

Every action must be validated against a hard-coded scope policy before execution.

If a target is out of scope:

```text
Action Rejected
```

### Rate Limiting

Prevent accidental denial-of-service conditions by enforcing:

- Per-target request budgets
- Concurrency limits
- Adaptive throttling

### Human-in-the-Loop Gates

High-risk actions require explicit approval before execution:

- Actual exploitation
- Credential attacks
- Destructive operations

### Audit Logging

Every action should be logged with:

- Timestamp
- Tool used
- Target
- Parameters
- Outcome

---

## 6. Reasoning Loop (ReAct Cycle)

```text
Observe  → Tool Output
Think    → Interpret Result
Act      → Select Next Tool
Reflect  → Validate Expectations
Re-plan  → Update Goal Tree
```

A dedicated **self-critique phase** should exist specifically for reducing false positives.

The agent must actively challenge its own findings before reporting them.

---

# The 7 Architectural Pillars That Separate This from Basic Agents


## 1. Evidence-First, Never Claim-First

The LLM's role is to:

- Plan
- Interpret
- Coordinate

The tools provide proof.

The agent must never claim a vulnerability exists unless structured evidence is returned from a validated tool.

This is the single most important anti-hallucination mechanism.
<div align="center">
<img width="672" height="600" alt="Evidence First"
     src="https://github.com/user-attachments/assets/08ad9eb4-2c92-4609-bbc1-4172f3ef2642" />
</div>

---

## 2. Fine-Tuned LLM + Deep RAG

Avoid using a vanilla base model.

Fine-tune using:

- HackerOne disclosed reports
- Exploit-DB writeups
- CVE descriptions
- OWASP Testing Guide
- Security research blogs

At every reasoning step, the agent queries the vector database using the active context.

Example:

```text
Spring Boot 2.3.1 + Actuator Endpoint
→ Retrieve Real-World Exploitation Patterns
→ Generate Context-Aware Test Cases
```
<div align="center">
<img width="672" height="600" alt="Fine Tuned LLM"
     src="https://github.com/user-attachments/assets/fab0b9c6-34ed-4cd1-ab84-c1ab015895c2" />
</div>

---

## 3. Strict Typed Message Bus Between Agents

Agents should never communicate using raw free-form strings.

Every message must be a typed structure:

- Task
- ToolResult
- UnvalidatedFinding
- ValidatedFinding
- BlockedAction

Benefits:

- Prevents prompt injection
- Eliminates ambiguity
- Improves auditability
- Creates deterministic hand-offs

<div align="center">
<img width="672" height="600" alt="Typed Message Bus"
     src="https://github.com/user-attachments/assets/9d925b8f-49a6-43ff-b1d0-c9d8a16bb0d8" />
</div>

---

## 4. Task State Machine (Executive Function)

The task system is sacred.

Task lifecycle:

```text
PENDING
→ IN_PROGRESS
→ AWAITING_VALIDATION
→ DONE | BLOCKED
```

Rules:

- Tasks cannot be marked complete without evidence
- Tasks cannot be skipped
- Reordering requires re-planning

This mirrors how senior penetration testers operate:

- Methodical
- Documented
- Evidence-driven

<div align="center">
<img width="672" height="600" alt="Task State Machine"
     src="https://github.com/user-attachments/assets/a206b93c-7249-4c8f-90b2-629b29e815b9" />
</div>

---

## 5. Triple-Gate Validation (Zero False Positives)

Multiple validation gates should exist before any finding reaches the final report.

Critical validation stages include:

1. Reproducibility from clean state
2. Differential analysis (payload-on vs payload-off)
3. Exploitability proof

If a finding cannot be demonstrated, it should not be reported.

<div align="center">
<img width="672" height="600" alt="Triple Gate Validation"
     src="https://github.com/user-attachments/assets/98edfd90-2b34-40f5-8806-a2967d86b670" />
</div>

---

## 6. Dynamic Testcase Generation, Not Templates

Example input:

```http
POST /api/search
{
  "query": "test"
}
```

The agent should not blindly run generic SQL injection wordlists.

Instead, it should:

1. Analyze the response structure
2. Identify likely backend technologies
3. Generate stack-specific payloads
4. Adapt encoding strategies dynamically
5. Learn WAF behavior in real time

Capabilities include:

- Cloudflare bypass adaptation
- Akamai-specific evasion
- ModSecurity signature-aware payload mutation

<div align="center">
<img width="672" height="600" alt="Dynamic Testcase Generation"
     src="https://github.com/user-attachments/assets/aadbf604-7b11-4faf-8d70-5f813e155678" />
</div>

---

## 7. Business Logic Agent (Highest Value Layer)

This is what separates senior testers from commodity scanners.

The business logic agent should:

- Map application state machines
- Understand workflow constraints
- Detect logic flaws

Example application flow:

```text
Login → Cart → Payment → Confirmation
```

The agent systematically tests violations such as:

- Negative quantities
- Payment bypasses
- Token replay attacks
- Currency arbitrage
- Referral abuse
- State desynchronization

These vulnerabilities rarely appear in CVE databases and often represent the highest-value findings.

<div align="center">
<img width="672" height="600" alt="Business Logic Agent"
     src="https://github.com/user-attachments/assets/40a0d6d3-4f6a-453a-beb0-c9ade412a615" />

</div>
