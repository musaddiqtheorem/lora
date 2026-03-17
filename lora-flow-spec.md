# LoRA Flow Manager — Technical Specification

## Overview

A Next.js dashboard for managing the end-to-end LoRA training workflow on fal.ai. Users create "LoRA Flows" that guide them from defining a vision through image upload, AI-assisted captioning, training execution, and finally testing the trained LoRA.

**Stack:** Next.js 14 (App Router), Supabase (Auth + Postgres + Storage), Tailwind CSS, Vercel deployment, fal.ai API, Anthropic API.

---

## 1. Database Schema (Supabase / Postgres)

### `lora_flows`

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid (PK, default gen_random_uuid()) | |
| `user_id` | uuid (FK → auth.users) | RLS: users see only their own flows |
| `vision` | text | Purpose statement, e.g. "Product LoRA for UGC campaign" |
| `theme` | text | Overarching concept, e.g. "Clean product shots of X from multiple angles" |
| `trigger_word` | text | e.g. "MYBRAND_serum" |
| `status` | text | One of: `draft`, `uploading`, `captioning`, `ready`, `training`, `complete`, `failed` |
| `ai_guidance` | jsonb | Claude's suggestions (image count, shot types, tips) |
| `training_config` | jsonb | `{ steps: 1000, learning_rate: 0.0001, batch_size: 1 }` |
| `fal_job_id` | text | fal.ai async job ID, set when training starts |
| `lora_url` | text | URL to .safetensors file, set on completion |
| `training_started_at` | timestamptz | |
| `training_completed_at` | timestamptz | |
| `estimated_duration_sec` | int | Calculated: steps × 2 (rough seconds per step) |
| `error_message` | text | Set if status = failed |
| `created_at` | timestamptz (default now()) | |
| `updated_at` | timestamptz (default now()) | |

### `flow_images`

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid (PK, default gen_random_uuid()) | |
| `flow_id` | uuid (FK → lora_flows.id, ON DELETE CASCADE) | |
| `storage_path` | text | Path in Supabase Storage bucket, e.g. `flows/{flow_id}/{filename}` |
| `public_url` | text | Supabase Storage public URL (needed for fal.ai) |
| `original_filename` | text | e.g. "product_front_angle.jpg" |
| `caption` | text | Manual or AI-generated caption including trigger word |
| `ai_draft_caption` | text | Claude's suggested caption (preserved for reference) |
| `sort_order` | int | Display ordering |
| `created_at` | timestamptz (default now()) | |

### `lora_tests`

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid (PK) | |
| `flow_id` | uuid (FK → lora_flows.id) | |
| `prompt` | text | Test generation prompt |
| `result_url` | text | Generated image URL from fal.ai |
| `model_used` | text | e.g. "fal-ai/flux-2/lora" |
| `created_at` | timestamptz | |

### Supabase Storage

- **Bucket:** `flow-images` (public, so fal.ai can access URLs directly)
- **Path pattern:** `{user_id}/{flow_id}/{filename}`

### Row Level Security

```sql
-- lora_flows: users see only their own
CREATE POLICY "Users manage own flows" ON lora_flows
  FOR ALL USING (auth.uid() = user_id);

-- flow_images: access through flow ownership
CREATE POLICY "Users manage own flow images" ON flow_images
  FOR ALL USING (
    flow_id IN (SELECT id FROM lora_flows WHERE user_id = auth.uid())
  );

-- lora_tests: access through flow ownership
CREATE POLICY "Users manage own tests" ON lora_tests
  FOR ALL USING (
    flow_id IN (SELECT id FROM lora_flows WHERE user_id = auth.uid())
  );
```

---

## 2. Application Structure

