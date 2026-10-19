# Agentic Voice AI

**An LLM-powered conversational healthcare assistant for intelligent patient scheduling, voice interactions, and automated appointment workflows.**

Agentic Voice AI is a full-stack application built with Next.js, TypeScript, Anthropic Claude, and Vapi. It uses **LLM tool calling and structured workflow orchestration** to manage patient interactions across web chat and voice channels.

The system supports multi-step appointment scheduling, specialist matching, slot selection from simulated availability, and automated notifications while maintaining conversational context during chat-to-voice transitions.

## Key Features

### 1. Agentic AI & Conversational Workflows

- **LLM-powered reasoning:** Uses Claude with structured function calling to interpret patient requests and execute domain-specific actions.
- **Multi-step orchestration:** Coordinates patient intake, specialist matching, availability retrieval, and appointment booking.
- **Context-aware interactions:** Maintains session context across successive tool executions.
- **Workflow guardrails:** Restricts the agent to supported scheduling, office information, and simulated prescription-status tasks.

### 2. Intelligent Appointment Scheduling

- Collects patient details including name, date of birth, contact information, and reason for visit.
- Matches patients with appropriate specialists based on reported concerns.
- Retrieves available appointment slots based on preferred dates and times.
- Handles booking conflicts using structured `alreadyBooked` responses.
- Prevents duplicate booking attempts after successful confirmation.
- Exposes appointment records through a dedicated dashboard API.
- Generates simulated appointment availability for 45 days.

### 3. Conversational Voice AI

- Integrates **Vapi** for outbound and inbound voice interactions.
- Supports chat-to-voice handoff via a "Continue via Phone Call" action.
- Associates voice tool calls with existing chat sessions using phone-based session indexing.
- Handles Vapi webhook payloads in `toolCallList` and `toolWithToolCallList` formats.
- Personalizes inbound callbacks using session context.

### 4. Automated Notifications

- Sends booking confirmation emails through **Resend**.
- Supports opt-in SMS reminders through **Twilio**.
- Triggers notifications through structured tool workflows.

### 5. Appointment Dashboard

- Displays scheduled appointments using a Next.js dashboard.
- Retrieves current appointment data from `/api/appointments`.
- Uses `no-store` fetching to avoid stale dashboard responses.
- Provides an interactive chat interface with animated messaging components.

## System Architecture

The application follows a tool-augmented conversational AI architecture:

```text
Patient
   |
   +---- Web Chat (Next.js)
   |          |
   |          v
   |     Claude LLM
   |          |
   |     Tool Calling Loop
   |          |
   |          +---- Patient Intake
   |          +---- Specialist Matching
   |          +---- Slot Availability
   |          +---- Appointment Booking
   |          +---- Office Information
   |          +---- Prescription Status
   |
   +---- Voice AI (Vapi)
              |
              v
        Webhook Handler
              |
        Session Resolution
              |
        Shared Scheduling Tools
              |
              v
       Appointment Store
              |
              +---- Email (Resend)
              +---- SMS (Twilio)
              +---- Appointment Dashboard
```

### Architecture Highlights

- Claude handles conversational reasoning and structured tool selection for web chat.
- Vapi manages voice interactions and forwards tool execution requests to the backend.
- Both interfaces use the application's scheduling tools.
- Session indexing supports contextual continuity between chat and voice.
- Appointments and session records are currently held in memory.

## Technology Stack

| Category | Technologies |
|---|---|
| Frontend | Next.js 16, React 19, TypeScript, Tailwind CSS 4 |
| LLM | Anthropic Claude Sonnet 4 |
| Agentic Workflows | LLM Tool Calling, Function Schemas, Tool Orchestration |
| Voice AI | Vapi, Voice Webhooks |
| Notifications | Resend, Twilio |
| Backend | Next.js App Router, Server Routes, REST APIs |
| Deployment Documentation | AWS EC2, Nginx, PM2, HTTPS |
| State Management | In-memory sessions and appointment storage |

## Project Structure

