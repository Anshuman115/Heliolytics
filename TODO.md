# Heliolytics — What I Need to Figure Out

> **Status:** Checklist. Will be updated as I discover answers.

## Phase 1: Connect to the Strap

- [ ] What BLE library to use in Flutter?
- [ ] What are the Service UUIDs?
- [ ] What are the Characteristic UUIDs?
- [ ] How does the ECDH key exchange work?
- [ ] What does the authentication flow look like?

## Phase 2: Discover the Data

- [ ] What data types does the strap send?
- [ ] What's the resolution of each data type? (per-second? per-minute?)
- [ ] What's the format of each data type?
- [ ] How do I request historical data?
- [ ] What's the sync window?

## Phase 3: Store the Data

- [ ] What database to use?
- [ ] How to structure the schema? (wide vs. long format)
- [ ] How to handle deduplication?
- [ ] How to handle sync conflicts?

## Phase 4: Build the UI

- [ ] What does the dashboard look like?
- [ ] What metrics to show?
- [ ] How to visualize trends?
- [ ] What charts to use?

## Phase 5: Analytics

- [ ] What algorithms to implement?
- [ ] What research papers to reference?
- [ ] How to validate the algorithms?
- [ ] What baselines to use?

## Reference: Questions to Answer from the Strap

When I first connect, I need to find out:

1. **Activity data (0x01)** — What's the actual resolution? Per-second or per-minute?
2. **HRV data (0x49)** — How often does it come? What's the format?
3. **SpO2 data (0x25)** — When does it measure? Daytime only? During sleep?
4. **Sleep data (0x48)** — Is this pre-computed by Zepp's cloud, or raw data?
5. **Workout data (0x05)** — How is it structured?

## Notes

- Don't trust pre-existing parsers — verify from raw bytes
- Don't pre-design schemas — let the data tell us what to store
- Don't pre-design analytics — research after seeing real data