```
src/
├── app/
│   ├── layout.tsx                    # Root layout with auth provider
│   ├── page.tsx                      # Dashboard: list of all LoRA Flows
│   ├── login/
│   │   └── page.tsx                  # Auth page (Supabase Auth UI)
│   ├── flows/
│   │   ├── new/
│   │   │   └── page.tsx              # Step 1: Create flow (vision + theme)
│   │   └── [id]/
│   │       ├── page.tsx              # Flow detail / hub page (routes to current step)
│   │       ├── upload/
│   │       │   └── page.tsx          # Step 2: Upload images
│   │       ├── caption/
│   │       │   └── page.tsx          # Step 3: Review & edit captions
│   │       ├── train/
│   │       │   └── page.tsx          # Step 4: Configure & launch training
│   │       └── test/
│   │           └── page.tsx          # Step 5: Test the trained LoRA
│   └── api/
│       ├── flows/
│       │   ├── route.ts              # POST: create flow
│       │   └── [id]/
│       │       └── route.ts          # GET, PATCH, DELETE flow
│       ├── ai/
│       │   ├── guidance/
│       │   │   └── route.ts          # POST: Claude generates training guidance
│       │   └── captions/
│       │       └── route.ts          # POST: Claude generates draft captions
│       ├── training/
│       │   ├── start/
│       │   │   └── route.ts          # POST: trigger fal.ai training
│       │   ├── status/
│       │   │   └── route.ts          # GET: poll fal.ai job status
│       │   └── webhook/
│       │       └── route.ts          # POST: fal.ai completion callback
│       └── test/
│           └── generate/
│               └── route.ts          # POST: generate test image with LoRA
├── components/
│   ├── FlowCard.tsx                  # Flow summary card for dashboard
│   ├── FlowStepper.tsx              # Progress indicator (5 steps)
│   ├── ImageUploader.tsx             # Drag-and-drop multi-image upload
│   ├── CaptionEditor.tsx            # Single image card with caption field
│   ├── CaptionGrid.tsx              # Grid of all CaptionEditor cards
│   ├── GuidancePanel.tsx            # Claude's AI suggestions display
│   ├── TrainingConfig.tsx           # Training parameter controls
│   ├── TrainingProgress.tsx         # Live progress with time estimate
│   ├── TestPlayground.tsx           # Prompt + generate with trained LoRA
│   └── ui/                          # Shared primitives (Button, Input, etc.)
├── lib/
│   ├── supabase/
│   │   ├── client.ts                # Browser Supabase client
│   │   ├── server.ts                # Server-side Supabase client
│   │   └── middleware.ts            # Auth middleware
│   ├── fal.ts                       # fal.ai API helpers
│   ├── anthropic.ts                 # Anthropic API helpers
│   └── constants.ts                 # Status enums, default configs
└── types/
    └── index.ts                     # TypeScript interfaces
```

---

## 3. API Route Specifications

### POST `/api/ai/guidance`

Called after user enters vision + theme. Sends both to Claude to get training recommendations.

```typescript
// Request body
{
  vision: string,   // "Product LoRA for UGC campaign"
  theme: string     // "Clean product shots of X serum from multiple angles"
}

// Claude system prompt (sent to Anthropic API)
`You are an expert in training LoRA models for FLUX image generation.
Given a user's vision and theme for their LoRA, provide specific guidance:

1. recommended_image_count: number (typically 15-50)
2. shot_types: string[] — specific types of photos they should include
   (e.g., "front-facing with label visible", "45-degree angle showing cap detail",
   "product in-hand for scale reference", "close-up of texture/finish")
3. suggested_trigger_word: string — a unique, specific trigger word
4. background_tips: string — advice on background variety
5. lighting_tips: string — advice on lighting variation
6. common_mistakes: string[] — things to avoid
7. estimated_training_steps: number — recommended step count

Respond in JSON only.`

// Response: Claude's JSON guidance, stored in lora_flows.ai_guidance
```

### POST `/api/ai/captions`

Called after images are uploaded. Sends each image to Claude for draft caption generation.

```typescript
// Request body
{
  flow_id: string,
  trigger_word: string,
  theme: string,
  images: Array<{ id: string, public_url: string }>
}

// Claude system prompt
`You are writing training captions for a LoRA fine-tuning dataset.
The trigger word is: "${trigger_word}"
The theme is: "${theme}"

For each image, write a detailed caption that:
- Starts with "a photo of ${trigger_word}"
- Describes the subject precisely (shape, color, material, text on labels)
- Describes the angle/perspective
- Describes lighting conditions
- Describes the background/setting
- Uses natural language, not keywords
- Is 1-3 sentences long

Respond as JSON: { "captions": [{ "image_id": "...", "caption": "..." }] }`

// Send images as base64 or URLs in Claude's vision input
// Response: array of { image_id, caption } → saved to flow_images.ai_draft_caption
```

### POST `/api/training/start`

Triggers fal.ai LoRA training.

```typescript
// Request body
{ flow_id: string }

// Server-side logic:
// 1. Fetch flow + all flow_images from Supabase
// 2. Build fal.ai training request
// 3. Submit async job, store job ID

// fal.ai API call
const result = await fal.subscribe("fal-ai/flux-2-trainer", {
  input: {
    images_data_url: null,  // OR zip URL if using zip approach
    steps: flow.training_config.steps,       // default 1000
    learning_rate: flow.training_config.learning_rate,  // default 0.0001
    trigger_word: flow.trigger_word,
    // Individual image + caption pairs:
    images: flow_images.map(img => ({
      url: img.public_url,
      caption: img.caption
    }))
  },
  webhookUrl: `${process.env.NEXT_PUBLIC_APP_URL}/api/training/webhook`
});

// Store: fal_job_id, training_started_at, estimated_duration_sec
// Update status → "training"
```

