# UAE ICP Agentic Bootcamp — Step by Step Tutorial
### IBM watsonx Orchestrate · 100% UI · No Code · No Terminal

---

## What You Will Build

A UAE ICP Tourist Visa eligibility processing system using 3 agents:

- **ICP - Visa Eligibility Agent** — Master Agent that talks to the user
- **document_agent** — extracts data from passport and birth certificate
- **eligibility_agent** — runs all ICP eligibility checks

### How it flows

```
User
 │
 ▼
ICP - Visa Eligibility Agent (Master Agent)
 │
 ├── Phase 0: asks user 2 qualification questions
 │
 ├── Phase 1: calls document_agent → extracts passport + birth cert
 │
 ├── Phase 2: calls eligibility_agent → runs 6 checks
 │
 └── Phase 3: presents final visa decision to user
```

---

---

## PART 1 — Build Sub-Agent 1: Document Agent

### 1.1 Create the agent

Click the **hamburger menu** (☰) in the top left.
Select **Build**.
Click **Create Agent**.
Select **From scratch**.

Fill in:

| Field | Value |
|---|---|
| Name | document_agent |
| Description | Extracts and returns structured data from passport and birth certificate. Makes no decisions. |

Under **Style** select:
```
Default
```

Click **Create**.

---

### 1.2 Add the Behaviour

You are now on the agent page.
Click the **Behaviour** tab.
Paste the following:

```
You are a document extraction agent for UAE ICP visa processing.
When called, run the document extraction workflow.
The workflow will handle reading the passport and birth
certificate and returning the structured data.
Do not make any decisions or assessments about the documents.
Do not add any commentary or explanation to the output.
Return only what the workflow produces.
```

---

### 1.3 Create the Agentic Workflow

Scroll down to the **Tools** section on the agent page.
Click **Add tool**.
From the options select **Agentic Workflow**.
This opens the workflow canvas.

---

### 1.4 Build the Workflow

You will see a canvas with a **START** node and an **END**
node already placed.

> **How to add nodes:**
> Hover over the arrow between any two nodes and click the
> **+** button that appears on the arrow. A menu pops up —
> select the node type you need. The new node is inserted
> automatically between the existing nodes.

---

#### Node 1 — Document Extractor (Passport)

Click **+** on the arrow between START and END.
From the menu select:
```
Add a flow activity → Document extractor
```

> **Rename the node:**
> Click the **pencil icon** in the top left corner of the
> node. Type:
> ```
> Extract Passport Fields
> ```

> **Change the model:**
> In the top right corner of the node configuration panel
> click the model selector dropdown and select:
> ```
> gpt-oss-120b
> ```

In the field configuration section click **Add field**
and add these fields one by one:

| Field name | Type | Description |
|---|---|---|
| `passport_num` | string | Passport number as printed on the document |
| `nationality` | string | 3-letter ISO code only. POL=Polish, ARE=Emirati, IND=Indian, GBR=British, USA=American, PAK=Pakistani, PHL=Filipino, EGY=Egyptian, SYR=Syrian, AFG=Afghan, IRQ=Iraqi, SOM=Somali, YEM=Yemeni |
| `given_name` | string | Given name as printed |
| `surname` | string | Surname as printed |
| `DOB` | string | Date of birth in format YYYY-MM-DD |
| `EXP_DATE` | string | Expiry date in format YYYY-MM-DD |

When prompted to choose a document type select:
```
Unstructured
```

Then upload the test passport file provided:
```
passport_juan_tapia.jpg
```

---

#### Node 2 — Document Extractor (Birth Certificate)

Click **+** on the arrow between Node 1 and END.
From the menu select:
```
Add a flow activity → Document extractor
```

> **Rename the node:**
> Click the **pencil icon** in the top left corner. Type:
> ```
> Extract Birth Cert Fields
> ```

> **Change the model:**
> In the top right corner click the model selector and select:
> ```
> gpt-oss-120b
> ```

