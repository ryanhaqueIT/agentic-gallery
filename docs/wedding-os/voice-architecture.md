# Phone Call Integration Architecture — Wedding OS

> **Date**: 2026-02-26 (updated with Amazon Connect decision)
> **Status**: RESEARCH COMPLETE — Architecture defined, ready for implementation planning
> **Prerequisite**: Mail integration (Phases A–I) — DONE
> **Telephony Decision**: Amazon Connect (native Nova Sonic, managed phone numbers, no custom Voice Bridge)
> **Target Market**: Australia (Melbourne) — Connect instance in us-east-1 with AU phone numbers
> **Research Sources**: 6 research agents analyzed Oratio voice infrastructure, Nova Sonic API, OpenClaw extension patterns, competitive landscape, Amazon Connect capabilities, and Australia region availability

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Architecture Overview](#2-architecture-overview)
3. [Two-Tier Voice Architecture](#3-two-tier-voice-architecture)
4. [Nova 2 Sonic Integration](#4-nova-2-sonic-integration)
5. [Amazon Connect Telephony](#5-amazon-connect-telephony)
6. [OpenClaw Integration](#6-openclaw-integration)
7. [Vendor Context Injection](#7-vendor-context-injection)
8. [Agent Tools for Voice](#8-agent-tools-for-voice)
9. [DynamoDB Schema Extensions](#9-dynamodb-schema-extensions)
10. [Frontend Voice Components](#10-frontend-voice-components)
11. [Voice Personality System](#11-voice-personality-system)
12. [Deployment Architecture](#12-deployment-architecture)
13. [Implementation Roadmap](#13-implementation-roadmap)
14. [Competitive Positioning](#14-competitive-positioning)
15. [Cost Analysis](#15-cost-analysis)
16. [Key Reference Material](#16-key-reference-material)

---

## 1. Executive Summary

Wedding OS adds **phone call integration** as a second communication channel alongside email. **Amazon Connect** provides managed phone numbers (including Australian DIDs for Melbourne vendors) with **native Nova 2 Sonic speech-to-speech** — no custom Voice Bridge code needed. Nova Sonic handles the real-time conversation, delegating business logic to the existing **OpenClaw agent** via tool calls. After the call ends, the transcript flows into OpenClaw as an SMS message, creating a persistent record and enabling cross-channel follow-up.

### The Killer Differentiator

No competitor in the wedding AI space shares context between email and voice. The same agent that drafts email responses also answers phone calls — with full knowledge of past email threads, lead context, vendor personality, and pricing. A couple who emailed about October 4 availability will hear the agent reference that email when they call.

### Architecture Decisions

```
1. Speech-to-Speech (Nova 2 Sonic) — NOT cascaded pipeline (ASR + LLM + TTS)
2. Amazon Connect (managed telephony) — NOT Twilio (eliminates custom Voice Bridge)
3. Connect instance in us-east-1 — Nova Sonic native integration (not yet in ap-southeast-2)
```

**Rationale**: Speech-to-speech delivers 100-330ms latency vs 450-1200ms for cascaded pipelines. Nova Sonic understands tone, emotion, and cadence — not just words. It handles barge-in (interruptions) natively. Amazon Connect bundles Nova Sonic into its per-minute rate ($0.038/min all-in) — no separate Bedrock charges, no custom WebSocket bridge, no audio format conversion code.

### Australia/Melbourne Support

Amazon Connect is available in `ap-southeast-2` (Sydney) and supports Australian DID phone numbers. However, Nova Sonic native integration is currently only in `us-east-1` and `us-west-2`. Our approach: **Connect instance in `us-east-1` with Australian phone numbers** — callers dial a local AU number, audio is processed in Virginia. The ~200ms cross-Pacific latency + ~300ms model latency = ~500ms total, well within the natural conversation threshold (<800ms).

---

## 2. Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        WEDDING OS — FULL ARCHITECTURE                       │
│                                                                             │
│  EMAIL CHANNEL (existing)              PHONE CHANNEL (new)                  │
│  ─────────────────────                 ─────────────────────                │
│                                                                             │
│  Customer Email                        Couple calls vendor number           │
│       │                                       │                             │
│       ▼                                       ▼                             │
│  AgenticMail (SSE)                    Amazon Connect (PSTN)                │
│       │                               (managed phone numbers,              │
│       ▼                                native Nova Sonic)                   │
│  Platform API                                 │                             │
│  (FastAPI)                                    ▼                             │
│       │                               Nova 2 Sonic (built into Connect)    │
│       ▼                               Speech-to-speech, no bridge needed   │
│  OpenClaw Gateway                             │                             │
│       │                                       │ ask_agent tool              │
│       ▼                                       ▼                             │
│  Pi Agent                             OpenClaw Gateway ◄──── same agent    │
│  (Nova 2 Lite)                                │                             │
│       │                                       ▼                             │
│       ▼                               Pi Agent (Nova 2 Lite)               │
│  Tools:                                       │                             │
│  - get_lead_context                           ▼                             │
│  - get_thread_history              Same tools + voice-specific:             │
│  - store_draft                      - check_availability                    │
│  - send_email                       - get_packages                          │
│                                     - create_lead_from_call                 │
│                                     - send_sms_followup                     │
│                                                                             │
│  ┌──────────────────────────────────────────────────────────┐              │
│  │                 SHARED DATA LAYER                         │              │
│  │  DynamoDB: vendors, leads, threads, messages, drafts     │              │
│  │  S3: SOUL.md, Principles.md, workspace files             │              │
│  │  NEW: calls, call_transcripts tables                      │              │
│  └──────────────────────────────────────────────────────────┘              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Two-Tier Voice Architecture

This pattern is **proven in production by Oratio** (the scaffold this project is built on). It cleanly separates concerns:

### Tier 1 — Conversation Layer (Nova 2 Sonic)

- Handles ALL speech: STT, TTS, turn-taking, barge-in, emotional tone
- Maintains conversational flow and natural speech patterns
- Configured with voice personality (voice ID, pacing, formality)
- Calls `ask_agent` tool when business logic is needed

### Tier 2 — Business Logic Layer (OpenClaw + Pi Agent)

- Invoked as a tool by Nova Sonic (NOT the other way around)
- Runs existing tools: availability check, pricing, lead creation
- Returns structured data — Nova Sonic converts to speech
- Same agent, same SOUL.md, same Principles.md as email

### Why This Separation Matters

| Problem | How Two-Tier Solves It |
|---------|----------------------|
| Awkward silence during tool execution | Nova Sonic says "Let me check that for you" BEFORE calling tool |
| High latency from cascaded STT+LLM+TTS | Nova Sonic processes audio end-to-end (~300ms) |
| Different models for speech vs reasoning | Each model does what it's best at |
| Context sharing across channels | Same OpenClaw agent handles both email and voice tool calls |

### The `ask_agent` Tool Pattern

Nova Sonic is configured with exactly **one tool** — `ask_agent`:

```json
{
  "toolSpec": {
    "name": "ask_agent",
    "description": "IMPORTANT: This tool queries a specialized AI agent for business data. YOU MUST ALWAYS speak to the caller BEFORE calling this tool. Say something like 'Let me look that up for you' first. Then call the tool. Then respond with the result.",
    "inputSchema": {
      "json": {
        "type": "object",
        "properties": {
          "query": { "type": "string", "description": "The caller's question or request" }
        },
        "required": ["query"]
      }
    }
  }
}
```

The "speak first" instruction is critical — it prevents dead air while the tool executes (~300ms-1.5s).

---

## 4. Nova 2 Sonic Integration

### Model Details

| Parameter | Value |
|-----------|-------|
| **Model ID** | `amazon.nova-2-sonic-v1:0` |
| **API** | `InvokeModelWithBidirectionalStream` (HTTP/2) |
| **Regions** | `us-east-1`, `us-west-2`, `ap-northeast-1`, `eu-north-1` |
| **Max session** | 8 minutes |
| **Context window** | 1 million tokens |
| **Audio input** | LPCM, 16kHz (or 8kHz for telephony), 16-bit, mono, base64 |
| **Audio output** | LPCM, 24kHz, 16-bit, mono, base64 |
| **Tool calling** | Full support mid-conversation |
| **Barge-in** | Native — `interrupted: true` in textOutput event |
| **Turn detection** | LOW, MEDIUM, HIGH sensitivity settings |
| **Temperature** | `0` for tool calls, `0.7` for conversation |

### Voice Options

| Voice ID | Style | Best For |
|----------|-------|----------|
| `tiffany` | Feminine, warm | Default wedding vendor voice |
| `matthew` | Masculine, friendly | Alternative voice option |
| `amy` | Feminine, UK accent | International vendors |

Nova 2 Sonic adds polyglot voices that speak multiple languages naturally.

### Session Lifecycle (Event Protocol)

```
Client                          Nova Sonic (Bedrock)
  │                                    │
  ├─── sessionStart ──────────────────►│  (inference config)
  ├─── promptStart ───────────────────►│  (voice config, tool defs)
  ├─── contentStart (SYSTEM) ─────────►│  (system prompt)
  ├─── textInput ─────────────────────►│  (vendor personality + protocol)
  ├─── contentEnd ────────────────────►│
  ├─── contentStart (AUDIO, USER) ────►│  (begin listening)
  │                                    │
  ├─── audioInput (32ms chunks) ──────►│  (continuous audio stream)
  ├─── audioInput ────────────────────►│
  │                                    │
  │    ◄──── completionStart ──────────┤  (response begins)
  │    ◄──── textOutput (USER) ────────┤  (ASR transcription)
  │    ◄──── textOutput (ASSISTANT) ───┤  (speculative preview)
  │    ◄──── audioOutput (chunks) ─────┤  (speech response)
  │    ◄──── completionEnd ────────────┤  (turn complete)
  │                                    │
  │  ... OR tool needed:               │
  │    ◄──── toolUse ──────────────────┤  (ask_agent call)
  ├─── toolResult ────────────────────►│  (business data)
  │    ◄──── audioOutput (chunks) ─────┤  (speaks the result)
  │                                    │
  ├─── contentEnd ────────────────────►│  (close audio)
  ├─── promptEnd ─────────────────────►│
  ├─── sessionEnd ────────────────────►│  (mandatory cleanup)
  │                                    │
```

### 8-Minute Session Limit — Reconnection Pattern

When session approaches 8 minutes:
1. Close current session properly (`contentEnd → promptEnd → sessionEnd`)
2. Open a new bidirectional stream
3. Replay conversation history via `contentStart(TEXT, USER/ASSISTANT)` events
4. Resume audio streaming

For typical wedding inquiries (2-5 minutes), most calls complete within a single session.

### Strands BidiAgent — Fastest Development Path

```python
from strands.experimental.bidi import BidiAgent, BidiAudioIO
from strands.experimental.bidi.models import BidiNovaSonicModel

model = BidiNovaSonicModel(
    model_id="amazon.nova-2-sonic-v1:0",
    provider_config={"audio": {"voice": "tiffany"}},
    client_config={"region": "us-east-1"},
)

agent = BidiAgent(
    model=model,
    system_prompt=vendor_voice_prompt,  # from SOUL.md + voice personality
    tools=[ask_agent_tool]              # delegates to OpenClaw
)

audio_io = BidiAudioIO()
await agent.run(inputs=[audio_io.input()], outputs=[audio_io.output()])
```

Requires Python 3.12+ and `pip install strands-agents[bidi]`.

---

## 5. Amazon Connect Telephony

Amazon Connect is the **primary telephony layer** — it provides managed phone numbers, native Nova Sonic speech-to-speech, and eliminates the need for a custom Voice Bridge service.

### Why Connect Over Twilio

| Factor | Amazon Connect | Twilio |
|--------|---------------|--------|
| Nova Sonic integration | **Native — managed, no code** | Custom Voice Bridge (~640 lines) |
| Audio format conversion | **Not needed** (Connect handles it) | You must convert μ-law ↔ LPCM |
| Per-minute cost (voice + AI) | **$0.038/min** (Nova Sonic included) | $0.079/min (ConversationRelay) + LLM |
| Phone number (local AU) | **~$0.90/month** | $1.15/month |
| Setup time | **~1 hour** (GUI + CDK) | Several hours (server + WebSocket) |
| Custom code needed | **Minimal** (Lambda for tool bridging) | ~640 lines (Voice Bridge service) |

### Australia / Melbourne Phone Numbers

Amazon Connect supports Australian DID phone numbers. Requirements:
- **Documentation**: Business address in Australia + ABN or business registration
- **Number types**: Local DID (geographic), Toll-Free (1800), Mobile
- **Region constraint**: Nova Sonic is only in `us-east-1` / `us-west-2`, so Connect instance lives in Virginia
- **Latency**: ~200ms network (AU→US) + ~300ms model = **~500ms total** (natural for conversation)

### Architecture

```
Caller's Phone (Melbourne +61)
       │
       ▼
Amazon Connect (us-east-1)
├── Contact Flow routes to Conversational AI Bot
├── Bot configured with Nova 2 Sonic speech-to-speech
├── Nova Sonic voice: "tiffany" (warm, feminine)
├── System prompt: vendor voice_prompt (from DynamoDB)
│
├── Nova Sonic handles conversation autonomously:
│   - Greetings, personality, simple Q&A (from system prompt)
│   - Tool calls when business logic needed:
│
│   ┌─────────────────────────────────────────┐
│   │  ask_agent tool → Lambda function       │
│   │  Lambda → OpenClaw Gateway (WebSocket)  │
│   │  OpenClaw → Pi Agent → tools:           │
│   │    - check_availability                 │
│   │    - get_packages                       │
│   │    - create_lead_from_call              │
│   │    - get_lead_context                   │
│   │  Result → Lambda → Nova Sonic           │
│   │  Nova Sonic speaks the answer            │
│   └─────────────────────────────────────────┘
│
├── Call ends:
│   ├── Connect saves call recording (S3, automatic)
│   ├── Connect provides transcript (Contact Lens, automatic)
│   ├── Lambda: persist call record → DynamoDB
│   └── Lambda: ingest transcript → OpenClaw as SMS
│
└── Analytics (included in Connect):
    ├── Call duration, wait time, sentiment
    ├── Contact Lens: real-time + post-call analytics
    └── Dashboard in Connect admin console
```

### Phone Number Provisioning (Programmatic)

```python
import boto3

connect = boto3.client('connect', region_name='us-east-1')

# 1. Search available Australian numbers
numbers = connect.search_available_phone_numbers(
    TargetArn=f'arn:aws:connect:us-east-1:{ACCOUNT_ID}:instance/{INSTANCE_ID}',
    PhoneNumberCountryCode='AU',
    PhoneNumberType='DID',
    MaxResults=10
)

# 2. Claim a number for this vendor
result = connect.claim_phone_number(
    InstanceId=INSTANCE_ID,
    PhoneNumber=numbers['AvailableNumbersList'][0]['PhoneNumber'],
    PhoneNumberDescription=f'WeddingOS vendor {vendor_id}'
)
phone_number_id = result['PhoneNumberId']

# 3. Associate with the vendor's contact flow
connect.associate_phone_number_contact_flow(
    PhoneNumberId=phone_number_id,
    InstanceId=INSTANCE_ID,
    ContactFlowId=vendor_contact_flow_id  # per-vendor flow with their voice_prompt
)

# 4. Store in DynamoDB
await vendor_repo.update(vendor_id, {
    "phone_number": claimed_number,
    "connect_phone_number_id": phone_number_id,
    "phone_enabled": True
})
```

### Contact Flow Design

Each vendor gets a Contact Flow that:
1. **Sets the voice** to Nova Sonic with vendor's preferred voice ID
2. **Loads the system prompt** (vendor's `voice_prompt` from DynamoDB via Lambda)
3. **Configures the `ask_agent` tool** for business logic delegation
4. **Routes to the Conversational AI Bot** which handles the full conversation
5. **On call end**: triggers Lambda for persistence + transcript ingestion

This can be templated — one base Contact Flow, parameterized per vendor.

### CDK Deployment (Infrastructure as Code)

AWS provides `sample-amazon-connect-bedrock-agent-voice-integration` CDK sample that deploys:
- Connect instance + phone number
- Bedrock agent with custom actions
- Lambda functions for tool execution
- DynamoDB tables

We adapt this template for Wedding OS.

### Fallback: Twilio (Alternative Path)

If Amazon Connect has issues with AU number provisioning or Nova Sonic availability, the Twilio + custom Voice Bridge architecture remains viable. The Voice Bridge code (~640 lines) is documented in the appendix. Key difference: you build and host the audio bridge yourself.

---

## 6. OpenClaw Integration

### No Core Changes Required

OpenClaw's architecture is already generic enough. The voice bridge calls OpenClaw's existing gateway WebSocket exactly like the email ingress does — just with a different session key format.

### Session Key for Voice

```
Email:  agent:vendor-<vendorId>:email:<threadId>
Phone:  agent:vendor-<vendorId>:phone:<callerPhone>
SMS:    agent:vendor-<vendorId>:sms:<callerPhone>
```

Voice tool calls use a temporary session: `agent:vendor-<vendorId>:voice-call:<callId>`

After the call ends, the transcript is ingested into the SMS session: `agent:vendor-<vendorId>:sms:<callerPhone>` — this enables cross-channel continuity. If the same phone number later texts or emails, the agent has the call context.

### SMS Channel Plugin (New OpenClaw Extension)

SMS follows the standard OpenClaw channel plugin contract:

```typescript
// extensions/twilio-sms/types.ts
export const TwilioSmsChannel: ChannelPlugin = {
  id: "twilio-sms",
  meta: { label: "Twilio SMS", blurb: "SMS via Twilio" },

  normalize(webhook) {
    return {
      Channel: "twilio-sms",
      From: webhook.From,        // +15125550100
      To: webhook.To,            // vendor's Twilio number
      Body: webhook.Body,
      SessionKey: webhook.From,  // phone number = session identity
      ChatType: "direct",
      Provider: "twilio",
    }
  },

  outbound: {
    async sendText(to, body, account) {
      const client = twilio(account.accountSid, account.authToken)
      await client.messages.create({ to, from: account.phoneNumber, body })
    }
  }
}
```

**Estimated build: 1-2 days.**

---

## 7. Vendor Context Injection (How the Voice Agent Knows About the Vendor)

This is the critical design question: how does Nova Sonic know about SOUL.md, Principles.md, packages, and vendor personality?

### Answer: Hybrid Injection (Proven by Oratio)

**Two injection points work together:**

### Point 1: System Prompt (loaded once at call start — instant responses)

Nova Sonic's system prompt includes the vendor's personality and basic info from SOUL.md:

```
"You are Sarah from Sarah's Photography. You're warm, friendly,
 and genuinely excited to talk to potential clients.

 VOICE STYLE:
 - Speak naturally with occasional filler words
 - Be enthusiastic but not pushy
 - Use the caller's name once you learn it

 PACKAGES (from Principles.md):
 - Essential: $2,500 (4 hours coverage)
 - Classic: $4,000 (8 hours + engagement session)
 - Luxury: $6,500 (full day + prints)

 CRITICAL TOOL USAGE PROTOCOL:
 STEP 1: SPEAK FIRST - 'Let me check that for you!'
 STEP 2: USE ask_agent TOOL
 STEP 3: RESPOND with tool results naturally
 NEVER skip Step 1. Silence while processing feels broken."
```

This is stored in the vendor's DynamoDB record as `voice_prompt`, generated from SOUL.md + Principles.md during onboarding. Simple questions (pricing, style, personality) are answered **instantly** — no tool call needed.

### Point 2: Dynamic Tool Calls (fetched during conversation — detailed data)

Complex questions trigger the `ask_agent` tool → OpenClaw:

```
Caller: "Are you available October 4th?"

Nova Sonic: "Let me check that for you!"           ← instant (from system prompt)
            → ask_agent("October 4 availability")  ← tool call to OpenClaw

OpenClaw Pi Agent:
  → get_lead_context(caller_phone)                  ← DynamoDB query
  → check_availability("2026-10-04")                ← calendar check
  ← "Available. 3 bookings that week. Prior email
     thread with this caller about Oct venues."      ← cross-channel context!

Nova Sonic: "Great news! October 4th is available.  ← speaks tool result naturally
             I actually see we were chatting by
             email about venues — have you decided
             on the Ritz?"                           ← references email thread!
```

### Why Hybrid Works

| Question Type | Source | Latency | Example |
|---|---|---|---|
| Personality / tone | System prompt | **Instant** | "Tell me about yourself" |
| Package pricing | System prompt | **Instant** | "What are your rates?" |
| Date availability | Tool call | **~500ms** | "Are you free Oct 4?" |
| Lead history | Tool call | **~500ms** | "What did we discuss before?" |
| Cross-channel context | Tool call | **~500ms** | References prior email threads |

### Data Flow: From S3 to Caller's Ear

```
ONBOARDING (one-time):
  SOUL.md (S3) ──► Generate voice_prompt ──► Store in DynamoDB vendor record
  Principles.md ──► Extract packages/pricing ──► Include in voice_prompt

CALL START:
  Connect Contact Flow ──► Lambda fetches vendor.voice_prompt from DynamoDB
                       ──► Injects as Nova Sonic system prompt
                       ──► Nova Sonic session starts with full personality

DURING CALL:
  Nova Sonic answers simple Qs from system prompt (instant)
  Nova Sonic calls ask_agent for complex Qs:
    Lambda ──► OpenClaw Gateway ──► Pi Agent ──► tools ──► DynamoDB/S3
    Result ──► Lambda ──► Nova Sonic ──► speaks to caller
```

---

## 8. Agent Tools for Voice

### Existing Tools (Reused from Email)

| Tool | Description | Used in Voice? |
|------|-------------|---------------|
| `get_lead_context` | Fetch lead history + context | YES — "Let me pull up your info" |
| `get_thread_history` | Fetch conversation history | YES — cross-channel context |
| `store_draft` | Save email draft | NO — voice responds immediately |
| `send_email` | Send via AgenticMail | MAYBE — send email recap after call |

### New Tools for Voice

```typescript
// Voice-specific tools registered in OpenClaw

tool: check_availability
  description: "Check if vendor is available on a specific date"
  parameters: { date: string, duration_hours?: number }
  returns: { available: boolean, reason?: string, alternative_dates?: string[] }

tool: get_packages
  description: "Get vendor's package list and pricing"
  parameters: {}
  returns: { packages: [{ name, description, price, duration }] }

tool: create_lead_from_call
  description: "Create a new lead from voice call information"
  parameters: {
    caller_name: string,
    caller_phone: string,
    event_date?: string,
    venue?: string,
    budget?: number,
    notes?: string
  }
  returns: { lead_id: string, next_steps: string }

tool: schedule_consultation
  description: "Schedule a follow-up consultation or callback"
  parameters: {
    caller_phone: string,
    preferred_datetime: string,
    duration_minutes?: number
  }
  returns: { confirmation: string, calendar_link?: string }

tool: send_sms_followup
  description: "Send SMS followup to caller after voice call"
  parameters: { phone: string, message: string }
  returns: { success: boolean, message_id: string }

tool: get_call_pricing_from_history
  description: "Find similar past bookings to suggest pricing"
  parameters: { event_type: string, budget_range?: [number, number] }
  returns: { similar_bookings: [{ date, price, notes }] }
```

---

## 9. DynamoDB Schema Extensions

### New Table: `calls`

```
Table: weddingos-calls
  PK: vendorId (String)
  SK: callId (String, ULID)

  Attributes:
    caller_phone:     String     "+15125550100"
    caller_name:      String     "Sarah Johnson" (extracted from conversation)
    start_time:       String     ISO timestamp
    end_time:         String     ISO timestamp
    duration_seconds: Number     180
    status:           String     "completed" | "missed" | "voicemail" | "transferred"
    outcome:          String     "lead_created" | "booking_made" | "info_only" | "callback_scheduled"
    lead_id:          String     (optional) link to leads table
    transcript_summary: String   AI-generated 2-3 sentence summary
    recording_s3_key: String     (optional) S3 path for call recording
    tools_invoked:    List       ["check_availability", "get_packages"]
    nova_sonic_tokens: Map       { input_speech: N, output_speech: N }
    created_at:       String     ISO timestamp

  GSI: PhoneLookup
    PK: caller_phone
    SK: start_time
    Purpose: Find all calls from a specific phone number
```

### New Table: `call_transcripts`

```
Table: weddingos-call-transcripts
  PK: vendorId#callId (String)
  SK: segment_ts (Number, milliseconds within call)

  Attributes:
    speaker:     String    "caller" | "agent"
    content:     String    Transcribed text segment
    confidence:  Number    ASR confidence (0-1)
    tool_call:   Map       (optional) { tool_name, input, output }
```

### Updates to Existing Tables

```
vendors table — new attributes:
  phone_number:     String     Australian DID number (+61...)
  connect_phone_id: String     Amazon Connect phone number ID
  phone_enabled:    Boolean    Whether phone channel is active
  voice_config:     Map        { voice_id: "tiffany", personality: "warm and professional" }
  voice_prompt:     String     Nova Sonic system prompt (generated from SOUL.md)

threads table — expanded:
  channel:          String     "email" | "sms" | "phone_call" (was just "email")

messages table — expanded:
  media_type:       String     "text" | "voice_transcript"
  call_id:          String     (optional) link to calls table
```

---

## 10. Frontend Voice Components

### Adapted from Oratio's `VoiceTestingInterface.tsx`

Oratio provides a complete voice testing component with:
- Start/stop call button with visual voice bars (48-bar animation)
- Timer showing call duration
- Live transcript display (left = agent, right = caller)
- Tool invocation panel
- WebSocket audio capture at 16kHz + playback at 24kHz
- Barge-in handling (clears audio queue on interrupt)

### Wedding OS Voice Pages

| Page | Purpose |
|------|---------|
| **Vendor Dashboard** — Call Log | List of recent calls with duration, outcome, transcript summary |
| **Call Detail View** | Full transcript, tools used, lead created, follow-up actions |
| **Phone Setup** (in onboarding) | Claim AU number via Connect, choose voice, test call |
| **Voice Settings** | Change voice, update personality, enable/disable auto-answer |
| **Live Call Monitor** (stretch) | Real-time view of active call with transcript |

### Call Log Component

```tsx
// pages/vendor/calls/index.tsx
export default function CallLogPage() {
  const { data: calls } = useQuery(['calls'], fetchCalls)

  return (
    <div>
      <h1>Phone Calls</h1>
      {calls.map(call => (
        <CallCard
          key={call.callId}
          callerName={call.caller_name}
          callerPhone={call.caller_phone}
          duration={call.duration_seconds}
          outcome={call.outcome}
          summary={call.transcript_summary}
          timestamp={call.start_time}
        />
      ))}
    </div>
  )
}
```

---

## 11. Voice Personality System

### Generation from SOUL.md

The vendor's SOUL.md (already generated during email onboarding) is used to create a voice-specific system prompt:

```
SOUL.md content:
  "Sarah's Photography captures authentic moments with a candid,
   documentary style. She's warm, approachable, and values genuine
   connections with her clients."

Voice Prompt (generated):
  "You are Sarah, a wedding photographer. You're warm, friendly, and
   genuinely excited to talk to potential clients.

   VOICE STYLE:
   - Speak naturally with occasional filler words ("um", "yeah")
   - Be enthusiastic but not pushy
   - Use the caller's name once you learn it
   - Mirror their energy level

   CRITICAL TOOL USAGE PROTOCOL:
   STEP 1: SPEAK FIRST - "Let me check that for you!"
   STEP 2: USE ask_agent TOOL
   STEP 3: RESPOND with tool results naturally
   NEVER skip Step 1. Silence while processing feels broken.

   PACKAGES: [from Principles.md]
   - Essential: $2,500 (4 hours)
   - Classic: $4,000 (8 hours)
   - Luxury: $6,500 (full day + engagement session)

   QUALIFICATION: Ask about date, venue, guest count, and what
   style of photography they're looking for."
```

### Voice Personality Fields

| Field | Example | Purpose |
|-------|---------|---------|
| `voice_id` | `"tiffany"` | Nova Sonic voice selection |
| `identity` | `"Sarah, wedding photographer"` | Who the agent represents |
| `tone` | `"warm and conversational"` | Speaking style |
| `formality` | `"casual-professional"` | Register |
| `enthusiasm` | `"high"` | Energy level |
| `filler_words` | `"occasionally"` | Natural speech pattern |
| `pacing` | `"moderate"` | Speech speed |

---

## 12. Deployment Architecture

### Topology

```
┌───────────────────────────────────────────────────────────────────┐
│                  AWS DEPLOYMENT (us-east-1)                        │
│                                                                   │
│  ECS Fargate Cluster                                             │
│  ┌──────────────────┐  ┌──────────────────┐                     │
│  │  Frontend         │  │  Platform API     │                     │
│  │  (Next.js)        │  │  (FastAPI)        │                     │
│  │  Port 3001        │  │  Port 8000        │                     │
│  └──────────────────┘  └──────────────────┘                     │
│                                                                   │
│  EC2 (t3.medium or larger)                                       │
│  ┌──────────────────┐  ┌──────────────────┐                     │
│  │  OpenClaw Runtime │  │  AgenticMail      │                     │
│  │  Port 18789       │  │  Port 3100        │                     │
│  └──────────────────┘  └──────────────────┘                     │
│                                                                   │
│  Amazon Connect (us-east-1)           ← NEW (managed)            │
│  ┌──────────────────────────────────────────┐                    │
│  │  AU Phone Number (+61...)                │                    │
│  │  Contact Flow → Nova Sonic Bot           │                    │
│  │  Nova Sonic (native, no custom code)     │                    │
│  │  ask_agent tool → Lambda → OpenClaw      │                    │
│  │  Contact Lens (analytics, transcripts)   │                    │
│  │  Call recordings → S3 (automatic)        │                    │
│  └──────────────────────────────────────────┘                    │
│                                                                   │
│  Lambda Functions (tool bridging)         ← NEW                  │
│  ┌──────────────────┐  ┌──────────────────┐                     │
│  │  ask_agent_handler│  │  call_lifecycle   │                     │
│  │  (tool → OpenClaw)│  │  (persist + ingest│                     │
│  └──────────────────┘  └──────────────────┘                     │
│                                                                   │
│  Amazon Bedrock                                                   │
│  ┌──────────────────┐  ┌──────────────────┐                     │
│  │  Nova 2 Lite      │  │  Nova 2 Sonic     │ (managed by Connect)│
│  │  (text reasoning)  │  │  (speech-to-speech│                     │
│  └──────────────────┘  └──────────────────┘                     │
│                                                                   │
│  DynamoDB  │  S3  │  Cognito  │  CloudWatch                     │
└───────────────────────────────────────────────────────────────────┘
```

### Key Advantage: No Voice Bridge Service

With Amazon Connect, we do NOT need a custom Voice Bridge. Connect handles:
- Phone number provisioning and PSTN connectivity
- Audio encoding/decoding
- Nova Sonic session management
- Call recording and transcript generation (Contact Lens)
- Real-time analytics and sentiment detection

The only custom code is Lambda functions for tool bridging (ask_agent → OpenClaw) and call lifecycle events (persist records, ingest transcripts).

---

## 13. Implementation Roadmap

### Phase P (Phone) — Task Breakdown

#### P-01: Amazon Connect Instance + Contact Flow [1 day]
- Create Connect instance in `us-east-1` via CDK/CloudFormation
- Configure Conversational AI Bot with Nova 2 Sonic
- Create base Contact Flow template (parameterized per vendor)
- Set up IAM roles for Lambda ↔ Connect ↔ Bedrock
- **Done**: Connect instance running, base Contact Flow accepts calls and speaks via Nova Sonic

#### P-02: Tool Bridging Lambda — `ask_agent` [1 day]
- Lambda function that receives `ask_agent` tool call from Connect/Nova Sonic
- WebSocket client to OpenClaw Gateway (reuse existing session protocol)
- Tool result forwarding back to Nova Sonic via Lambda response
- Error handling: graceful fallback if OpenClaw is unreachable
- **Done**: Nova Sonic can query OpenClaw for business logic during a live call

#### P-03: Voice-Specific Agent Tools [2 days]
- `check_availability`, `get_packages`, `create_lead_from_call`
- `schedule_consultation`, `send_sms_followup`
- Register in OpenClaw tool system (TypeScript, global registry)
- **Done**: Voice agent can check dates, quote prices, create leads

#### P-04: Vendor Phone Provisioning [1 day]
- Search + claim AU DID number via Connect API (`search_available_phone_numbers` → `claim_phone_number`)
- Associate number with vendor's parameterized Contact Flow
- Store `phone_number`, `connect_phone_number_id`, `phone_enabled` in DynamoDB vendor record
- **Done**: Vendor gets an Australian phone number during onboarding

#### P-05: Voice Personality Generation [1 day]
- Generate `voice_prompt` from SOUL.md + Principles.md (Nova Lite generation, like existing onboarding)
- Include packages/pricing, personality traits, tool usage protocol in system prompt
- Voice config: `voice_id`, `tone`, `pacing` stored in vendor record
- Lambda fetches `voice_prompt` at call start → injects as Nova Sonic system prompt
- **Done**: Each vendor's phone agent has their unique personality

#### P-06: Call Lifecycle Lambda [1 day]
- Triggered on call end by Connect Contact Flow
- Persist call record to DynamoDB `calls` table (duration, outcome, caller info)
- Fetch transcript from Contact Lens → store in `call_transcripts` table
- Ingest transcript into OpenClaw as SMS message (cross-channel context)
- Generate AI summary (Nova Lite) → store in call record
- **Done**: Every call is recorded, transcribed, and searchable

#### P-07: DynamoDB Schema Extensions [0.5 day]
- Create `weddingos-calls` table (PK: vendorId, SK: callId, GSI: PhoneLookup)
- Create `weddingos-call-transcripts` table (PK: vendorId#callId, SK: segment_ts)
- Add phone fields to vendors table (`phone_number`, `connect_phone_id`, `voice_config`, `voice_prompt`)
- Add `channel` field to threads table, `media_type` and `call_id` to messages table
- **Done**: All phone-related data models deployed

#### P-08: SMS Channel Plugin [1 day]
- OpenClaw channel plugin for SMS (Connect SMS or SNS)
- Inbound webhook handler + outbound send
- Session key by phone number: `agent:vendor-<vendorId>:sms:<callerPhone>`
- **Done**: SMS follow-ups work through OpenClaw

#### P-09: Frontend — Phone Setup + Call Log [2 days]
- Phone setup step in onboarding wizard (claim number, choose voice, test call)
- Call log page with search/filter/duration/outcome
- Call detail view with full transcript + tools used + lead created
- Voice settings page (change voice, update personality, enable/disable)
- **Done**: Vendor can set up phone and see call history

#### P-10: E2E Voice Test [1 day]
- Full flow: call AU number → Nova Sonic greets → ask_agent tool → OpenClaw responds → caller hears answer
- Post-call: transcript persisted → ingested to OpenClaw → cross-channel context available
- Cross-channel: email thread references phone call context and vice versa
- Connect Contact Lens analytics visible in dashboard
- **Done**: Complete phone integration verified end-to-end

### Total: ~11.5 days (~2 weeks with buffer)

### Key Savings vs Twilio Approach

| Eliminated | Savings |
|-----------|---------|
| Voice Bridge service (FastAPI + Smithy SDK) | ~2.5 days |
| Audio format conversion (μ-law ↔ LPCM) | Built into Connect |
| Bidirectional audio streaming code | Managed by Connect |
| Session reconnection logic | Managed by Connect |
| Call recording implementation | Contact Lens (automatic) |
| Transcript generation | Contact Lens (automatic) |

### Hackathon Fast-Track (Demo Only)

For hackathon demo, use browser WebRTC with Strands BidiAgent (skip Connect setup):

| Task | Time |
|------|------|
| P-02 (Lambda tool bridge → OpenClaw) | 4 hours |
| P-03 (2-3 voice tools only) | 4 hours |
| P-05 (hardcode one vendor personality) | 1 hour |
| Browser UI (Strands BidiAgent + BidiAudioIO) | 3 hours |
| Quick demo of cross-channel context | 2 hours |
| **Total** | **~2 days** |

For full production with Amazon Connect, add P-01 + P-04 + P-06 (~3 more days).

---

## 14. Competitive Positioning

### Market Landscape

| Competitor | Latency | Cross-Channel Context | Wedding-Specific |
|-----------|---------|----------------------|-----------------|
| Vapi.ai | ~700ms | NO | NO |
| Retell AI | ~600ms | NO | NO |
| Bland AI | ~800ms | NO | NO |
| ElevenLabs | Low | NO | YES (templates) |
| Smith.ai | N/A (human hybrid) | Partial | YES |
| My AI Front Desk | Unknown | NO | YES |
| **Wedding OS** | **~300ms** | **YES (email + phone + SMS)** | **YES** |

### Our Differentiator

> **"The same AI agent that emails couples also answers their phone calls, with full context from both channels, using the vendor's own personality and communication style."**

No competitor does this. They are all standalone voice receptionists with no memory of email conversations.

### Cost Comparison

| Solution | Monthly Cost |
|----------|-------------|
| Human receptionist | $2,000-4,000 |
| Smith.ai (human hybrid) | $200-500 |
| Managed AI receptionist platforms | $100-500 |
| **Wedding OS (per vendor)** | **~$45** |

---

## 15. Cost Analysis

### Amazon Connect Pricing (us-east-1)

| Component | Price |
|-----------|-------|
| **Voice (inbound)** | $0.018/min |
| **Nova Sonic (bundled in Conversational AI)** | $0.020/min |
| **Combined voice + AI** | **$0.038/min** |
| AU DID phone number | ~$0.90/month |
| Contact Lens (post-call analytics) | $0.015/min |
| Contact Lens (real-time) | $0.025/min (optional) |

> **Key advantage**: Nova Sonic token costs are bundled into the $0.020/min Conversational AI rate — no separate Bedrock charges for voice AI.

### Per-Call Estimate

- Average wedding inquiry call: 3-5 minutes
- Connect voice + Nova Sonic: $0.038/min × 4 min = **$0.15/call**
- Contact Lens post-call analytics: $0.015/min × 4 min = $0.06/call (optional)
- Lambda invocations (~3 tool calls): ~$0.001
- **Total per call: ~$0.15-0.21**

### Per-Vendor Monthly Estimate

| Item | Cost |
|------|------|
| AU DID phone number | $0.90/month |
| Connect voice + AI (~50 calls × 4 min) | $7.60 |
| Contact Lens post-call (~50 calls × 4 min) | $3.00 |
| Lambda invocations | $0.10 |
| SMS follow-ups (~50 messages via SNS) | $0.50 |
| DynamoDB (call records + transcripts) | $0.50 |
| **Total per vendor** | **~$12.60/month** |

### Comparison: Connect vs Twilio + Custom Bridge

| Cost Item | Amazon Connect | Twilio + Voice Bridge |
|-----------|---------------|----------------------|
| Phone number | $0.90/mo | $1.00/mo |
| Minutes (50 calls × 4 min) | $7.60 | $2.60 |
| Nova Sonic | Included | $3.50 (separate Bedrock) |
| EC2 for Voice Bridge | Not needed | ~$5.00 |
| Transcripts/analytics | $3.00 (Contact Lens) | Custom build |
| **Monthly total** | **~$12.60** | **~$12.10** |
| **Custom code** | **~50 lines (Lambda)** | **~640 lines (Voice Bridge)** |
| **Maintenance** | **Minimal (managed)** | **Moderate (self-hosted)** |

> Connect is ~$0.50/month more per vendor but eliminates ~640 lines of custom audio bridging code and all the maintenance/debugging overhead that comes with it. The Contact Lens analytics are a bonus.

With platform margin: **$39-49/month retail** — still 10x cheaper than alternatives.

---

## 16. Key Reference Material

### AWS Sample Repositories
- `aws-samples/amazon-nova-samples/speech-to-speech/` — Python console client, tool use
- `aws-samples/sample-nova-sonic-websocket-agentcore` — React + Strands + CDK
- `aws-samples/sample-Nova-Sonic-AgentCore-Healthcare-Call-Center` — Healthcare calls

### Oratio Reference Implementation
- `voice_simple.py` (~462 lines) — WebSocket + Smithy SDK + ToolProcessor
- `VoiceTestingInterface.tsx` (~400 lines) — Frontend audio capture/playback
- `PromptGenerator` DSPy module — Voice prompt generation from personality text
- See: `docs/understand-oratio/` (7 files)

### Competitive Analysis
- `docs/mail-integration-completed/discussion/voice-ai-competitive-analysis.md` — Full 12-section analysis

### Amazon Connect + Nova Sonic
- `aws-samples/sample-amazon-connect-bedrock-agent-voice-integration` — CDK sample (Connect + Bedrock agent + Lambda)
- Amazon Connect admin guide — Contact Flow design, phone number management
- Amazon Connect Conversational AI — Nova Sonic native integration
- Contact Lens — Post-call and real-time analytics, transcript generation

### Framework Documentation
- [Strands Agents BidiAgent](https://strandsagents.com) — `strands.experimental.bidi` (hackathon fast-track)
- [Nova Sonic Developer Guide](https://docs.aws.amazon.com/nova/latest/userguide/)
- [Pipecat + Nova Sonic](https://github.com/pipecat-ai/pipecat) — 5.8K stars (alternative to Strands)
- [LiveKit Agents](https://github.com/livekit/agents) — 5.8K stars (WebRTC alternative)

### Project Architecture Docs
- `plan/VENDOR_PLATFORM_ARCHITECTURE.md` — Email MVP architecture (this document's sibling)
- `docs/aws-nova-hackathon.md` — Full Nova model family reference

---

*Research compiled: 2026-02-26*
*Sources: 4 parallel research agents — Oratio infrastructure analysis, Nova Sonic API deep research, OpenClaw extension analysis, competitive landscape analysis*