### POST `/api/training/webhook`

Receives completion callback from fal.ai.

```typescript
// fal.ai sends:
{
  request_id: string,
  status: "OK" | "ERROR",
  payload: {
    diffusers_lora_file: { url: string },
    // ... other metadata
  }
}

// On success:
// Update flow: lora_url = payload.diffusers_lora_file.url
// Update status → "complete"
// Update training_completed_at

// On failure:
// Update status → "failed"
// Store error_message
```

### GET `/api/training/status`

Polling endpoint (backup for webhook). Checks fal.ai job status.

```typescript
// Query: ?flow_id=xxx
// Server calls fal.ai status endpoint with stored fal_job_id
// Returns: { status, progress_percentage, estimated_remaining_sec }
```

### POST `/api/test/generate`

Generate a test image using the trained LoRA.

```typescript
// Request body
{
  flow_id: string,
  prompt: string  // Must include trigger word
}

// Server-side:
const result = await fal.subscribe("fal-ai/flux-2/lora", {
  input: {
    prompt: body.prompt,
    loras: [{
      path: flow.lora_url,
      scale: 0.8
    }],
    image_size: "landscape_16_9",
    num_images: 1
  }
});

// Store result in lora_tests table
// Return generated image URL
```

---

## 4. Page-by-Page UI Specification

### Dashboard (`/`)

The home page shows all LoRA Flows as cards in a grid layout. Each card shows:
- Flow name (derived from theme, truncated)
- Vision text (subtitle)
- Status badge (color-coded: draft=gray, uploading=blue, captioning=yellow, ready=green, training=orange with pulse animation, complete=green with checkmark, failed=red)
- Image count (e.g. "24 images")
- Created date
- If complete: thumbnail of a test generation

Primary action: "+ New LoRA Flow" button → navigates to `/flows/new`

### Step 1: Create Flow (`/flows/new`)

Two-panel layout:
- **Left panel:** Form with Vision (textarea) and Theme (textarea) fields. Below: trigger word input (auto-suggested by Claude but editable). "Get AI Guidance" button.
- **Right panel:** GuidancePanel component. Initially empty/placeholder. After Claude responds, displays the structured guidance: recommended image count (prominent number), shot type checklist, lighting tips, background tips, common mistakes to avoid. This panel should feel like a helpful advisor, not a wall of text.

On submit: creates the flow record, navigates to `/flows/[id]/upload`

### Step 2: Upload Images (`/flows/[id]/upload`)

- **Top:** FlowStepper showing progress (step 2 of 5 highlighted)
- **Top-right:** Guidance summary (collapsed by default, expandable) reminding user of recommended shot types and count
- **Main area:** Drag-and-drop zone (ImageUploader). Accepts .jpg, .png, .webp. Multi-file upload. Shows upload progress per file.
- **Below drop zone:** Grid of uploaded image thumbnails with filename, file size, and a remove button. Running count vs. recommended count (e.g. "12 / 20 images uploaded").
- **Bottom:** "Continue to Captioning" button (enabled once minimum images uploaded, e.g. 10+)

Images upload to Supabase Storage on drop/select. Each creates a `flow_images` row immediately.

### Step 3: Caption Images (`/flows/[id]/caption`)

- **Top:** FlowStepper (step 3 highlighted)
- **Action bar:** "Generate AI Captions" button → calls `/api/ai/captions`, shows loading state per card
- **Main area:** CaptionGrid — scrollable grid of CaptionEditor cards. Each card:
  - Image thumbnail (left or top)
  - Caption textarea (right or bottom)
  - If AI caption was generated: subtle indicator showing it's AI-drafted, with a "Reset to AI draft" option if user has edited
  - Character count
  - Trigger word presence indicator (green check if trigger word is in caption, red warning if missing)
- **Bottom:** Summary stats (X/Y images captioned, all trigger words present ✓). "Ready for Training" button (enabled when all images have captions containing the trigger word).

### Step 4: Training (`/flows/[id]/train`)