Click **Add field** and add:

| Field name | Type | Description |
|---|---|---|
| `full_name_birth_certificate` | string | Full name exactly as written in the document combining surname and given name |
| `DOB_birth_certificate` | string | Date of birth in format YYYY-MM-DD |

When prompted to choose a document type select:
```
Unstructured
```

Then upload the test birth certificate file provided:
```
birth_certificate_juan_tapia.pdf
```

---

#### Node 3 — Generative Prompt (Package Output)

Click **+** on the arrow between Node 2 and END.
From the menu select:
```
Add a flow activity → Generative prompt
```

> **Rename the node:**
> Click the **pencil icon** in the top left corner. Type:
> ```
> Package Output
> ```

**Add input variables:**
Click the **Input variables** tab inside the node.
Click **Add variable** and add these one by one:

| Variable name | Type |
|---|---|
| `passport_num` | string |
| `nationality` | string |
| `given_name` | string |
| `surname` | string |
| `DOB` | string |
| `EXP_DATE` | string |
| `DOB_birth_certificate` | string |
| `full_name_birth_certificate` | string |

**System Prompt:**
```
You are a data packaging assistant for UAE ICP visa processing.
Your only job is to combine extracted document data into a
clean JSON object.
You must return only valid JSON. No explanation, no commentary,
no extra text.
Never modify, correct, or interpret any field values.
Always preserve the exact values as given to you.
```

**User Prompt:**
```
Combine the following two documents into a single JSON object
with exactly two keys: "passport" and "birth_certificate".

Passport data:
- Passport Number: {self.input.passport_num}
- Nationality: {self.input.nationality}
- Given Name: {self.input.given_name}
- Surname: {self.input.surname}
- Date of Birth: {self.input.DOB}
- Expiry Date: {self.input.EXP_DATE}

Birth Certificate data:
- Date of Birth: {self.input.DOB_birth_certificate}
- Full Name: {self.input.full_name_birth_certificate}

Return only this structure and nothing else:
{
  "passport": {
    "passport_num": {self.input.passport_num},
    "nationality": {self.input.nationality},
    "given_name": {self.input.given_name},
    "surname": {self.input.surname},
    "DOB": {self.input.DOB},
    "EXP_DATE": {self.input.EXP_DATE}
  },
  "birth_certificate": {
    "DOB": {self.input.DOB_birth_certificate},
    "full_name": {self.input.full_name_birth_certificate}
  }
}
```

**Data Mapping:**
Click the **Input mapping** tab inside the node.
Click **Add mapping** and map each variable:

| Input variable | Expression |
|---|---|
| `passport_num` | `flow["Extract Passport Fields"].output.passport_num` |
| `nationality` | `flow["Extract Passport Fields"].output.nationality` |
| `given_name` | `flow["Extract Passport Fields"].output.given_name` |
| `surname` | `flow["Extract Passport Fields"].output.surname` |
| `DOB` | `flow["Extract Passport Fields"].output.DOB` |
| `EXP_DATE` | `flow["Extract Passport Fields"].output.EXP_DATE` |
| `DOB_birth_certificate` | `flow["Extract Birth Cert Fields"].output.DOB_birth_certificate` |
| `full_name_birth_certificate` | `flow["Extract Birth Cert Fields"].output.full_name_birth_certificate` |

> The node name in the expression must match exactly what
> you typed using the pencil icon. If you named it
> differently update the expression accordingly.

---

#### Final canvas for document_agent

```
START
  │
  ▼
Extract Passport Fields  (Document Extractor)
  │
  ▼
Extract Birth Cert Fields  (Document Extractor)
  │
  ▼
Package Output  (Generative Prompt)
  │
  ▼
END
```

---

### 1.5 Save and Preview

Click **Save** in the top right of the workflow canvas.
Go back to the agent page and click **Preview** in the
top right to open the Agent Builder preview.
In the chat panel type:

```
Extract the documents
```

Expected output:

