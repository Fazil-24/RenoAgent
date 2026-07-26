# Build Log — Reno Quote Audit Agent

Kept as I went, not reconstructed afterward. Timestamps approximate.

---

## Day 1 — Scoping

Spent the first stretch just deciding what to build. Assessment brief leaves it open (news digest, price watcher, listing aggregator, etc.) but I wanted something with a genuine gap, not a textbook example. First instinct was a general "cross-business synergy engine," but pulled back from that once I realized it was solving a problem that doesn't exist without a specific employer context — didn't want to build something ungrounded.

Landed on renovation quote comparison after researching what actually exists in the Dubai market. Found several platforms already doing "submit once, get multiple quotes" (Remodeling.ae, RenovationQuoteDubai, TaskRight, a few others) — so the base idea is not novel, and I don't want to pretend otherwise. What none of them seem to do is anything past collecting the quotes: no automated auditing of the quotes themselves, no automated negotiation. That's the actual gap I'm going after.

Checked whether "snap a photo, get an AI cost estimate" already exists too — it does, globally (Cost AI, HouseRemodelCost, RenoCalc — the last one explicitly serves Dubai). So the photo-to-estimate step is table stakes, not a differentiator. Repositioning the pitch: the photo step is the trigger, the differentiator is what happens after — audit, rank, negotiate, human-gate.

Decided against building a cross-vertical Alba Corp idea for the final submission (was originally exploring that angle) — this project needed to stand on its own, not depend on inside knowledge of one company's internal data.

## Day 1 (later) — Architecture planning

Sketched the full pipeline before touching n8n: photo → vision → vendor RFQ fan-out → PDF quote intake → price-benchmark search → audit → rank/negotiate → human approval → send/log. Deliberately designed for branching, fan-out, and a real HITL gate, since a straight-line flow wouldn't demonstrate much.

Decided early: mocks are fine for vendor contact data (no real vendor directory exists that would let me RFQ real businesses for a take-home), but the mechanics around them — RFQ dispatch, PDF parsing, audit logic, search grounding — should be as real as possible. Wrote this down explicitly so I don't quietly drift into mocking everything under time pressure.

## Day 1 — Environment setup

n8n no longer has a permanent free Cloud tier (checked current docs — only a 14-day trial now). Built locally via Docker first to avoid burning the trial clock while still learning the interface. Moved to Cloud once the core chain was stable, specifically for shareability.

Chose Cerebras for the LLM layer — gpt-oss-120b for text reasoning, gemma-4-31b for vision. Tested the vision model in the Cerebras playground first: one image analysis came back at 647 total tokens, well under half a second. Free tier gives roughly 1M tokens/day, so cost isn't a real constraint for a build-and-test cycle — but added a self-imposed daily cap on vision calls anyway (10/day), specifically to demonstrate cost-aware API usage rather than because it's a hard limit.

## Day 1 — First real bug: Cerebras rejects remote image URLs

Vision Analysis node failed immediately: `Remote image URLs are not supported; send images as data URIs`. Had to add a Download Image (HTTP GET, response format = File) → Convert to Base64 (Code node reading binary buffer, building a `data:image/...;base64,...` string) step before the actual vision call. Straightforward once diagnosed, but a good reminder not to assume every "OpenAI-compatible" API behaves identically to OpenAI's own multimodal format.

## Day 1 — Second bug: Gmail Trigger and binary attachments

Wanted the Vendor Reply Trigger to pull PDF attachments off vendor emails. First pass had "Simplify" toggled on — turns out that mode skips downloading attachment binaries entirely, so there was nothing for the PDF extraction step to read. Turned Simplify off, enabled "Download Attachments" explicitly, and it worked. Used `Object.keys($binary)[0]` as the binary field reference rather than hardcoding a name, since Gmail's attachment field naming isn't guaranteed consistent.

## Day 1 — Third bug: JSON body syntax errors from raw text

The PDF-quote-parsing LLM call kept failing with `Bad control character in string literal in JSON`. Root cause: I was hand-pasting an expression into a manually-typed JSON template, and the extracted PDF text contained raw line breaks that broke JSON syntax. Fixed by building the entire request body via a single `JSON.stringify({...})` expression instead — properly escapes anything the input text contains. Applied the same pattern to every other LLM call afterward rather than waiting to hit the same bug three more times.

## Day 1 — SerpAPI integration, and a token-limit lesson

Wired up SerpAPI for the audit step's price-reference search. First version forwarded the entire raw SerpAPI JSON (including base64 favicon images and search metadata) straight into the LLM prompt — immediately hit a `tokens per minute` rate limit on the free tier. Added a cleanup step that extracts just the `snippet` text fields before sending anything to the LLM. Obvious in hindsight; a good example of why raw API payloads need trimming before they hit a token-priced endpoint.

