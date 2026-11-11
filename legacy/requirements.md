# Banking & Recordkeeping

## Situation

The goal is a simple, maintainable recordkeeping system — primarily for tax prep, secondarily for family financial decision-making. <!--Are we sure about this? There's the general sense that we don't know how much money we actually have, or how quickly it would run out if income dropped off, or whether we are even truly making ends meet with income vs. biting into savings every month. So I think the financial decisionmaking (maybe just awareness?) is equal in priority to tax prep.--> Records in Quicken went stale around March 2026 and everything since then has been seat-of-the-pants.

**Background:** Quicken worked but became a perfectionism trap — categorizing transactions and reconciling statements nearly every evening. The fix was to scale back: retirement funds moved to an annual check-in, wife took over her own credit cards. That left nothing in the middle.

**What needs tracking** covers four distinct entities:
1. **Personal** — individual accounts, spending, income
2. **Joint / Couple** — joint accounts, shared household expenses, US joint filing <!--The US joint filing will require data from all these entities, as will my Canada tax filing.-->
3. **Rental property** — self-managed, runs through personal accounts, was tracked in Quicken
4. **Shelbourne Tech (sole prop)** — effectively personal income for this tax year; Freshbooks handles invoicing and expense recording but has fallen behind; recently incorporated and the corporation will be managed separately going forward

**2026 has been a year of transition:** left GitLab employment (no more personal RSUs or ESPP, though Amor still has stock plan and ESPP income from her employer), started STS as a sole proprietor, opened and closed accounts, received some income through unusual channels, and may have already made a mess of business banking. Wife's accounts are otherwise out of scope — except one card where personal charges sometimes appear, which is a blind spot.

**Tax complexity is the main driver.** We file in both the US and Canada; the business and rental property add schedules; this year's prep consumed several complete weekends even with the accountant doing the actual forms. The accountant needs a full packet — dozens of reports and spreadsheets. The two hardest parts: (1) FBAR and Canadian equivalent forms that require the highest balance for every account during the year, and (2) brokerage account activity for equity plans (RSUs, ESPP, stock plans). Tax prep needs to be ready by January, which means organizing records now.

**Upcoming decisions that need financial visibility:**

<!--Not really decisions, these are events/facts-->

- Sell the rental house and buy a home (next year)
- One child starting university
- Possibly months without a contract

**Preferred approach:** A monthly-cycle model — download statements, process and categorize, generate summaries — built locally with Python, YAML, SQLite, and Git. No nightly sync, no Mint-style online tools. The goal is clean, organized data; reporting is the easy part once the data is there. Acceptable threshold for ignoring a transaction: ~$100 CAD. <!--I'd like LLM evaluation of my logic here - does the thinking seem reasonable and sound?-->

**What already exists in this repo (keep as-is):**
- Retirement portfolio tracking (annual, YAML + Python report)
- STS income distribution plan and event log
- Exchange rate data from Bank of Canada

---

## Open Questions

<!--Most of these questions are about specific accounts but I think we need to design a system first, then figure out what accounts to load and which data to load. For the system-->

### Accounts inventory
1. Can you list every account open at any point in 2026, including closed ones? (Institution, currency, type.) This is the foundation for FBAR/T1135 tracking.
2. For each account: can you download a statement or transaction export (CSV, OFX, PDF)? Which are painful or impossible?
3. Which accounts are personal, joint with Amor, or business-only?

### Business banking
4. What accounts received STS client payments before the current Wise setup? Were any payments made to personal accounts that are really business income?
5. Are there business expenses paid from personal accounts (or vice versa) that need to be untangled?
6. What's the current state of Freshbooks — how far behind are expenses, and is the invoice record complete for all of 2026?

### Rental property
7. How is rental income received, and which account does it land in?
8. Are rental expenses (maintenance, insurance, mortgage interest, property tax) paid from a dedicated account or mixed in with personal spending?

### Amor's accounts
9. Which card does Amor hold where personal charges sometimes appear? Are those charges easy to identify on the statement?
10. Does Amor track her own equity/ESPP activity, or will that need to be gathered for filing?

### Categorization
11. Are there spending categories that matter for decisions or tax, or is the main goal just to tag each transaction as business / rental / personal / ignore?
12. Are there any cash transactions that matter, or is everything electronic and traceable?

### Highest-balance tracking (FBAR / T1135)
13. Is last year's FBAR filing handy? It has the account list and highest-balance figures — a useful starting template.
14. Do Schwab and Fidelity produce year-end statements with highest-balance figures, or does that need to be tracked during the year?
