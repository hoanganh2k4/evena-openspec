# Enum: UserStatus

| Value | Meaning |
|---|---|
| `ACTIVE` | Normal; can log in |
| `INACTIVE` | Deactivated by admin |
| `SUSPENDED` | Temporarily suspended |
| `PENDING_VERIFICATION` | Registered but email not verified |

## Transitions

```
PENDING_VERIFICATION → ACTIVE     (email verified)
ACTIVE → INACTIVE                 (admin)
ACTIVE → SUSPENDED                (admin)
INACTIVE → ACTIVE                 (admin)
SUSPENDED → ACTIVE                (admin)
```
