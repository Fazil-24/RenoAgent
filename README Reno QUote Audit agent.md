# Reno Quote Audit Agent

An n8n workflow that turns a photo of a home renovation issue into an audited, negotiated, human-approved vendor decision — not just a quote comparison tool.

---

## 1. What This Is, and Why It's Useful

Homeowners in Dubai (and anywhere, really) getting renovation work done today go through a manual, trust-based process: describe the job, get a few quotes back over WhatsApp or email, eyeball the totals, and pick one. I looked at the existing local platforms doing "get multiple renovation quotes" (Remodeling.ae, RenovationQuoteDubai, TaskRight, and a few others) before building this, and every one of them still runs the vendor-matching and quote-comparison step manually on the human side — a person reads the form, a person matches contractors, a person compares the PDFs by eye. None of them automate the part that actually requires judgment: is this quote safe, and is it a fair price?

Snap-a-photo AI cost estimation itself isn't new either — tools like RenoCalc and Cost AI already do "upload a photo, get an instant estimate," and RenoCalc explicitly serves the Dubai market. So I didn't try to claim novelty on the photo-to-estimate step. What I found nobody doing is connecting that first step to an automated, multi-vendor RFQ pipeline that normalizes real quotes, audits them for hidden risk (not just price), and only commits to a vendor after a human explicitly approves an AI-drafted negotiation. That's the gap this workflow targets.

**In one line:** snap a photo → get real quotes from multiple vendors → an AI agent audits and ranks them for both price and risk → it drafts a negotiation with the best one → a human approves before anything is actually sent.

---

## 2. Architecture & Why I Built It This Way

I've tried to document not just *what* each piece does, but *why* I made that call, since a lot of these decisions were genuine trade-offs rather than obvious defaults.

### n8n Cloud, not self-hosted
n8n dropped its permanent free Cloud tier at some point — the only genuinely free option now is self-hosting the Community Edition via Docker. I actually built the first several nodes locally to avoid burning the 14-day Cloud trial while still learning the tool. Once the core logic was stable, I moved to Cloud specifically so the workflow is easy to share and open directly — a local instance isn't reachable by anyone but me.

### Cerebras as the LLM provider, two different models for two different jobs
I used `gemma-4-31b` for the vision/photo-analysis step and `gpt-oss-120b` for every text-reasoning step (parsing, auditing, ranking, negotiation drafting). This wasn't arbitrary — Cerebras's inference speed is genuinely fast (I measured a full image analysis at under half a second and under 400 total tokens in testing), and their free tier gives roughly 1M tokens/day, which comfortably covers a full build-and-test cycle without real cost. Splitting vision and text onto two models rather than forcing one multimodal model to do everything also let me self-impose a strict daily cap specifically on the *vision* calls (see below), since that's the more expensive, less frequently-needed call in this pipeline.

### A self-imposed daily quota guard on vision calls
Before every vision-analysis call, the workflow checks a running daily counter against a limit (currently 10/day) stored in a Google Sheet, and gracefully notifies the user instead of erroring out if the limit is hit. This isn't a hard API constraint — Cerebras's actual daily token allowance is far higher — it's a deliberate demonstration of cost-aware engineering: any real production system calling a metered API needs a budget guard before the call, not after a bill shows up. The number itself (10) is a demo-scale placeholder; in production it would scale with the actual plan's real allowance.

### Real search grounding for pricing (SerpAPI), not a hardcoded reference table
The audit step needs a market-rate benchmark to judge whether a vendor's line items are inflated. I could have hardcoded a price table, but that's not defensible or current. Instead, the workflow does a live SerpAPI search for renovation cost figures relevant to the detected trade category, strips the results down to just text snippets (see the note on token limits below), and has an LLM extract a usable price range with reasoning — which I watched it do correctly, including rejecting an irrelevant per-hour electrical rate in favor of the actual per-sqft renovation figures. This is the one external data source I was most deliberate about keeping "real" rather than mocked, since it's the piece an interviewer is most likely to probe.

*Cost/performance note:* the first version of this call sent the *entire* raw SerpAPI JSON response (including base64 favicon blobs and search metadata) to the LLM and blew through the free-tier tokens-per-minute limit in one call. I added a cleanup step that extracts only the text snippets before the LLM call — a good example of why raw API payloads shouldn't be forwarded to an LLM without pruning first.

### Vendor contact data is mocked; vendor discovery and RFQ dispatch mechanics are real
I initially built live vendor discovery via SerpAPI (real Google search results for local contractors). I removed it, deliberately, once I realized email addresses are never present in search results — Google surfaces phone/website, not email — so I'd have needed a second real API (Hunter.io) just to bridge that gap, adding real complexity for a small credibility gain. I chose instead to keep vendor identity/contact data as clearly-labeled mock records, and spend that complexity budget on the parts of the pipeline that are fully real: the RFQ dispatch mechanics, the PDF quote intake, the audit logic, and the negotiation agent. This is a scoping decision, not an oversight, and I'd rather state that plainly than imply the vendor list is a live directory.

