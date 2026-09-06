# Fogo Vivo — Conversational Ordering for a Multi-Concept Restaurant

A working prototype of an AI-powered ordering flow for a restaurant that runs three concepts under one roof — a wood-fired pizzeria, a burger counter, and a full-service kitchen — through a single QR code per table.

**Live demo:** _add your GitHub Pages link here_
**Stack:** vanilla HTML/CSS/JS, Anthropic Messages API, no backend, no build step

---

## The problem

Restaurants that combine multiple concepts (pizza, burgers, full plates) usually force one of two bad experiences on the customer: a single overloaded paper menu that's hard to scan, or three separate menus and three separate waits for a waiter. Table turnover suffers, and staff spend most of their time answering the same questions ("does this come with rice?", "is there a vegetarian option?") instead of running the kitchen.

Fogo Vivo replaces the paper menu and the first waiter interaction with a conversational agent, reachable by scanning a QR code at the table. The agent knows the full menu, answers questions in natural language, and builds the order live, item by item, inside the conversation — no app to install, no separate ordering screen to learn.

## Architecture

The whole prototype is a single static HTML file. There is intentionally no backend:

```
Browser
  ├─ Static menu data (JS array, source of truth for prices/descriptions)
  ├─ Chat UI (vanilla JS, no framework)
  └─ fetch() → Anthropic Messages API (claude-sonnet-4-6)
        - system prompt: full menu + persona + response contract
        - messages: full conversation history sent on every turn
```

Every user message triggers a request that includes the entire conversation so far — the model has no memory of its own, so state is reconstructed on each call, matching how the Anthropic Messages API is designed to be used (see "Context window management" in Anthropic's own API guidance).

### Prompt design

The system prompt does three jobs at once:

1. **Grounding** — it embeds the actual menu (name, category, description, price) so the model never invents a dish that doesn't exist or quotes a wrong price.
2. **Persona** — warm, direct, in Brazilian Portuguese, "like a good waiter," which keeps answers short instead of dumping the whole menu on every reply.
3. **A response contract for state** — see below.

### Tracking the cart without function calling

The interesting engineering problem here isn't the chat — it's getting structured state (the cart) out of a model that's producing free-text hospitality-toned replies, without adding a second round-trip or a tool-calling schema for a prototype this size.

The solution: the system prompt instructs the model to append a single hidden line to any reply that changes the cart:

```
<!--CART:[{"item":"Margherita","qty":1,"price":42}]-->
```

The client-side JS extracts that line with a regex, parses it as JSON, updates the cart UI, and strips it from the text before rendering — so the customer never sees it. The model is instructed to always emit the *complete* current cart, not a diff, which avoids state drift between turns since the source of truth resets from the model's own summary each time rather than being incrementally patched on the client.

This is a deliberate trade-off: for a production system, this would move to Claude's native tool use (a `update_cart` tool with a JSON schema) so the output is validated rather than regex-matched. The comment-based approach was fast to prototype and makes the contract inspectable directly in the system prompt, which was the right call for a first working version.

### Failure handling

If the API call fails (network issue, rate limit), the UI shows a plain-language fallback message rather than an error, and the pending "typing…" bubble is cleared so the interface doesn't appear stuck.

## What's deliberately out of scope (v1)

- **Payment** — the flow ends at "send to kitchen," not checkout.
- **Multi-table/session persistence** — cart state lives in memory for the current browser tab only.
- **Menu availability logic** (86'd items, kitchen load) — the menu is static.
- **Real QR routing** — the prototype simulates "scanning" with a button; a production version would encode a table ID in the QR URL and pass it to the agent as context.

## What I'd build next

- Replace the comment-based cart protocol with native tool use / structured output.
- Add per-table session IDs so the QR code actually differentiates tables.
- Add a kitchen-facing view (a second static page reading from the same order state) to close the loop from "order placed" to "order fulfilled."
- Add automated tests around the cart-parsing regex, since it's the one piece of client logic with no schema validation today.

## Running it locally

No build step. Open `fogo-vivo.html` in a browser. The Anthropic API call requires the environment it was prototyped in (Claude's Artifacts runtime) to supply credentials; to run it standalone, point the `fetch` call at your own backend proxy that holds the API key server-side — never ship an API key in client-side JS in a real deployment.

---

*Built as a hands-on exploration of applied LLM engineering: prompt-as-contract design, state extraction from free-text model output, and designing a conversational UI around a real operational workflow rather than a generic chatbot.*
