---
title: "Using LLMs the Simplest Way Possible — Without Sacrificing Reliability"
excerpt: "Call an LLM, get structured data back, use it safely in production. The simplest reliable path for backend engineers: schema, validation, retries, and clean boundaries."
categories: [ai]
tags:
  - LLM
  - backend
  - reliability
  - Python
  - production
  - schema
---

There's a temptation when adopting LLMs to jump straight into orchestration frameworks.

CrewAI.
LangChain.
Autonomous agents.
Multi-step reasoning graphs.
Parallel tool execution.

And to be clear — frameworks like CrewAI and other orchestration systems absolutely have their place. They're powerful. They accelerate experimentation. They're fantastic for complex agentic workflows.

But most backend engineers don't start there.

Most backend engineers want something much simpler:

* Call an LLM
* Get structured data back
* Use it safely in production
* Sleep at night

This article is about that path.

Not the most complex.

Not the most abstract.

The simplest reliable way to use LLMs in a deterministic backend system.

---

# Start With the Right Mental Model

An LLM is not magic.

It is not deterministic.

It is not a database.

It is not a microservice with strict contracts.

It is a probabilistic text generator.

If you treat it like a normal backend service, you will eventually ship a bug.

The correct mindset is this:

> An LLM is an intelligent but unreliable collaborator.
> Your system must remain reliable even when it isn't.

That's the foundation.

---

# "What Do I Want From the LLM?" — The Better Question

Backend engineers often ask:

> "What type of data do I want from the LLM?"

A better question is:

> "What contract must this LLM satisfy so the rest of my system remains deterministic?"

You don't want "text."

You want:

* A validated object
* With known fields
* With known types
* That satisfies domain rules
* Or fails safely

That's the difference between a demo and a production system.

---

# Keep It Simple: One Call, One Contract

You don't need multi-agent orchestration to start.

You need:

1. A clear schema
2. A validation layer
3. A retry mechanism
4. Clean boundaries

That's it.

If you get those four right, you can scale later.

---

# The Evolution of Reliability (Even If You Use Frameworks)

Even if you plan to use CrewAI, or any higher-level orchestration framework, understanding this progression is essential.

Because every framework eventually relies on these same principles underneath.

Let's walk through them.

---

# Stage 1: Prompt Formatting (The Naïve Approach)

You write:

> "Return the answer in JSON with fields: title, summary, confidence."

It works.

Until:

* The model adds commentary.
* It changes a key name.
* It returns `"confidence": "high"` instead of `0.92`.
* A prompt injection overrides your format.

Architecture:

```text
User Input
    ↓
Prompt with formatting instructions
    ↓
LLM
    ↓
Text (hopefully JSON)
    ↓
Manual parsing
```

In code, you end up hoping for the best and parsing blindly. When the model returns a string like `"high"` for confidence, your code that expects a float breaks:

```python
prompt = "Return JSON with fields: title, summary, confidence (number 0-1)."
response = llm.invoke(prompt)  # e.g. '{"title": "X", "summary": "Y", "confidence": "high"}'
data = json.loads(response)    # valid JSON, but...
score = data["confidence"] * 0.5  # TypeError: can't multiply sequence by non-int
```

This is equivalent to building an API without schema validation.

It's fragile.

Simple? Yes.
Reliable? No.

---

# Stage 2: JSON Mode (Syntactic Guarantees)

Now you enforce JSON output at the API level.

Good — you guarantee valid JSON.

But not correct structure.

It's like saying:

> "Requests must be JSON."

But not validating the shape.

You get valid JSON from the API, but the *shape* is still unchecked — wrong keys or types still slip through:

```python
# API guarantees valid JSON only
response = client.chat.completions.create(
    model="gpt-4",
    messages=[{"role": "user", "content": prompt}],
    response_format={"type": "json_object"}  # syntax only, not schema
)
data = json.loads(response.choices[0].message.content)  # no guarantee on keys or types
```

Better.

Still incomplete.

---

# Stage 3: Function Calling (Structural Contracts)

Now you define tools with explicit parameters.

The model must return those argument keys.

This introduces structure.

It's similar to defining:

* DTOs
* OpenAPI contracts
* Interface boundaries

But you still must validate:

* Types
* Domain constraints
* Missing values

Example tool schema: the model is constrained to return these keys, but you still need to validate types and domain rules yourself:

```json
{
  "name": "classify_content",
  "parameters": {
    "type": "object",
    "properties": {
      "title": { "type": "string" },
      "summary": { "type": "string" },
      "confidence": { "type": "number" }
    },
    "required": ["title", "summary", "confidence"]
  }
}
```

It's structured, but not fully enforced.