### PDF quote intake is genuinely live, not simulated
Vendors reply to the RFQ email with an actual PDF quote attachment. A separate, independent Gmail-triggered chain polls for these replies, extracts the PDF's text (via n8n's built-in PDF extraction, not an LLM vision call — cheaper and more reliable for text-based PDFs), and has an LLM structure it into clean pricing data, which gets logged to a sheet. I generated three distinctly-branded sample vendor PDFs (different layout, different pricing, different quirks — one clean, one with an oversized deposit, one with a vague line item) to drive this realistically rather than testing against one trivial document.

*A real bug worth naming:* the first version of the PDF-parsing LLM call failed with a JSON syntax error, because the extracted PDF text contained raw line breaks that broke a hand-typed JSON template. The fix was to build the whole request body via `JSON.stringify()` in a single expression rather than pasting a JS-templated variable into hand-written JSON — this correctly escapes anything the PDF text might contain, and I applied the same fix everywhere else a similar pattern appeared.

### A real audit layer — the actual differentiator
Rather than just comparing three totals, `Audit Quotes` parses each vendor's line items and checks for: (a) a deposit request above a sane threshold (30%), (b) vague, non-itemized line items using keyword matching ("miscellaneous," "contingency," etc.), and (c) a total price implausibly below the search-grounded reference range. This is the one piece of logic that directly answers the real fear this market has around upfront-deposit scams and hidden markup — and it's demonstrably working: across my three test vendors, it correctly flagged a 50% deposit as high-risk, a vague contingency line as medium-risk, and left the clean quote unflagged.

### A genuine AI Agent node, not just another LLM API call
Every other LLM interaction in this workflow is a plain HTTP call to Cerebras's chat completions endpoint — useful, but not "agentic" in any meaningful sense; it's just a fixed prompt in, fixed structure out. The vendor ranking and negotiation-drafting step specifically uses n8n's native LangChain Agent node, backed by the same Cerebras model via an OpenAI-compatible connector, because this is the one point in the workflow where I wanted the model actually reasoning over a full data set and producing a judgment call (rank by price *and* risk, decide whether negotiation is warranted, draft accordingly) rather than transforming one input into one output.

*A real bug worth naming:* the Agent node initially ran once *per item* instead of once for the whole batch, producing three near-identical outputs for one decision — wasteful, not wrong. The fix was the node's own "Execute Once" setting, which I'd missed on first pass. Also hit a `404` from Cerebras's endpoint the first time, caused by the "Use Responses API" toggle defaulting on — that's an OpenAI-specific newer API shape that Cerebras's compatibility layer doesn't implement; turning it off resolved it immediately.

### Human-in-the-loop gate before the one irreversible action
The agent drafts a counter-offer, but never sends it directly — I deliberately did not give it a live email-sending tool. Instead, its ranking and draft are emailed to the homeowner with Approve/Reject links, and the workflow pauses on an n8n Wait node (resuming via its own generated webhook URL) until a decision is made. Only on approval does a separate node actually send the counter-offer and log the outcome; on rejection, it just logs that and stops. I made this call specifically because sending a real commitment to a real vendor is the highest-stakes, least reversible action in the entire pipeline — the same reasoning that governs why, say, an autonomous emergency-response agent should confirm before dispatching a real call to authorities. Autonomy is fine for reasoning and drafting; a human should own the send.

*A real bug worth naming:* my first version of the approve/reject links had two `?` in the URL (`...resumeUrl?signature=...?action=approve`), which silently corrupted the signature token and threw a confusing "invalid token" error that had nothing to do with the token actually being wrong. Fixed by joining the second parameter with `&` instead of a second `?` — a good reminder that a broken URL and an invalid token look identical from the error message alone.

### The vacating/deposit-recovery branch was descoped
I originally planned a second workflow branch — for the "move-out repair vs. forfeit deposit" scenario — sharing the same vendor/audit pipeline. Partway through building it, I realized the only real behavioral difference between the two scenarios was one small comparison happening at the very end, which made an early IF-branch splitting the flow in two feel like decoration rather than real architecture. Rather than force a bigger vacating-specific build to justify the branch, I made the deliberate call to descope it and keep this submission focused on the renovation use case, which is fully built, tested, and audited end to end. Noted below as a Phase 2 item.

### Loop-and-wait for vendor replies, without a hard timeout
Before auditing, the workflow checks whether at least 3 vendor quotes have arrived; if not, it waits 60 seconds and re-checks, looping until they do. There's no retry cap or timeout notification on this loop — deliberately, since in my actual usage pattern I'm sending vendor replies within a couple of minutes of triggering the RFQ, and adding a full retry-limit-plus-notification system would have added real complexity for a demo-only edge case. Noted below as a known simplification.

---

## 3. Node-by-Node Walkthrough

**Intake & Vision**
| Node | What it does |
|---|---|
| RFQ Trigger | Webhook receiving the homeowner's photo URL, scenario details, and contact info |
| Get Data / vision quota | Reads and checks the daily vision-API usage counter before proceeding |
| Download Image / Convert to Base64 | Fetches the photo and encodes it as a data URI (Cerebras requires embedded image bytes, not a remote URL) |
| Vision Analysis | Cerebras `gemma-4-31b` identifies distinct issues, severity, and trade category from the photo |
| Parse Vision Output | Cleans the model's response into structured fields, with a safe fallback if parsing fails |
| Reattach Image Binary | Re-attaches the original photo file to the data stream so it's available later for the vendor email |

