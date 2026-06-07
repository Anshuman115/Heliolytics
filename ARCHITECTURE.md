# Heliolytics — Architecture (To Be Filled In)

> **Status:** Placeholder. Will be filled in as we discover the data and design decisions.

## High-Level Overview

```
┌──────────────┐
│  Helio Ring  │
└──────┬───────┘
       │ BLE
       v
┌──────────────┐
│  Mobile App  │
└──────┬───────┘
       │ ?
       v
┌──────────────┐
│  Database    │
└──────────────┘
```

## Decisions to Make

- [ ] **Tech stack** — What language/framework for mobile? What for backend (if any)?
- [ ] **Data flow** — Mobile-only? Or add a backend?
- [ ] **Storage** — Local DB? Cloud? Both?
- [ ] **Data format** — What does the strap actually send? (To be discovered)
- [ ] **Analytics** — What metrics to compute? (To be researched)
- [ ] **UI** — What does the dashboard look like? (To be designed)

## Current Status

**Just starting.** First step: build a basic app that can connect to the strap and see what data it sends.

## Reference: What Already Exists

I have an existing Android Kotlin app (Helio) that already does BLE + parsing. I can use it as a reference for the protocol, but I'm building this from scratch to make my own decisions.

## Next Steps

1. Create new repo
2. Set up Flutter project
3. Add BLE scanning
4. Connect to the ring
5. Dump raw bytes
6. Document what I find
