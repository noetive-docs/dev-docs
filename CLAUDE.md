Developer documentation from companies like Stripe, Amazon Web Services, Vercel, and Anthropic follows a fairly consistent style. It is written for engineers who want to implement something quickly, not study a system in the abstract.

Below are the main characteristics.

## 1. Task-first structure

Good documentation starts with **what the developer is trying to do**, not with the system design.

Typical flow:

1. Short explanation of the capability
2. Minimal working example
3. Step-by-step implementation
4. Reference details

Example pattern:

```
Create a payment
↓
Copy-paste request example
↓
Explain required parameters
↓
Link to API reference
```

The developer should be able to run something in minutes.

Stripe’s documentation is the clearest example: a working API call appears before any theory.

## 2. Example-driven writing

The core teaching tool is **real code**.

Examples usually include:

* curl request
* one or two SDK languages
* response payload
* error cases

The examples are complete enough to run. They are not fragments that require interpretation.

A typical pattern:

```
POST /v1/customers

curl https://api.stripe.com/v1/customers \
  -u sk_test_...: \
  -d email="user@example.com"
```

Explanation follows the example rather than preceding it.

## 3. Short paragraphs, dense information

Sentences are direct and technical.

Common traits:

* little narrative text
* minimal adjectives
* precise nouns and verbs
* short sections

Typical section length: **3–6 sentences**.

If an explanation runs longer than that, it usually becomes its own page.

## 4. Clear separation of guides vs reference

Strong documentation splits content into two categories.

### Guides (tutorials)

Goal: accomplish a task.

Examples:

* "Create a webhook"
* "Deploy a Next.js app"
* "Stream responses from the API"

Characteristics:

* step-by-step
* contextual explanation
* runnable examples

### Reference

Goal: look up exact behavior.

Examples:

* API endpoint parameters
* response schemas
* error codes
* rate limits

Characteristics:

* exhaustive
* structured
* minimal narrative

AWS documentation is mostly reference; Stripe balances both.

## 5. Strong information hierarchy

Good documentation is easy to scan.

Typical elements:

* short headings
* tables for parameters
* code blocks
* callout boxes for warnings
* inline links to deeper material

A developer should locate the needed information within seconds.

## 6. Explicit constraints and edge cases

High-quality docs surface operational realities early.

They state things like:

* rate limits
* retry rules
* idempotency behavior
* pagination mechanics
* timeouts
* error semantics

Stripe does this well. Many APIs hide these details.

## 7. Opinionated defaults

Good documentation reduces decision space.

Instead of listing every possible configuration, it says:

* "Use this endpoint"
* "Use this SDK"
* "Use this header"

Optional complexity appears later.

Vercel documentation often follows this approach.

# What excellent documentation deliberately leaves out

Good documentation omits information that does not help someone ship code.

Common exclusions:

### Internal architecture

Developers rarely need:

* internal service topology
* database structure
* infrastructure layout

Unless it affects the API contract.

### Historical context

No long explanations about:

* why the system was built
* past versions
* design debates

That belongs in blog posts or RFCs.

### Marketing language

You almost never see:

* product positioning
* feature hype
* philosophical framing

Those belong on product pages, not docs.

### Academic explanations

Documentation rarely teaches underlying theory unless required.

Example:

* Stripe explains **how to create a payment intent**, not the theory of payment processing.

# When documentation becomes too verbose

Documentation becomes ineffective when it increases reading time without increasing implementation clarity.

Common symptoms:

### 1. Explanations before examples

Bad pattern:

```
Paragraphs explaining the system
↓
More explanation
↓
Finally a code example
```

Developers want the example immediately.

### 2. Overexplaining obvious concepts

Example of verbosity:

> "A webhook is a way for your application to receive HTTP notifications when events occur."

Developers already know HTTP callbacks.

Better:

> "Configure a webhook endpoint to receive event notifications."

### 3. Excessive cross-referencing

If every paragraph contains multiple links to other pages, the reader cannot follow the flow.

Documentation should be **self-sufficient for the task at hand**.

### 4. Giant pages

Pages longer than ~1500–2000 words usually indicate poor structure.

Good docs split topics into:

* concept
* guide
* reference

### 5. Explaining the API twice

A common mistake:

1. narrative explanation
2. parameter table
3. example request
4. another explanation

One clear explanation plus example is enough.

# A useful rule of thumb

For developer documentation:

**The reader should succeed faster by reading it than by reading the source code.**

If a developer skims the page and still cannot:

* construct the request
* understand required parameters
* handle errors

then the documentation is either incomplete or buried under unnecessary text.

If useful, I can also break down **the exact documentation architecture Stripe uses** (their page layout, section order, and information density). Many teams copy that structure when designing API docs.


## Mintlify llms.txt

https://www.mintlify.com/docs/llms.txt