**Vendor RFQ Dispatch**
| Node | What it does |
|---|---|
| Get Vendors | Reads the mock vendor list (name, email, specialty) from Google Sheets |
| Loop Vendors | Iterates each vendor one at a time |
| Attach Image To Item / Send RFQ | Sends a personalized, photo-attached RFQ email per vendor |
| Check Send Success / Log Failed Vendor | Detects a failed send and logs it separately, without halting the loop |

**Vendor Reply Intake (separate, always-on chain)**
| Node | What it does |
|---|---|
| Vendor Reply Trigger | Gmail Trigger polling for replies matching `subject:RFQ- has:attachment` |
| Extract PDF Text | Pulls raw text out of the attached PDF quote |
| Parse Quote PDF / Parse Quote JSON | LLM structures the raw text into vendor name, price, deposit %, line items |
| Log Received Quote | Appends the structured quote to the `ReceivedQuotes` sheet |

**Audit & Negotiation Decision**
| Node | What it does |
|---|---|
| Check Quote Count / Wait For Replies | Loops until at least 3 quotes are logged |
| Price Reference Search / Clean Search Snippets / Extract Price Range / Parse Price Range | Live web search for a real market-rate benchmark |
| Audit Quotes | Flags deposit risk, vague line items, and implausible pricing per vendor |
| Vendor Negotiation Agent | AI Agent ranks vendors by price + risk, drafts a counter-offer if warranted |
| Parse Agent Decision / Attach Vendor Email | Structures the agent's output and looks up the top vendor's real contact email |

**Human Approval & Outcome**
| Node | What it does |
|---|---|
| Send HITL Approval Request | Emails the homeowner the ranking, recommendation, and drafted counter-offer with Approve/Reject links |
| Wait | Pauses execution until the homeowner clicks a link |
| Check Approval Decision | Branches on the recorded decision |
| Send Vendor Counter-Offer / Log Confirmation | Sends the approved counter-offer and logs it |
| Log Rejection | Logs a rejected decision, no vendor contact made |

---

## 4. Setup & Credentials

All values below are placeholders — replace with your own real keys, never commit real secrets.

| Credential | Type | Where to get it |
|---|---|---|
| `Cerebras API` | Header Auth (`Authorization: Bearer <CEREBRAS_API_KEY>`) | cloud.cerebras.ai → API Keys |
| `Cerebras (OpenAI-compatible)` | OpenAI-style credential, Base URL `https://api.cerebras.ai/v1` | Same Cerebras key, used specifically for the AI Agent node |
| SerpAPI key | Query parameter `api_key` | serpapi.com → free tier |
| Google Sheets account | OAuth2 | Connect via n8n's Google login flow |
| Gmail account | OAuth2 | Connect via n8n's Google login flow (used for both sending and the Vendor Reply Trigger) |

Supporting spreadsheet tabs needed (single Google Sheet, multiple tabs): `VendorsMock`, `VisionQuota`, `ReceivedQuotes`, `FailedSends`, `Confirmations`.

---

## 5. How to Run It

1. Ensure the workflow is **Active** (published) in n8n.
2. POST a JSON payload to the `RFQ Trigger` webhook's Production URL, containing `scenario_type`, `image_url`, `area_sqft`, `location_area`, `user_email`, `user_name`.
3. Reply to the generated RFQ email thread from each mock vendor address with a PDF quote attached — the Vendor Reply Trigger picks these up automatically within about a minute.
4. Once at least 3 quotes are logged, the main chain proceeds automatically to audit, rank, and email the homeowner an approval request.
5. Click **Approve** or **Reject** in that email to resume and complete the workflow.

---

## 6. How to Verify It Worked

- **Vendor inboxes** receive a personalized RFQ email with the photo attached.
- **`ReceivedQuotes` sheet** fills with structured pricing data as vendors reply.
- **Homeowner inbox** receives a ranked, audited recommendation with a drafted counter-offer and Approve/Reject links.
- **Vendor inbox** (on approval) receives the counter-offer email, referencing the RFQ code.
- **`Confirmations` sheet** logs the final outcome — confirmed or rejected — with timestamp.


---

## 7. Known Limitations / Phase 2

- No zero-quotes guard: if a homeowner triggers the flow with zero vendor replies ever arriving, the wait loop runs indefinitely with no timeout or notification.
- No cross-case reference-code filtering: `ReceivedQuotes` isn't filtered by which specific case a quote belongs to, since the demo runs one case at a time.
- The vacating/deposit-recovery scenario (repair vs. forfeit security deposit) was scoped out — the webhook still accepts `scenario_type: "vacating"` and `estimated_deposit_aed`, but no distinct logic currently acts on them.
- Approve/Reject links are unauthenticated webhook URLs; a production version would use signed, expiring tokens.