Considered live vendor discovery via SerpAPI too (real local contractor search results), and built it briefly — then removed it once I confirmed Google search results never surface email addresses, only phone/website. Adding a second real API (Hunter.io) just to bridge that one gap felt like complexity for a marginal credibility gain, so I kept vendor contact data as mock records instead and put the real-API budget into the price-benchmark search, which is harder to fake convincingly and actually feeds a decision.

## Day 2 — Generated the three vendor quote PDFs

Built three distinctly-branded sample PDFs (different layout, different color scheme, same underlying job) rather than three near-identical documents, specifically so the audit step would have something real to differentiate: one clean quote, one with an oversized 50% deposit, one with a vague "miscellaneous and contingency" line eating ~25% of the total. Wanted the demo to show three genuinely different audit outcomes, not the same flag three times.

## Day 2 — Audit logic

Wrote the actual audit rules: deposit percentage above 30% flags high risk, keyword-matched vague line items flag medium risk, totals implausibly below the search-grounded reference range flag a low-price warning. Tested against the three PDFs — correctly caught the 50% deposit and the vague line item, correctly left the clean quote unflagged. This is the piece of the workflow I'm most confident actually demonstrates judgment rather than just data-shuffling.

## Day 2 — AI Agent node, and two bugs

Built the ranking/negotiation step using n8n's native LangChain Agent node (not another plain HTTP call — wanted at least one point in the workflow doing genuine multi-step reasoning with tool-use potential, not just prompt-in-text-out).

Bug one: the Agent ran once per input item instead of once for the whole batch, giving three near-identical outputs for one decision. Fixed via the node's own "Execute Once" setting under Settings — missed it on first pass, went looking for a workaround before noticing it was already there.

Bug two: got a `404` from Cerebras's endpoint on the Chat Model sub-node. Traced it to "Use Responses API" being toggled on by default — that's a newer OpenAI-specific API shape that Cerebras's OpenAI-compatible layer doesn't implement. Turned it off, resolved immediately.

Decided explicitly **not** to give the agent a live email-sending tool. It drafts the ranking and counter-offer; it doesn't send anything itself. That's a deliberate boundary, not a missing feature — sending a real commitment to a real vendor is the one genuinely irreversible action in this whole pipeline, and that's exactly the kind of action that should sit behind a human approval, not full autonomy.

## Day 2 — HITL gate, and a URL bug that looked like a token bug

Built the Wait-node-based approval gate: agent drafts → email to homeowner with Approve/Reject links → workflow pauses on a Wait node → resumes on whichever link is clicked.

Hit an "invalid token" error clicking the approve link. Spent a while suspecting the resume token itself had expired or been generated wrong. Actual cause: my email template had two `?` characters in the URL (`{{ $execution.resumeUrl }}?action=approve`, where `resumeUrl` already included its own `?signature=...`) — the second `?` corrupted the whole query string, so the "signature" the server received was garbage. Fixed by joining with `&` instead. Good reminder that an "invalid token" error doesn't always mean the token logic is wrong — sometimes it means the URL never got there intact.

## Day 2 — Scoping the vacating branch out

Had planned a second scenario branch (move-out repair-vs-forfeit-deposit decision), sharing the same vendor/audit pipeline as renovation. Partway through wiring it, noticed both branches were converging into the exact same node immediately, with the only real behavioral difference being one small comparison at the very end — meaning the early IF-branch wasn't actually branching anything meaningful, just relabeling one path as two. Rather than force extra build time to justify a decorative fork, made the call to descope the vacating logic entirely and keep this submission tightly focused on renovation, fully built and tested end to end. Logged as a Phase 2 item in the README rather than silently dropped.

## Day 2 — Cleanup pass

Found and fixed a duplicate Google Sheets read (VendorsMock was being read twice — once for RFQ dispatch, once again just to look up an email address for the negotiation step, when the first read's data was already available in memory). Removed the redundant second read.

Found the vision-quota IF node's False branch was dangling with no fallback — added a simple notification email so a quota-exceeded case degrades gracefully instead of just silently doing nothing.

Confirmed a real SerpAPI key had ended up hardcoded in a query parameter rather than a proper credential — rotated the key and moved it into a Header Auth credential before finalizing anything for submission. Worth naming this explicitly: a workflow JSON export is a deliverable file, and a hardcoded key in it is a real leak, not a cosmetic issue.

## Day 2 — Final end-to-end test

Ran the complete chain: photo webhook → vision analysis → 4 vendor RFQs sent with attachments → 3 vendor PDF replies processed live via the Gmail-triggered chain → price-benchmark search → audit correctly flagging 3 different risk profiles → agent ranking and drafting a counter-offer → HITL email → approved → counter-offer sent → confirmation logged. Confirmed the rejection path separately by re-running with the reject link instead.

Noted for the README: the "wait and re-check for vendor replies" loop only reliably resumes on its own timer when the workflow is genuinely published and triggered via its Production URL, not while testing manually via "Listen for test event." Verified this once in published mode; for the actual demo, pre-populating the quotes sheet before triggering avoids depending on that live timing at all.