```json
{
  "passport": {
    "passport_num": "15082701",
    "nationality": "POL",
    "given_name": "JUAN",
    "surname": "TAPIA",
    "DOB": "1988-08-08",
    "EXP_DATE": "2030-02-24"
  },
  "birth_certificate": {
    "DOB": "1988-08-08",
    "full_name": "JUAN TAPIA"
  }
}
```

---

## PART 2 — Build Sub-Agent 2: Eligibility Agent

### 2.1 Create the agent

Click the **hamburger menu** (☰) in the top left.
Select **Build**.
Click **Create Agent**.
Select **From scratch**.

Fill in:

| Field | Value |
|---|---|
| Name | eligibility_agent |
| Description | Receives extracted document data and user declaration. Runs all UAE ICP eligibility checks and returns ELIGIBLE, PENDING, or REJECTED with a reason. |

Under **Style** select:
```
Default
```

Click **Create**.

---

### 2.2 Add the Behaviour

You are now on the agent page.
Click the **Behaviour** tab.
Paste the following:

```
You are a UAE ICP visa eligibility checking agent.
When called, run your eligibility workflow.
The workflow will handle all checks and return the decision.
Do not modify the result.
Do not add any commentary.
Return only what the workflow produces.
```

---

### 2.3 Create the Agentic Workflow

Scroll down to the **Tools** section on the agent page.
Click **Add tool**.
From the options select **Agentic Workflow**.
This opens the workflow canvas.

---

### 2.4 Define inputs at the START node

Click the **START** node on the canvas.
In the configuration panel on the right click **Add input**
and add these variables one by one:

| Variable | Type |
|---|---|
| `passport_num` | string |
| `nationality` | string |
| `given_name` | string |
| `surname` | string |
| `DOB` | date |
| `EXP_DATE` | date |
| `DOB_birth_certificate` | date |
| `full_name_birth_certificate` | string |
| `has_flight_ticket` | boolean |
| `accommodation_address` | string |

---

### 2.5 Build the Workflow

---

#### Node 1 — Logic Block (all eligibility checks)

Click **+** on the arrow between START and END.
From the menu select:
```
Add a flow activity → Logic block
```

> **Rename the node:**
> Click the **pencil icon** in the top left corner. Type:
> ```
> ICP Eligibility Checks
> ```

**Declare output variables:**
Click the **Outputs** tab inside the node.
Click **Add output** and add:

| Variable | Type |
|---|---|
| `status` | string |
| `reason` | string |

**Code to paste:**

```python
# ── RULE 1: Passport validity ──
today = datetime.date.today()
expiry = datetime.datetime.strptime(flow.input.EXP_DATE, "%Y-%m-%d").date()
days_remaining = (expiry - today).days

if days_remaining < 180:
    self.output.status = "REJECTED"
    self.output.reason = "Passport expires in " + str(days_remaining) + " days. Minimum 180 days required."

# ── RULE 2: Nationality restriction ──
elif flow.input.nationality.upper() in ["AFG", "IRQ", "SYR", "SOM", "YEM"]:
    self.output.status = "REJECTED"
    self.output.reason = "Your application cannot be processed. Nationals of " + flow.input.nationality + " are currently subject to UAE ICP entry restrictions and are not eligible for a Tourist Visa at this time. Please contact your nearest UAE embassy for further guidance."

# ── RULE 3: DOB cross-check ──
elif flow.input.DOB != flow.input.DOB_birth_certificate:
    self.output.status = "REJECTED"
    self.output.reason = "DOB mismatch — passport: " + flow.input.DOB + ", birth certificate: " + flow.input.DOB_birth_certificate

# ── RULE 4: Name cross-check ──
elif flow.input.surname.upper() not in flow.input.full_name_birth_certificate.upper() and (flow.input.given_name + " " + flow.input.surname).upper() not in flow.input.full_name_birth_certificate.upper():
    self.output.status = "REJECTED"
    self.output.reason = "Name mismatch — passport: " + (flow.input.given_name + " " + flow.input.surname).upper() + ", birth certificate: " + flow.input.full_name_birth_certificate.upper()

# ── RULE 5: Flight ticket check ──
elif flow.input.has_flight_ticket != True:
    self.output.status = "REJECTED"
    self.output.reason = "A confirmed flight ticket is required for UAE Tourist Visa processing."

# ── RULE 6: Accommodation check ──
elif not flow.input.accommodation_address or len(flow.input.accommodation_address.strip()) < 10:
    self.output.status = "REJECTED"
    self.output.reason = "A valid accommodation address in the UAE is required."

# ── ALL CHECKS PASSED ──
else:
    self.output.status = "ELIGIBLE"
    self.output.reason = "All UAE ICP Tourist Visa requirements met. Applicant: " + flow.input.given_name + " " + flow.input.surname + " | Passport: " + flow.input.passport_num + " | Nationality: " + flow.input.nationality
```