```text
src/
├── app/
│   ├── page.tsx                 # Conversational chat interface
│   ├── dashboard/
│   │   └── page.tsx            # Appointment dashboard
│   └── api/
│       ├── chat/route.ts       # Claude tool-calling loop
│       ├── appointments/route.ts
│       └── vapi/
│           ├── call/route.ts   # Outbound voice calls
│           └── webhook/route.ts
├── data/
│   ├── doctors.ts              # Simulated providers and slots
│   └── store.ts                # In-memory data storage
└── lib/
    ├── claude.ts               # System prompt and tool schemas
    ├── chat-tools.ts           # Scheduling tool implementations
    ├── vapi-webhook.ts         # Webhook payload normalization
    ├── vapi-tools.ts           # Voice function definitions
    ├── email.ts                # Email notifications
    └── sms.ts                  # SMS notifications
```

## Getting Started

### Prerequisites

- Node.js 20+
- npm 10+
- Anthropic API key
- Optional Vapi, Resend, and Twilio credentials for integrations

### 1. Clone the Repository

```bash
git clone https://github.com/SurajPatelM/agentic-voice-ai.git
cd agentic-voice-ai
npm install
```

### 2. Configure Environment Variables

```bash
cp .env.example .env.local
```

Configure the appropriate credentials in `.env.local`:

| Variable | Purpose |
|---|---|
| `ANTHROPIC_API_KEY` | Claude LLM access |
| `VAPI_API_KEY` | Voice API authentication |
| `VAPI_PHONE_NUMBER_ID` | Configured Vapi phone number |
| `VAPI_ASSISTANT_ID` | Voice assistant configuration |
| `RESEND_API_KEY` | Email notification service |
| `RESEND_FROM_EMAIL` | Verified sender address |
| `TWILIO_ACCOUNT_SID` | Twilio account identifier |
| `TWILIO_AUTH_TOKEN` | SMS authentication |
| `TWILIO_PHONE_NUMBER` | SMS sender phone number |
| `NEXT_PUBLIC_APP_URL` | Application base URL |

**Security:** Never commit `.env.local`, credentials, tokens, or patient-identifying data.

### 3. Run Locally

```bash
npm run dev
```

Open the application:

- **Chat interface:** http://localhost:3000
- **Appointment dashboard:** http://localhost:3000/dashboard

### 4. Build and Validate

```bash
npm run lint
npm run build
npm run start
```

## Voice Integration

The voice workflow requires Vapi configuration.

1. Configure the assistant's server URL to point to your public webhook endpoint:

   `https://YOUR_DOMAIN/api/vapi/webhook`

2. Ensure the voice function schemas match those defined in `getClaudeTools()` and `getVapiModelFunctions()`.

3. Configure synchronous tool execution (`async: false`).

4. Verify that webhook requests return successful HTTP responses containing the required tool `results` structure.

5. Test outbound initiation and inbound callback personalization.

See **[VAPI_SETUP.md](VAPI_SETUP.md)** for detailed configuration.

## Deployment

The repository includes deployment instructions for:

- AWS EC2 hosting
- Nginx reverse proxy
- HTTPS configuration
- PM2 process management

See **[DEPLOY_EC2.md](DEPLOY_EC2.md)** for deployment steps.

A successful production build does not by itself make the application ready for use with real patient information.

## Safety & Current Limitations

This project is a **functional engineering prototype**, not a clinical system or production healthcare platform.

- **Simulated healthcare data:** Providers, scheduling availability, and prescription-refill status are simulated.
- **In-memory persistence:** Appointment and session records are stored within a Node.js process and are lost when the process restarts.
- **No medical advice:** The agent is instructed to stay within scheduling, office-information, and simulated refill-status workflows.
- **External service dependencies:** Voice calls, email delivery, and SMS require valid third-party service credentials and configuration.
- **Healthcare privacy:** This implementation has not been established as HIPAA-compliant. It should not process real protected health information without appropriate authentication, access controls, durable storage, encryption, audit logging, and compliance review.

### Potential Future Improvements

- Database-backed appointment and session persistence
- Authentication and role-based access control
- Structured observability and tracing
- Automated end-to-end agent evaluation
- Stronger recovery from failed tool executions
- Improved concurrent booking and scheduling safeguards

## License

Private use. No open-source license is granted unless a license file is added to the repository.
