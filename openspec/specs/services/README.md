# Services — Index

Business logic layer. Each file documents one Spring `@Service` class.

| File | Class | Responsibility |
|---|---|---|
| [auth-service.md](auth-service.md) | `AuthService` | Registration, login, tokens, email verify, password reset |
| [organization-service.md](organization-service.md) | `OrganizationService`, `OrganizationMemberService` | Org CRUD, member invitations |
| [event-service.md](event-service.md) | `EventService` | Event CRUD, publish, cancel, search |
| [ticket-type-service.md](ticket-type-service.md) | `TicketTypeService` | TicketType CRUD, activate, deactivate |
| [order-service.md](order-service.md) | `OrderService`, `OrderInternalService` | Order creation, checkout, cancel, ticket issuance |
| [ticket-service.md](ticket-service.md) | `TicketService`, `QRCodeService` | Scan, validate, check-in, QR signing |
| [payment-service.md](payment-service.md) | `PaymentService` | Initiate, callback, refund |
| [flexpass-service.md](flexpass-service.md) | `FlexPassService` | Listing lifecycle, atomic transfer, QR rotation |
| [notification-service.md](notification-service.md) | `SSENotificationService`, `EmailService` | Realtime events, transactional emails |
| [scheduler.md](scheduler.md) | `EventScheduler` | Auto status transitions |