**Data Mapping for Logic Block:**
Click the **Input mapping** tab inside the node.
Click **Add mapping** and map each variable:

| Input variable | Expression |
|---|---|
| `passport_num` | `flow.input.passport_num` |
| `nationality` | `flow.input.nationality` |
| `given_name` | `flow.input.given_name` |
| `surname` | `flow.input.surname` |
| `DOB` | `flow.input.DOB` |
| `EXP_DATE` | `flow.input.EXP_DATE` |
| `DOB_birth_certificate` | `flow.input.DOB_birth_certificate` |
| `full_name_birth_certificate` | `flow.input.full_name_birth_certificate` |
| `has_flight_ticket` | `flow.input.has_flight_ticket` |
| `accommodation_address` | `flow.input.accommodation_address` |

---

#### Node 2 — Generative Prompt (Format result)

Click **+** on the arrow between Node 1 and END.
From the menu select:
```
Add a flow activity → Generative prompt
```

> **Rename the node:**
> Click the **pencil icon** in the top left corner. Type:
> ```
> Format Result
> ```

**Add input variables:**
Click the **Input variables** tab inside the node.
Click **Add variable** and add:

| Variable name | Type |
|---|---|
| `status` | string |
| `reason` | string |

**System Prompt:**
```
You are a UAE ICP visa processing assistant.
Present the eligibility result to the user in a clear
and professional way.
Do not add any information that was not in the result.
```

**User Prompt:**
```
The eligibility check is complete. Here is the result:

Status: {self.input.status}
Reason: {self.input.reason}

If ELIGIBLE — congratulate the applicant and confirm
their Tourist Visa application can proceed.
If PENDING — explain that manual review is required
and an ICP officer will follow up.
If REJECTED — explain the reason clearly and what the
applicant should address.
```

**Data Mapping for Format Result:**
Click the **Input mapping** tab inside the node.
Click **Add mapping** and map:

| Input variable | Expression |
|---|---|
| `status` | `flow["ICP Eligibility Checks"].output.status` |
| `reason` | `flow["ICP Eligibility Checks"].output.reason` |

---

#### Final canvas for eligibility_agent

```
START
  │
  ▼
ICP Eligibility Checks  (Logic Block)
  │
  ▼
Format Result  (Generative Prompt)
  │
  ▼
END
```

---

### 2.6 Save and Preview

Click **Save** in the top right of the workflow canvas.
Go back to the agent page and click **Preview** in the
top right to open the Agent Builder preview.
In the chat panel enter:

```
passport_num: 15082701
nationality: POL
given_name: JUAN
surname: TAPIA
DOB: 1988-08-08
EXP_DATE: 2030-02-24
DOB_birth_certificate: 1988-08-08
full_name_birth_certificate: JUAN TAPIA
has_flight_ticket: true
accommodation_address: Hilton Hotel, Sheikh Zayed Road, Dubai, UAE
```

Expected result: **ELIGIBLE**

---

## PART 3 — Build the Master Agent

