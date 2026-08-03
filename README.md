# BarSnaps — AI Drink Recognition & Recipe Generator

Snap a photo of a bottle on your bar cart, and get back an original cocktail recipe built around what you actually have. Live at [barsnaps.ai](https://barsnaps.ai).

This repo is an extracted piece of the full BarSnaps app — just the AI recognition + recipe generation feature — pulled out to show how it's built.

---

## What It Does

1. User takes or uploads a photo of a bottle (or their home bar setup)
2. The image is analyzed by a vision-capable AI model, which identifies the specific spirit
3. The AI generates an original cocktail recipe built around that spirit — ingredients, instructions, garnish, glassware, all included
4. The recipe is saved and shown to the user

## Architecture

\`\`\`mermaid
flowchart LR
    A[User captures/uploads photo] --> B[Image uploaded to storage]
    B --> C[Frontend calls generateRecipe function]
    C --> D[AI Vision Model identifies spirit]
    D --> E[AI generates structured recipe JSON]
    E --> F[Recipe saved to database]
    F --> G[Recipe displayed to user]
\`\`\`

**Two pieces, split by responsibility:**
- **Frontend** (`frontend-camera-capture.tsx`) — handles the camera/upload UI, image capture, and calling the backend
- **Backend** (`generateRecipe-function.ts`) — a Supabase Edge Function that does the actual AI work: image analysis, prompt construction, and structured recipe generation

Splitting it this way keeps the AI prompt and any API keys off the frontend entirely — the browser never sees or touches the actual AI call, it just asks the backend to do it and gets a result back.

## The Interesting Part: Prompt Design

The AI doesn't just "look at a photo and make something up." The system prompt enforces a strict pipeline:

1. **Identify first, create second** — the model is required to name the specific bottle/spirit it sees before it's allowed to generate anything, with an explicit brand-to-spirit mapping (e.g. "Don Julio, Patron → tequila") so it can't hand-wave the identification step
2. **The bottle is non-negotiable** — the prompt repeats, in multiple places, that whatever spirit is visible *must* be the base of the recipe. Vision models can drift toward generic answers without this kind of repeated constraint
3. **Two complexity modes** — "Classic" mode restricts the AI to common grocery-store ingredients only (explicitly blocking things like obscure liqueurs or molecular gastronomy techniques), while "Mixologist" mode allows more creative ingredients but still keeps everything achievable at home. This is enforced through detailed ingredient allow/deny lists in the prompt itself, not just a vague instruction
4. **Structured output via tool calling** — rather than asking the AI to return JSON and hoping it's formatted correctly, the request uses function/tool calling to force a strict schema. This eliminates the common failure mode of an AI wrapping its JSON in explanation text or markdown formatting

## Tech Stack

- **Supabase Edge Functions** (Deno runtime) for the backend AI logic
- **Vision-capable LLM** for image analysis and recipe generation, called via an AI gateway
- **Zod** for request validation, so malformed requests fail fast with a clear error instead of silently breaking
- **Supabase Storage + Postgres** for image hosting and recipe persistence

## Notable Engineering Details

- Requests are validated against a strict schema before any AI call is made — invalid input never reaches the (costly) AI step
- Duplicate recipe names are detected and automatically trigger a regeneration request, so users don't get repeat results
- Errors are logged internally with a unique error ID but returned to the client as a generic, user-friendly message — internal details never leak into the response
- Reference recipes are pulled from a database and passed to the AI for inspiration/technique grounding, explicitly instructed not to be copied — grounding creativity without limiting it

---

*This is an extracted feature from a live production app. Some project-specific details have been generalized.*