- **Top:** FlowStepper (step 4 highlighted)
- **Pre-training state:**
  - Training configuration panel (TrainingConfig):
    - Steps slider (500–2000, default 1000) with cost estimate updating live
    - Learning rate (default 1e-4, with "Advanced" toggle to change)
    - Summary: "~1,000 steps • ~30 min • ~$8.00 estimated cost"
  - Quick summary of the flow: image count, theme, trigger word
  - "Start Training" button (prominent, with confirmation modal)
- **Training state (after start):**
  - TrainingProgress component:
    - Large circular or linear progress indicator
    - Elapsed time / estimated total time
    - Status text ("Training in progress...", "Optimizing weights...", etc.)
    - Animated state to indicate activity
  - Poll `/api/training/status` every 30 seconds
  - Webhook updates status in real-time via Supabase Realtime subscription
- **Completion state:**
  - Success message with LoRA URL displayed (copyable)
  - "Test Your LoRA →" button navigating to test page
  - Training metadata: actual duration, steps completed, final loss (if available from fal.ai response)
- **Failure state:**
  - Error message display
  - "Retry Training" button

### Step 5: Test (`/flows/[id]/test`)

- **Top:** FlowStepper (step 5 highlighted, all green)
- **Left panel:** Prompt input area
  - Textarea for the generation prompt
  - Reminder text: "Include your trigger word: `{trigger_word}`"
  - Quick prompt suggestions (pre-filled buttons):
    - "{trigger_word} on a marble countertop, soft morning light"
    - "{trigger_word} held in a hand against a blurred outdoor background"
    - "{trigger_word} in a flat lay arrangement with complementary objects"
  - "Generate" button
  - Model selector (default: fal-ai/flux-2/lora, options: fal-ai/flux-lora for FLUX.1)
  - LoRA scale slider (0.5–1.0, default 0.8)
- **Right panel:** Generated image display area
  - Loading state while generating
  - Generated image with download button
  - History of previous test generations (scrollable, newest first)

---

## 5. Environment Variables

```env
# Supabase
NEXT_PUBLIC_SUPABASE_URL=https://xxx.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJ...
SUPABASE_SERVICE_ROLE_KEY=eyJ...  # Server-side only, for webhook handler

# fal.ai
FAL_KEY=fal_...  # Server-side only

# Anthropic
ANTHROPIC_API_KEY=sk-ant-...  # Server-side only

# App
NEXT_PUBLIC_APP_URL=https://your-app.vercel.app  # For webhook URL construction
```

---

## 6. Key Dependencies

```json
{
  "dependencies": {
    "next": "^14.2",
    "@supabase/supabase-js": "^2.45",
    "@supabase/ssr": "^0.5",
    "@fal-ai/client": "^1.2",
    "@anthropic-ai/sdk": "^0.30",
    "tailwindcss": "^3.4",
    "lucide-react": "^0.400",
    "react-dropzone": "^14.2",
    "zustand": "^4.5"
  }
}
```

---

## 7. fal.ai Integration Notes

### Training endpoint

```typescript
import { fal } from "@fal-ai/client";

fal.config({ credentials: process.env.FAL_KEY });

// Option A: Individual images with captions (preferred)
const result = await fal.subscribe("fal-ai/flux-2-trainer", {
  input: {
    images: images.map(img => ({
      url: img.public_url,
      caption: img.caption
    })),
    trigger_word: "MYBRAND_serum",
    steps: 1000,
    learning_rate: 0.0001,
    is_style: false,  // true for style LoRAs, false for subject LoRAs
    create_masks: true  // auto-mask subject for better training
  },
  webhookUrl: `${process.env.NEXT_PUBLIC_APP_URL}/api/training/webhook`,
  onQueueUpdate: (update) => {
    // Can be used for progress updates if polling
    console.log(update.status);
  }
});

// Result contains:
// result.diffusers_lora_file.url → the LoRA weights URL
// result.config_file.url → training config for reference
```

### Generation with trained LoRA

```typescript
const result = await fal.subscribe("fal-ai/flux-2/lora", {
  input: {
    prompt: "a photo of MYBRAND_serum on a marble countertop",
    loras: [{
      path: "https://fal.ai/files/xxx/lora.safetensors",  // stored lora_url
      scale: 0.8  // 0.5-1.0, higher = stronger LoRA influence
    }],
    image_size: "landscape_16_9",
    num_inference_steps: 28,
    guidance_scale: 3.5,
    num_images: 1,
    enable_safety_checker: false  // product photos don't need this
  }
});

// result.images[0].url → generated image URL
```

### Time estimation formula

