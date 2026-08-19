# Billing system for the visually impaired
# Voice Automated Billing System

A point of sale system for a cafeteria whose counter is operated by a visually impaired staff member. Built for the Blind Relief Association.

## Why this exists

Conventional POS software assumes a sighted operator: a grid of item buttons, a cart checked at a glance, a receipt read on paper. None of that is usable by a blind operator, so the practical outcome was that the cafeteria ran without a billing system at all. Orders were handled informally and no transaction record existed, which meant no visibility into what sold, what stock remained, or what a day took in.

This system replaces the visual interaction layer with a spoken one. The operator can take an order, correct it, and close a bill without ever looking at the screen. Making the counter accessible is also what produced the cafeteria's first sales and inventory record.

## Operator flow

1. Press space (or the on-screen button) to start voice input.
2. Speak the item and quantity, for example "veg sandwich 2". The system confirms the addition aloud and speaks the running total.
3. If the requested quantity exceeds stock, the system says how many are available and asks whether to bill that quantity instead. It waits for a spoken yes or no.
4. Say "generate bill" to produce a PDF bill, or "stop" to hear the total without closing.

Every state change is spoken. The on-screen bill preview exists for the customer, not the operator.

## Accessibility decisions

These came out of watching the system used in place rather than from the spec.

**Screen reader arbitration.** The operator runs NVDA as a system-wide screen reader. The app has its own text to speech engine, and both speaking at once makes the app unusable. NVDA is terminated when the app launches and restarted on exit via an `atexit` hook, so quitting the app never leaves the operator without a screen reader.

**Keyboard first.** The window opens fullscreen and every primary action has a single-key binding: space for voice input, `f` for the daily sales report, `j` to exit. Nothing requires locating a control on screen.

**Noise handling.** `adjust_for_ambient_noise` runs before every listen because a cafeteria floor is loud and the noise floor shifts through the day.

**Confirmation before commitment.** Stock shortfalls trigger a spoken yes/no prompt rather than a silent failure or a partial fill.

## Inventory model

Selling an item depletes both the item and its constituent ingredients. An `ingredients` table maps each sellable item to the raw stock it consumes, so selling two sandwiches also draws down bread and filling. This is what allows stock reporting to reflect actual consumption rather than only finished-goods count.

## Data model

```sql
items       (item_id, item_name, price, stock)
ingredients (item_id, ingredient_name, quantity)
sales       (item_id, quantity, total_price, sale_date)
```

## Reporting

- **Bill (PDF):** generated per order via ReportLab and opened automatically.
- **Daily sales (Excel):** per-item quantity and revenue for the current date, with a total row, via a join across `sales` and `items`.
- **Stock report (Excel):** current stock level for every item.

Administrative actions (add item, add stock) are available from the side panel and are also confirmed by voice.

## Stack

Python, `speech_recognition` with the Google recognizer, `pyttsx3` for speech output, `customtkinter` for the interface, MySQL for persistence, `pandas` and `openpyxl` for reports, `reportlab` for bills.

## Setup

```bash
pip install speech_recognition pyttsx3 mysql-connector-python customtkinter pandas openpyxl reportlab psutil
```

Create the `cafeteria` database and the three tables above, then set the database credentials as environment variables (see Known limitations) and run:

```bash
python blind_relief_billing_system.py
```

Requires a working microphone and an internet connection, since speech recognition is performed by a hosted API.

## Known limitations

Documented honestly rather than left for someone else to find.

**Sales are recorded on item addition, not on bill generation.** `record_sale` fires the moment an item is added to the order, so an order abandoned mid-flow still writes to `sales` and still decrements stock. This overstates revenue and understates inventory. The fix is to buffer the order in memory and commit the whole thing as a single transaction when the bill is generated, with a rollback on failure.

**Database credentials are hardcoded.** They belong in environment variables, and the application should connect as a scoped user rather than root.

**Speech recognition is cloud-dependent.** An outage or a dropped connection takes the counter down. An offline recognizer such as Vosk would remove that dependency at some cost to accuracy.

**Single terminal, single operator.** There is no concurrency handling on stock updates. Fine at one counter, incorrect at two.

**Item names must match the database exactly.** Recognition of an unlisted or misheard name is rejected outright rather than matched to the nearest known item, which pushes the operator into repeating themselves. Fuzzy matching against the item list would reduce retries.

## License

MIT
