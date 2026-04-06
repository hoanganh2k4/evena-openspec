# Evena — Spec Index

**Version:** 2.0 · **Date:** 2026-04-05

## Structure

```
specs/
├── README.md               ← this file
├── rules.md                ← global business rules (source of truth)
├── sse-flow-spec.md        ← SSE channels, events, integrity rules
├── sse-checklist.md        ← SSE review checklist
│
├── objects/                ← data model, entities, schemas
│   ├── README.md           ← entity relationship diagram
│   ├── auth/               user · role · pending-registration
│   ├── organization/       organization · organization-member
│   ├── event/              event · category · venue · event-media
│   ├── ticket-type/        ticket-type
│   ├── order/              order · order-item · ticket · scan-log
│   ├── payment/            payment
│   ├── flexpass/           ticket-transfer
│   └── monitor/            activity-log
│
├── enums/                  ← all enums with state machines
│   ├── README.md           ← enum index
│   ├── user-status.md
│   ├── event-status.md
│   ├── ticket-type-status.md
│   ├── ticket-status.md
│   ├── order-status.md
│   ├── payment.md          PaymentStatus · PaymentProvider · CurrencyMinimum
│   ├── scan-log-result.md
│   ├── organization-role.md
│   ├── ticket-transfer-status.md
│   └── audit.md            LogAction · EntityType · ActorRole
│
├── api/                    ← REST endpoints by domain
│   ├── README.md           ← conventions, auth, error format
│   ├── auth.md             /api/auth · /api/users
│   ├── organization.md     /api/organizations · /api/invitations
│   ├── event.md            /api/events · /api/categories · /api/venues
│   ├── ticket-type.md      /api/events/{id}/ticket-types
│   ├── order.md            /api/orders · /api/orders/tickets
│   ├── payment.md          /api/payment
│   ├── flexpass.md         /api/flexpass
│   └── monitor.md          /api/activity-log
│
└── services/               ← service layer business logic
    ├── README.md           ← service index
    ├── auth-service.md
    ├── organization-service.md
    ├── event-service.md
    ├── ticket-type-service.md
    ├── order-service.md
    ├── ticket-service.md
    ├── payment-service.md
    ├── flexpass-service.md
    ├── notification-service.md
    └── scheduler.md
```

## Quick links

| Need to know | Go to |
|---|---|
| Business rules | [rules.md](rules.md) |
| Entity fields / relationships | [objects/README.md](objects/README.md) |
| State machines | [enums/README.md](enums/README.md) |
| API endpoints | [api/README.md](api/README.md) |
| Service logic | [services/README.md](services/README.md) |
| SSE events | [sse-flow-spec.md](sse-flow-spec.md) |
| FlexPass transfer | [objects/flexpass/ticket-transfer.md](objects/flexpass/ticket-transfer.md) · [services/flexpass-service.md](services/flexpass-service.md) |
| QR signing | [objects/order/ticket.md](objects/order/ticket.md) · [services/ticket-service.md](services/ticket-service.md) |