```typescript
function estimateTrainingTime(steps: number, imageCount: number): number {
  // Base: ~2 seconds per step
  // Additional: ~0.05s per step per image over 20
  const baseSecondsPerStep = 2;
  const imageOverhead = Math.max(0, imageCount - 20) * 0.05;
  const totalSeconds = steps * (baseSecondsPerStep + imageOverhead);
  return Math.ceil(totalSeconds);
}

// 1000 steps, 25 images → ~2,250 seconds → ~37.5 minutes
```

### Cost estimation formula

```typescript
function estimateTrainingCost(steps: number): number {
  // fal.ai charges ~$0.008 per step for FLUX.2 trainer
  return steps * 0.008;
}

// 1000 steps → $8.00
```

---

## 8. Anthropic API Integration Notes

### Captioning with vision

```typescript
import Anthropic from "@anthropic-ai/sdk";

const anthropic = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });

// For caption generation, send images as URLs
// Claude can process multiple images in a single request
const response = await anthropic.messages.create({
  model: "claude-sonnet-4-20250514",
  max_tokens: 4096,
  system: `You are writing training captions for a LoRA fine-tuning dataset.
The trigger word is: "${triggerWord}"
The theme is: "${theme}"
...`, // full prompt from API spec above
  messages: [{
    role: "user",
    content: images.flatMap(img => [
      {
        type: "image",
        source: { type: "url", url: img.public_url }
      },
      {
        type: "text",
        text: `Image ID: ${img.id}. Write a training caption for this image.`
      }
    ])
  }]
});
```

### Guidance generation

```typescript
const response = await anthropic.messages.create({
  model: "claude-sonnet-4-20250514",
  max_tokens: 2048,
  system: "You are an expert in training LoRA models...",  // full prompt from API spec
  messages: [{
    role: "user",
    content: `Vision: ${vision}\nTheme: ${theme}`
  }]
});
```

---

## 9. Supabase Realtime (for training progress)

```typescript
// Client-side: subscribe to flow status changes
const channel = supabase
  .channel(`flow-${flowId}`)
  .on(
    "postgres_changes",
    {
      event: "UPDATE",
      schema: "public",
      table: "lora_flows",
      filter: `id=eq.${flowId}`
    },
    (payload) => {
      // Update UI when webhook updates the flow
      if (payload.new.status === "complete") {
        setLoraUrl(payload.new.lora_url);
        setStatus("complete");
      } else if (payload.new.status === "failed") {
        setError(payload.new.error_message);
        setStatus("failed");
      }
    }
  )
  .subscribe();
```

---

## 10. Design Direction

**Aesthetic:** Clean, utilitarian, developer-tool-meets-creative-studio. Think Linear meets Figma — precise spacing, monospace accents for technical values (trigger words, URLs, costs), generous whitespace, subtle depth through shadows not gradients.

**Color system:**
- Background: near-white warm gray (#FAFAF9) with white cards
- Primary accent: deep indigo (#4F46E5) for CTAs and active states
- Status colors: gray (draft), blue (uploading), amber (captioning), emerald (ready/complete), orange (training), red (failed)
- Text: charcoal (#1C1917) primary, muted gray (#78716C) secondary
- Monospace accent: for trigger words, URLs, costs, technical values

**Typography:**
- Headings: DM Sans (bold, clean, slightly geometric)
- Body: DM Sans (regular)
- Monospace accents: JetBrains Mono for trigger words, file names, URLs, costs

**Key UI patterns:**
- Flow cards use subtle left-border color coding by status
- Image upload area: dashed border, drag highlight with indigo tint
- Caption cards: image on left (fixed width), caption textarea fills remaining space
- Training progress: large centered ring/arc with percentage, elapsed/remaining below
- The stepper at the top of flow pages is horizontal, minimal, with numbered circles connected by lines

---

## 11. Deployment Notes

### Vercel Configuration

- Framework preset: Next.js
- Build command: `next build`
- Environment variables: all from Section 5 above
- **Important:** The webhook endpoint (`/api/training/webhook`) must be publicly accessible. Vercel handles this automatically since API routes are serverless functions.

### Supabase Setup Checklist

1. Create project
2. Run SQL for tables (schema from Section 1)
3. Enable RLS on all tables, add policies
4. Create `flow-images` storage bucket (set to public)
5. Add storage policy: authenticated users can upload to their own path prefix
6. Enable Realtime on `lora_flows` table

### Post-Deployment

1. Set `NEXT_PUBLIC_APP_URL` to actual Vercel URL
2. Test webhook endpoint is reachable: `curl -X POST https://your-app.vercel.app/api/training/webhook`
3. Verify Supabase Storage URLs are publicly accessible (fal.ai needs to fetch them)
