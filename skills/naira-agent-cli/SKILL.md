---
name: naira-agent-cli
description: Use the NairaAgent CLI to transfer Naira, pay bills (electricity, data bundles, airtime, TV), and run secure session-based workflows for bulk, large, or scheduled payments.
---

# Naira Payment Skill

This skill defines a safe, repeatable workflow for AI agents using the `naira` CLI for transfers and bill payments.

## Install and Run

Preferred (global):

```bash
npm i -g @naira-agent/cli
naira --help
```

## Prerequisites

- CLI is installed and available as `naira`
- User has completed login flow (`naira login`).

## Agent Rules (Required)

1. Always resolve recipients before first transfer to a new account.
2. Never assume bank codes; fetch with `naira banks`.
3. If any command returns an OTP-pending response, pause and request OTP from the user.
4. Do not retry payment commands blindly after `pending_otp`; use `naira confirm`.
5. For bulk, large, or long-running tasks, create a labeled session and pass `--session-token` explicitly.
6. Revoke task sessions when done.
7. For scheduled payments, the scheduler/orchestrator triggers CLI execution at due time; this CLI does not provide a built-in cron scheduler.

## Core Commands

### Authentication

- `naira login`
- `naira logout`

### Read Operations

- `naira balance`
- `naira profile` (includes KYC status details in profile output)
- `naira banks`
- `naira history [-l <limit>] [-o <offset>]`
- `naira bills categories`
- `naira bills billers -c <AIRTIME|DATA|TV|ELECTRICITY>`
- `naira bills plans -c <AIRTIME|DATA|TV|ELECTRICITY> -b <billerId>`

### Payment Operations

- Resolve account:
  `naira resolve -b <bankCode> -a <accountNumber>`
- Transfer:
  `naira transfer -b <bankCode> -a <accountNumber> -m <amount> [-r <reason>] [-s|--session-token <token>]`
- Bill payment:
  `naira bills pay -c <category> -b <billerId> [-p <planId>] -t <target> -a <amount> [-s|--session-token <token>]`

Supported bill categories: `AIRTIME`, `DATA`, `TV`, `ELECTRICITY`.

If no valid session token is supplied while OTP confirmation is enabled, payment commands may return `pending_otp` and a reference (`TX-...` or `BILL-...`).

### Account Operations

- `naira accounts list`
- `naira beneficiary list`
- `naira beneficiary add -n <name> -a <account> -b <bankCode> [--bank-name <name>]`

### OTP and Confirmation

- OTP setting:
  `naira settings otp status|enable|disable`
- Confirm pending transfer/bill/session request:
  `naira confirm -r <reference> -c <otp>`

### Session Workflow (Duration-Based, Multi-Session)

Sessions are explicit and can exist in parallel.

- Request:
  `naira session request [-l <label>] [-d <minutes>]`
- Check status:
  `naira session status`
- List sessions:
  `naira session list`
- Revoke one session:
  `naira session revoke -i <sessionId>`
- Revoke all sessions:
  `naira session revoke`
`
Important:

- CLI does not auto-attach session tokens to transfer/bill operations.
- Agent must pass `--session-token` per payment command when session-based execution is desired.

## Examples

### 1. Single Transfer With OTP

1. `naira transfer -b 100004 -a 9066269233 -m 100 -r "Gift"`
2. If response is pending: ask user for OTP.
3. `naira confirm -r TX-XXXX -c <otp>`

### 2. Bulk Payments With Explicit Session

1. `naira session request -l "bulk-payouts" -d 90`
2. Confirm: `naira confirm -r SES-XXXX -c <otp>`
3. Capture token from confirmation output or inspect `naira session list`.
4. Use token on every payment:
   `naira transfer -b 100004 -a 9066269233 -m 1000 --session-token <token>`
5. Revoke at end:
   `naira session revoke -i <sessionId>`

### 3. Bill Payment With Discovery Flow

1. `naira bills categories`
2. `naira bills billers -c DATA`
3. `naira bills plans -c DATA -b <billerId>`
4. `naira bills pay -c DATA -b <billerId> -p <planId> -t <phone> -a <amount>`
5. If pending, confirm: `naira confirm -r BILL-XXXX -c <otp>`

### 4. Scheduled Payment Pattern (Agent + Scheduler)

1. Scheduler stores payment instruction and due time.
2. At due time, agent runs:
   `naira transfer -b <bankCode> -a <accountNumber> -m <amount> --session-token <token>`
3. If token is expired or OTP is required, pause and request user confirmation (`naira confirm` flow).