### 3.1 Create the agent

Click the **hamburger menu** (☰) in the top left.
Select **Build**.
Click **Create Agent**.
Select **From scratch**.

Fill in:

| Field | Value |
|---|---|
| Name | ICP - Visa Eligibility Agent |
| Description | UAE ICP Tourist Visa eligibility orchestrator. Qualifies the user upfront, extracts documents, runs eligibility checks and delivers the final visa decision. |

Under **Style** select:
```
React
```

Click **Create**.

---

### 3.2 Add the Welcome Message

You are now on the agent page.
Find the **Welcome message** field and paste:

```
Welcome to the UAE Tourist Visa Eligibility Agent!
```

---

### 3.3 Quick Start Prompts

Scroll to the **Quick start prompts** section.
Delete all existing questions by clicking the **X** on each.
Click **+** on the right and add:

```
Check for Tourist Visa Eligibility
```

---

### 3.4 Add the Behaviour

Click the **Behaviour** tab.
Paste the following:

```
You are the UAE ICP Tourist Visa processing assistant.
You are the only agent that communicates with the user.
You manage the full visa application flow step by step.

PHASE 0 — Upfront qualification:
Start every conversation by greeting the user and asking
these two questions before doing anything else:
  1. "Do you have a confirmed flight ticket to the UAE?"
  2. "What is your planned accommodation address in the UAE?
      For example: a hotel name and address, or the address
      of where you will be staying."

Wait for both answers before proceeding.
If the user answers one question, only ask for the missing
answer. Never repeat a question already answered.
If the user answers both in one message, capture both
and proceed without asking again.
If the user's answer is unclear, ask only for clarification
on that specific point.

If the user answers NO to the flight ticket question:
  - Do not proceed further.
  - Respond with:
    "Unfortunately we cannot process your UAE Tourist Visa
     application at this time. A confirmed flight ticket is
     a mandatory requirement. Please book your flight and
     return when you have a confirmed ticket."
  - End the conversation.

If the user answers YES to the flight ticket question
AND provides an accommodation address:
  - Interpret the accommodation answer and structure it cleanly.
  - Always append UAE at the end if not already mentioned.
  - Examples:
      "relatives in Fujairah" → "Fujairah, UAE"
      "Hilton on sheikh zayed road dubai" → "Hilton, Sheikh Zayed Road, Dubai, UAE"
      "my friend's place in abu dhabi" → "Abu Dhabi, UAE"
  - Present the structured summary to the user in this format:
    "Here is what I have recorded:
    - Flight Ticket: Yes
    - Accommodation Address: [structured address]

    Kindly confirm to proceed."
  - Wait for the user to confirm.
  - If the user wants to make changes, update the relevant
    detail and show the summary again asking to confirm.
  - Only proceed to Phase 1 AFTER the user confirms.

PHASE 1 — Document extraction:
  - Inform the user:
    "Thank you for confirming. I will now extract your
     documents for processing."
  - Call document_agent immediately.
  - Wait for document_agent to return the extracted data.
  - If document_agent returns INCOMPLETE, inform the user and stop.

PHASE 2 — Eligibility check:
  - Once document_agent returns successfully, immediately
    call eligibility_agent silently.
  - Do not list or narrate the data you are passing to
    eligibility_agent.
  - Do not show any intermediate message between document
    extraction and the eligibility result.
  - Pass the following without mentioning them to the user:
      - All extracted data returned by document_agent
      - has_flight_ticket: true
      - accommodation_address: the confirmed structured address from Phase 0
  - Only speak to the user again when eligibility_agent
    returns the final result.

PHASE 3 — Deliver result:
  Once you receive the result from eligibility_agent,
  present the final result to the user in this exact
  structure with bold labels:

  Applicant: [given_name] [surname]
  Passport Number: [passport_num]
  Nationality: [nationality]
  Date of Birth: [DOB]

  Application Status: [ELIGIBLE / PENDING / REJECTED]

  Details:
  [reason from eligibility_agent]

  If ELIGIBLE:
    Add a congratulations message and wish them a pleasant trip to the UAE.

  If PENDING:
    Add a message that an ICP officer will be in touch with next steps.

  If REJECTED:
    Add a message clearly advising what they need to address before reapplying.

  Only include these 4 applicant fields in the summary.
  Do not include accommodation address, flight ticket,
  birth certificate details, or any other extracted fields.
  Never present the status and details as a single concatenated line.
  Always break status and details into separate lines.

Rules you must always follow:
- Never skip Phase 0.
- Never call document_agent before the user confirms in Phase 0.
- Never call document_agent if has_flight_ticket is NO.
- Never call eligibility_agent before document_agent has succeeded.
- Never narrate or list data being passed between agents.
- Never make up or assume any information not provided by the user
  or returned by an agent.
- Always be polite and professional throughout.
```

