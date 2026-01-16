# Stripe Payment Lifecycle

A production-grade reference implementation demonstrating proper Stripe payment handling with explicit state management, webhook-driven confirmation, and idempotent processing.

**This is not a demo checkout. This is a payment system.**

---

## Why This Project?

Many Stripe integrations make critical mistakes:

- **Trusting client-side payment confirmation** — The frontend's `confirmPayment()` returning success doesn't mean funds were captured
- **Missing webhook signature verification** — Exposes the API to forged payment confirmations
- **Processing duplicate webhook events** — Leads to double-processing and data inconsistencies
- **Storing card data on the backend** — Violates PCI compliance and increases security risk

**This project demonstrates the correct patterns.**

---

## Key Principles

### 1. Webhooks Are the Source of Truth

```
❌ Wrong: Trust stripe.confirmPayment() result
✅ Right: Only mark payment succeeded when webhook confirms it
```

The frontend's `confirmPayment()` returning success doesn't mean funds were captured. Network issues, 3D Secure failures, or fraud checks can still fail the payment. **Only the webhook provides definitive confirmation.**

### 2. Explicit State Machine

```
created → requires_action → processing → succeeded
                ↓              ↓
              failed        failed
                ↓
            canceled
```

Every payment has exactly one of 6 states. Transitions are validated — you can't go from `created` directly to `succeeded`. Only webhooks can move payments to terminal states (`succeeded`, `failed`, `canceled`).

### 3. Idempotent Webhook Processing

```sql
-- Database constraint prevents duplicate processing
CREATE TABLE processed_webhook_events (
    stripe_event_id VARCHAR(255) UNIQUE NOT NULL
);
```

Stripe may send the same webhook multiple times. Database constraints ensure exactly-once processing, even with concurrent requests across multiple server instances.

### 4. PCI Compliance by Design

```
Frontend (Stripe.js) → Stripe API → Backend (IDs only)
         ↑                              ↓
    Card data                    No card data
    never leaves                 ever touches
    browser                      our servers
```

By using Stripe Payment Element, card data goes directly to Stripe. Our backend never sees card numbers, CVVs, or expiry dates. This qualifies us for the simplest PCI compliance level (SAQ A).

---

## Quick Start

### Prerequisites

- **Node.js 20+** — [Download](https://nodejs.org/)
- **pnpm** — `corepack enable && corepack prepare pnpm@latest --activate`
- **Stripe CLI** — [Install guide](https://stripe.com/docs/stripe-cli)
- **Stripe test account** — [Sign up](https://dashboard.stripe.com/register)
- **Supabase account** — [Sign up](https://supabase.com/dashboard) (database is pre-configured)

### Setup

```bash
# Clone and install
git clone <repository-url>
cd stripe-payment-lifecycle
pnpm install

# Configure environment
cp .env.example .env
# Edit .env with:
#  - Your Stripe test keys from https://dashboard.stripe.com/test/apikeys
#  - Your Supabase database password and service role key
#    (Get from https://supabase.com/dashboard/project/vnfggcntplxvcfdihjmx/settings)

# Build packages
pnpm build

# Start API and Frontend (database is already live on Supabase)
pnpm dev

# In another terminal, forward webhooks
stripe listen --forward-to localhost:3001/webhooks/stripe
# Copy the webhook secret (whsec_...) to your .env file
# Restart API to pick up the new secret
```

### Access

- **Frontend**: http://localhost:3000
- **API**: http://localhost:3001
- **Database**: Supabase (hosted) — `https://vnfggcntplxvcfdihjmx.supabase.co`
- **Supabase Dashboard**: https://supabase.com/dashboard/project/vnfggcntplxvcfdihjmx

---

## Project Structure

See [docs/architecture.md](docs/architecture.md) for detailed package documentation.

```
stripe-payment-lifecycle/
├── apps/
│   ├── api/                 # Express backend
│   └── frontend/            # Next.js + Tailwind CSS + shadcn/ui
├── packages/
│   ├── payment-domain/      # State machine & validation
│   ├── database/            # PostgreSQL repositories (Supabase)
│   ├── stripe-client/       # Stripe SDK wrapper
│   └── webhook-handler/     # Webhook processing
├── infra/
│   └── docker/              # Dockerfiles for API/Frontend
├── docs/                    # Documentation
└── README.md
```

---

## Features

### Interactive Homepage
- **API Health Check** — Real-time server status with automatic wake-up for free tier deployments
- **Technical Showcase** — Visual architecture diagrams, state machine visualization, and tech stack overview
- **Features Highlight** — Key capabilities: webhook-driven truth, explicit state machine, PCI compliance
- **Live Demo Flow** — Animated payment flow visualization
- **Responsive Design** — Mobile-first Tailwind CSS with smooth Framer Motion animations

### Production-Ready UI
- **shadcn/ui Components** — Accessible, customizable components (Button, Card, Badge)
- **Tailwind CSS** — Utility-first styling with custom Stripe branding
- **Framer Motion** — Smooth animations and page transitions
- **API Gate** — Prevents checkout access until API is confirmed ready

---

## Documentation

- **[Architecture Overview](docs/architecture.md)** — System design, package structure, architectural decisions
- **[Payment States](docs/payment-states.md)** — State machine, transitions, event handling
- **[Security Model](docs/security-model.md)** — PCI compliance, webhook security, secret management
- **[API Reference](docs/api-reference.md)** — Endpoint documentation, request/response formats
- **[Development Guide](docs/development.md)** — Local setup, testing, troubleshooting

---

## Testing Payments

Use [Stripe test cards](https://stripe.com/docs/testing):

| Scenario | Card Number | Description |
|----------|-------------|-------------|
| **Success** | `4242 4242 4242 4242` | Payment succeeds immediately |
| **3D Secure** | `4000 0025 0000 3155` | Requires 3DS authentication |
| **Declined** | `4000 0000 0000 0002` | Card declined by issuer |
| **Insufficient Funds** | `4000 0000 0000 9995` | Insufficient funds error |

---

## Who This Is For

This repository is for:

- **Teams building Stripe integrations** — Learn correct patterns from day one
- **Senior developers** — Reference implementation for payment system design
- **Code reviewers** — See what production-quality Stripe code looks like
- **Upwork clients** — Evaluate payment system expertise

**If you're reviewing this from Upwork**: This project reflects how I design real payment systems used in production. Every decision is documented with reasoning.

---

## License

MIT

---

**Remember**: This is not just a demo. This is a blueprint for building payment systems correctly.