---

# Stage 4: Schema-First (Instructor + Pydantic)

Now we're in backend territory.

You define a schema as code.

Validation is automatic.

Retries happen when validation fails.

Architecture:

```text
User Input
    ↓
LLM
    ↓
Validation Layer (Schema Enforcement)
    ↓ (retry if invalid)
Typed Object
    ↓
Business Logic
```

Define the contract as a Pydantic model and validate before trusting the result; retry when validation fails:

```python
from pydantic import BaseModel, ValidationError

class ClassificationResult(BaseModel):
    title: str
    summary: str
    confidence: float  # 0.0 to 1.0

# After getting raw LLM output (e.g. via Instructor or manual parse):
for attempt in range(3):
    raw = llm.invoke(prompt)
    try:
        result = ClassificationResult.model_validate_json(raw)
        break
    except ValidationError:
        if attempt == 2:
            raise
        continue
# result is typed and validated; safe for domain logic
```

This is the simplest reliable system.

No agents.
No orchestration graphs.
No cognitive architectures.

Just:

LLM → Schema → Domain

Clean. Deterministic. Maintainable.

---

# Where Frameworks Like CrewAI Fit

Frameworks like CrewAI are excellent when:

* You need multi-step reasoning
* You need role-based agents
* You need complex tool orchestration
* You want rapid experimentation

They help manage complexity.

But here's the key:

Even in those systems, each agent call must still be reliable.

If you don't understand:

* Structured outputs
* Validation layers
* Contracts
* Retry mechanisms

You're stacking complexity on top of instability.

It's like building microservices before understanding REST contracts.

Frameworks amplify power.

They also amplify fragility if fundamentals aren't solid.

---

# The Backend Engineer's Minimal LLM Stack

If your goal is simplicity with production safety, here's the minimal pattern:

### 1. Define a Schema First

Contract-first design.

Before writing a prompt, define what your system expects.

### 2. Wrap LLM Calls Behind an Interface

Don't scatter LLM calls across the codebase.

Create a client:

```python
from pydantic import BaseModel

class ClassificationResult(BaseModel):
    title: str
    summary: str
    confidence: float

class ContentClassifier:
    """Single entry point for classification. Validates before return."""

    def classify(self, text: str) -> ClassificationResult:
        raw = self._call_llm(text)
        return ClassificationResult.model_validate_json(raw)  # validate at boundary
```

Clean boundary.

Testable.

Replaceable.

### 3. Validate Before Domain Logic

Never let raw LLM output touch:

* Databases
* External APIs
* Billing logic
* State transitions

Validation first.

Always.

### 4. Implement Retries

LLMs fail probabilistically.

Retries are not optional.

They are part of the contract.

```python
from tenacity import retry, stop_after_attempt, retry_if_exception_type
from pydantic import ValidationError

@retry(stop=stop_after_attempt(3), retry=retry_if_exception_type(ValidationError))
def classify_with_retry(text: str) -> ClassificationResult:
    raw = llm.invoke(prompt)
    return ClassificationResult.model_validate_json(raw)
```

---

# Determinism Is an Architectural Property

You cannot make an LLM deterministic.

But you can make your system deterministic around it.

That means:

* Strict schemas
* Explicit validation
* Observability
* Failure handling
* Clean separation of concerns

The LLM becomes just another adapter layer.

Not your core.

---

# A Simple Mental Model

Think of the LLM like this:

It's an unreliable upstream dependency.

And as backend engineers, we already know how to handle those.

We use:

* Timeouts
* Retries
* Circuit breakers
* Validation layers
* Contracts
* Logging
* Monitoring

LLM integration is not a new discipline.

It's distributed systems engineering applied to probabilistic models.

---

# You Don't Have to Write Everything Yourself

You might never implement raw schema validation yourself.

You might use:

* Instructor
* Framework abstractions
* Agent systems
* Managed AI platforms

That's fine.

But understanding:

* Why schema enforcement matters
* Why prompt formatting isn't enough
* Why retries are essential
* Why clean boundaries protect your system

That knowledge changes how you design systems.

Even when you delegate implementation to a framework.

---

# The Simplest Reliable Path

If I had to reduce everything to one sentence for backend engineers:

> Use LLMs as simple, schema-validated functions — not autonomous agents — until your use case genuinely requires more.

Start with:

One call.
One schema.
One validation boundary.

You can always add complexity later.

It's much harder to remove it.

---

# Final Thought

There is nothing wrong with advanced orchestration frameworks.

But reliability comes from discipline, not abstraction.

Before building AI agents…

Make sure you can safely call an LLM once.

And trust the result.

That's where real production AI begins.