---

### 3.5 Add Sub-Agents

Scroll down to the **Agent** section.
Click **Add agents**.
When prompted choose **Local instance**.
From the list select:

```
+ document_agent
+ eligibility_agent
```

Click **Add** to confirm.

---

### 3.6 Save

Click **Save** in the top right.

---

## Full Pipeline Test

Click **Preview** on the Master Agent to open the chat.
Click the quick start prompt:
```
Check for Tourist Visa Eligibility
```

Run through the full conversation:

```
Agent : Welcome to the UAE Tourist Visa Eligibility Agent!
        1. Do you have a confirmed flight ticket to the UAE?
        2. What is your planned accommodation address in the UAE?

User  : yes i have a ticket, staying at Hilton Dubai

Agent : Here is what I have recorded:
        - Flight Ticket: Yes
        - Accommodation Address: Hilton, Dubai, UAE
        Kindly confirm to proceed.

User  : confirm

Agent : Thank you for confirming. I will now extract
        your documents for processing.

        [document_agent and eligibility_agent run silently]

Agent : Applicant: Juan Tapia
        Passport Number: 15082701
        Nationality: POL
        Date of Birth: 1988-08-08

        Application Status: ELIGIBLE

        Details:
        All UAE ICP Tourist Visa requirements met.

        Congratulations! Your Tourist Visa application can
        proceed. We wish you a wonderful trip to the UAE!
```

---

## Test Scenarios

Run all 5 to validate every path in the system.

| # | Scenario | How to simulate | Expected |
|---|---|---|---|
| 1 | Happy path | Yes to ticket · valid address · confirm | ELIGIBLE |
| 2 | No flight ticket | Answer no to ticket question | Stops — mandatory requirement message |
| 3 | Expired passport | Change EXP_DATE in passport extractor to 2025-01-01 | REJECTED — expires in X days |
| 4 | DOB mismatch | Change DOB_birth_certificate in birth cert extractor to 1991-06-20 | REJECTED — DOB mismatch |
| 5 | Restricted nationality | Change nationality in passport extractor to SYR | PENDING — manual review required |

---

## Key Concepts

| Concept | Where it appears |
|---|---|
| Hamburger menu → Build → Create Agent → From scratch | How to navigate to agent creation |
| Style: Default | Both sub-agents — keeps agent focused on tool execution |
| Style: React | Master Agent — enables conversational multi-turn interaction |
| Add tool → Agentic Workflow | How the workflow canvas is opened inside an agent |
| Pencil icon | Renames any node in the canvas |
| Model selector top right | Set to gpt-oss-120b on each Document Extractor node |
| Input variables | Declared inside each Generative Prompt node before mapping |
| Data mapping | Wires outputs of one node into inputs of the next |
| Logic Block | All 6 ICP eligibility rules in one Python block |
| Add agents → Local instance | How sub-agents are wired to the master agent |
| Gate logic | Master Agent stops the flow if ticket answer is NO |
| Separation of concerns | Agent 1 extracts · Agent 2 decides · Master orchestrates |
