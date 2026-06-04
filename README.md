# UAE ICP Agentic Bootcamp — Step by Step Tutorial
### IBM watsonx Orchestrate · UI + Python Tool

---

## What You Will Build

A UAE ICP Tourist Visa eligibility processing system using 3 agents:

- **ICP - Visa Eligibility Agent** — Master Agent that talks to the user
- **document_agent** — extracts data from passport and birth certificate
- **eligibility_agent** — runs all ICP eligibility checks via a Python tool

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
 ├── Phase 2: calls eligibility_agent → runs 6 checks via Python tool
 │
 └── Phase 3: presents final visa decision to user
```

---

## Prerequisites

Before starting, make sure you have:

- **watsonx Orchestrate SaaS environment** provisioned and accessible,
  with your environment URL and API key ready —
  no account yet? [Provision a free trial here](https://www.ibm.com/account/reg/us-en/signup?formid=urx-52753&cm_sp=ibmdev-_-developer-_-trial&utm_source=ibm_developer&utm_content=in_content_link&utm_id=tutorials_develop-agents-no-code-watsonx-orchestrate)

- **Python 3.11** installed on your machine

### How to get your API key and Service instance URL

1. Log in to your watsonx Orchestrate environment
2. Click your **Profile icon** in the top-right corner
3. Click **Settings**
4. Click the **API details** tab
5. Click **Generate API key** — copy and save it somewhere safe
6. Copy the **Service instance URL** shown below the button

```
Service instance URL:
https://api.dl.watson-orchestrate.ibm.com/instances/<your-instance-id>
```

> You will need both the **API key** and the **Service instance URL**
> when setting up the eligibility tool in Part 2.

---

## Documents

Three passports and three birth certificates are provided.
Each has a specific role in the bootcamp:

| Role | Person | Passport | Birth Certificate |
|---|---|---|---|
| **Training** — used while building the agents | Juan Tapia | `JT_polpp.jpg` | `birth_certificate_juan_tapia.pdf` |
| **Test: ELIGIBLE** — used to test the happy path | Maksym Staniszewski | `maksym_passport.png` | `birth_certificate_maksym.pdf` |
| **Test: REJECTED** — used to test nationality rejection | Celeste Nguemo | `cameroon_passport.jpg` | `birth_certificate_celeste_nguemo.pdf` |

> During **Part 1** you will upload the **training documents** (Juan Tapia)
> into the Document Extractor nodes. This is the document the agent
> learns to extract from while you are building.
>
> During **Test Scenarios** at the end, you will swap the documents
> to run the ELIGIBLE and REJECTED test cases.

---

## PART 2 — Build Sub-Agent 2: Eligibility Agent

The eligibility agent uses a Python tool imported via the ADK CLI.
Complete the setup steps below before building the agent in the UI.

### 2.0 Setup — Import the Eligibility Tool

You will need the **API key** and **Service instance URL** from
the Prerequisites section above.

Open a terminal and run:

```bash
pip install ibm-watsonx-orchestrate
orchestrate env start
orchestrate env activate
```

When prompted, enter your:
- Service instance URL
- API key

---

### 2.0.1 The tool file — eligibility_check_tool.py

Create a file called `eligibility_check_tool.py` with this code:

```python
from ibm_watsonx_orchestrate.agent_builder.tools import tool
from pydantic import BaseModel, Field
from datetime import date, datetime


class VisaEligibilityResult(BaseModel):
    status: str = Field(description="ELIGIBLE or REJECTED")
    reason: str = Field(description="Explanation of the eligibility result")


@tool
def check_visa_eligibility(
    passport_num: str,
    nationality_code: str,
    Given_name: str,
    surname: str,
    Date_of_Birth: date,
    Date_of_Expiry: date,
    Birth_Certificate_DOB: date,
    Birth_Certificate_Full_Name: str,
    has_flight_ticket: bool,
    accommodation_address: str
) -> VisaEligibilityResult:
    """
    Checks UAE ICP Tourist Visa eligibility based on extracted
    document data and user declaration.

    Args:
        passport_num (str): Passport number
        nationality_code (str): 3-letter ISO nationality code from CODE field
        Given_name (str): Given name from passport
        surname (str): Surname from passport
        Date_of_Birth (date): Date of birth from passport
        Date_of_Expiry (date): Passport expiry date
        Birth_Certificate_DOB (date): Date of birth from birth certificate
        Birth_Certificate_Full_Name (str): Full name from birth certificate
        has_flight_ticket (bool): Whether applicant has a confirmed flight ticket
        accommodation_address (str): Planned accommodation address in UAE

    Returns:
        VisaEligibilityResult: status (ELIGIBLE or REJECTED) and reason
    """

    # ── Convert dates safely — handle date, datetime or string ──
    def to_date(val):
        if isinstance(val, datetime):
            return val.date()
        elif isinstance(val, str):
            return datetime.strptime(val.strip().split("T")[0], "%Y-%m-%d").date()
        else:
            return val

    exp = to_date(Date_of_Expiry)
    dob_passport = to_date(Date_of_Birth)
    dob_cert = to_date(Birth_Certificate_DOB)

    failures = []

    # ── RULE 1: Nationality restriction — checked first, exits immediately ──
    restricted = ["AFG", "LBY", "YEM", "SOM", "SDN", "CMR"]
    if nationality_code.upper() in restricted:
        return VisaEligibilityResult(
            status="REJECTED",
            reason="Your application cannot be processed. Nationals of " + nationality_code + " are currently subject to UAE ICP entry restrictions and are not eligible for a Tourist Visa at this time. Please contact your nearest UAE embassy for further guidance."
        )

    # ── RULE 2: Passport validity ──
    today = date.today()
    days_remaining = (exp - today).days
    if days_remaining < 180:
        failures.append("Passport expires in " + str(days_remaining) + " days. Minimum 180 days required.")

    # ── RULE 3: DOB cross-check ──
    if dob_passport != dob_cert:
        failures.append("DOB mismatch — passport: " + str(dob_passport) + ", birth certificate: " + str(dob_cert))

    # ── RULE 4: Name cross-check ──
    if surname.upper() not in Birth_Certificate_Full_Name.upper() or Given_name.upper() not in Birth_Certificate_Full_Name.upper():
        failures.append("Name mismatch — passport: " + (Given_name + " " + surname).upper() + ", birth certificate: " + Birth_Certificate_Full_Name.upper())

    # ── RULE 5: Flight ticket check ──
    if has_flight_ticket != True:
        failures.append("A confirmed flight ticket is required for UAE Tourist Visa processing.")

    # ── RULE 6: Accommodation check ──
    if not accommodation_address or len(accommodation_address.strip()) < 10:
        failures.append("A valid accommodation address in the UAE is required.")

    # ── RETURN RESULT ──
    if failures:
        return VisaEligibilityResult(
            status="REJECTED",
            reason=" | ".join(failures)
        )

    return VisaEligibilityResult(
        status="ELIGIBLE",
        reason="All UAE ICP Tourist Visa requirements met. Applicant: " + Given_name + " " + surname + " | Passport: " + passport_num + " | Nationality: " + nationality_code
    )
```

---

### 2.0.2 requirement.txt

Create a file called `requirement.txt`:

```txt
ibm-watsonx-orchestrate
pydantic
```

---

### 2.0.3 Import the tool

```bash
orchestrate tools import --kind python -r requirement.txt -f eligibility_check_tool.py
```

Confirm the tool appears in:
```
Home → Tools → check_visa_eligibility
```

---

## PART 1 — Build Sub-Agent 1: Document Agent

> **Accessing your environment:**
> Open the watsonx Orchestrate instance URL shared with you
> in the email after account creation. Log in and you will
> land on the home page — this is where you will build
> all three agents.

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

When prompted to choose a document type select:
```
Unstructured
```

Then upload the **training** passport file:
```
JT_polpp.jpg
```

> This is the training document used while building.
> You will swap this file during the test scenarios.

In the field configuration section click **Add field**
and add these fields one by one:

| Field name | Type | Description |
|---|---|---|
| `passport_num` | string | Passport number as printed on the document |
| `nationality` | string | Nationality as written on the passport e.g. POLISH, CAMEROONIAN |
| `nationality_code` | string | 3-letter ISO code from the CODE/KOD field at the top of the passport. This is NOT the nationality word — it is the machine-readable code printed next to TYPE. CMR=Cameroonian, POL=Polish, ARE=Emirati, IND=Indian, GBR=British, USA=American, PAK=Pakistani, PHL=Filipino, EGY=Egyptian, AFG=Afghan, LBY=Libyan, SDN=Sudanese, YEM=Yemeni, SOM=Somali |
| `Given_name` | string | Given name as printed |
| `surname` | string | Surname as printed |
| `Date_of_Birth` | date | Date of birth in format YYYY-MM-DD |
| `Date_of_Expiry` | date | Expiry date in format YYYY-MM-DD |

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

When prompted to choose a document type select:
```
Unstructured
```

Then upload the **training** birth certificate file:
```
birth_certificate_juan_tapia.pdf
```

> This is the training document used while building.
> You will swap this file during the test scenarios.

Click **Add field** and add:

| Field name | Type | Description |
|---|---|---|
| `Birth_Certificate_Full_Name` | string | Full name exactly as written in the Full Name field of the document. Do not duplicate any part of the name. |
| `Birth_Certificate_DOB` | date | Date of birth in format YYYY-MM-DD |

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
| `nationality_code` | string |
| `Given_name` | string |
| `surname` | string |
| `Date_of_Birth` | string |
| `Date_of_Expiry` | string |
| `Birth_Certificate_DOB` | string |
| `Birth_Certificate_Full_Name` | string |

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
- Nationality Code: {self.input.nationality_code}
- Given Name: {self.input.Given_name}
- Surname: {self.input.surname}
- Date of Birth: {self.input.Date_of_Birth}
- Expiry Date: {self.input.Date_of_Expiry}

Birth Certificate data:
- Date of Birth: {self.input.Birth_Certificate_DOB}
- Full Name: {self.input.Birth_Certificate_Full_Name}

Return only this structure and nothing else:
{
  "passport": {
    "passport_num": {self.input.passport_num},
    "nationality": {self.input.nationality},
    "nationality_code": {self.input.nationality_code},
    "Given_name": {self.input.Given_name},
    "surname": {self.input.surname},
    "Date_of_Birth": {self.input.Date_of_Birth},
    "Date_of_Expiry": {self.input.Date_of_Expiry}
  },
  "birth_certificate": {
    "Birth_Certificate_DOB": {self.input.Birth_Certificate_DOB},
    "Birth_Certificate_Full_Name": {self.input.Birth_Certificate_Full_Name}
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
| `nationality_code` | `flow["Extract Passport Fields"].output.nationality_code` |
| `Given_name` | `flow["Extract Passport Fields"].output.Given_name` |
| `surname` | `flow["Extract Passport Fields"].output.surname` |
| `Date_of_Birth` | `flow["Extract Passport Fields"].output.Date_of_Birth` |
| `Date_of_Expiry` | `flow["Extract Passport Fields"].output.Date_of_Expiry` |
| `Birth_Certificate_DOB` | `flow["Extract Birth Cert Fields"].output.Birth_Certificate_DOB` |
| `Birth_Certificate_Full_Name` | `flow["Extract Birth Cert Fields"].output.Birth_Certificate_Full_Name` |

> The node name in the expression must match exactly what
> you typed using the pencil icon.

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

#### END Node — Output Variables

Click the **END** node on the canvas.
Click **Add** and add these 9 output variables:

| Variable name | Type |
|---|---|
| `Birth_Certificate_DOB` | date |
| `Birth_Certificate_Full_Name` | string |
| `Date_of_Birth` | date |
| `Date_of_Expiry` | date |
| `Given_name` | string |
| `nationality` | string |
| `nationality_code` | string |
| `passport_num` | string |
| `surname` | string |

Then click **Edit data mapping** at the bottom of the END node
and map each variable:

| Output variable | Expression |
|---|---|
| `Birth_Certificate_DOB` | `flow["Extract Birth Cert Fields"].output.Birth_Certificate_DOB` |
| `Birth_Certificate_Full_Name` | `flow["Extract Birth Cert Fields"].output.Birth_Certificate_Full_Name` |
| `Date_of_Birth` | `flow["Extract Passport Fields"].output.Date_of_Birth` |
| `Date_of_Expiry` | `flow["Extract Passport Fields"].output.Date_of_Expiry` |
| `Given_name` | `flow["Extract Passport Fields"].output.Given_name` |
| `nationality` | `flow["Extract Passport Fields"].output.nationality` |
| `nationality_code` | `flow["Extract Passport Fields"].output.nationality_code` |
| `passport_num` | `flow["Extract Passport Fields"].output.passport_num` |
| `surname` | `flow["Extract Passport Fields"].output.surname` |

---

### 1.5 Save and Preview

Click **Save** in the top right of the workflow canvas.
Go back to the agent page and click **Preview**.
In the chat panel type:

```
Extract the documents
```

Expected output:

```json
{
  "passport": {
    "passport_num": "15082701",
    "nationality": "POLISH",
    "nationality_code": "POL",
    "Given_name": "JUAN",
    "surname": "TAPIA",
    "Date_of_Birth": "1988-08-08",
    "Date_of_Expiry": "2030-02-24"
  },
  "birth_certificate": {
    "Birth_Certificate_DOB": "1988-08-08",
    "Birth_Certificate_Full_Name": "JUAN TAPIA"
  }
}
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
| Description | Receives extracted document data and user declaration. Runs all UAE ICP eligibility checks via the check_visa_eligibility tool and returns ELIGIBLE or REJECTED with a reason. |

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
When called, use the check_visa_eligibility tool with all
the inputs provided to you.
Do not modify any input values before passing them to the tool.
Do not add any commentary to the result.
Return only what the tool produces.
```

---

### 2.3 Add the Tool

Scroll down to the **Tools** section on the agent page.
Click **Add tool**.
From the options select **From your tools**.
Find and select:

```
check_visa_eligibility
```

Click **Add** to confirm.

---

### 2.4 Save and Preview

Click **Save** in the top right.
Click **Preview** to open the chat panel.
Enter this test input:

```
passport_num: 15082701
nationality_code: POL
Given_name: JUAN
surname: TAPIA
Date_of_Birth: 1988-08-08
Date_of_Expiry: 2030-02-24
Birth_Certificate_DOB: 1988-08-08
Birth_Certificate_Full_Name: JUAN TAPIA
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
      Please include the Emirate you will be staying in.
      For example: Hilton Hotel, Dubai or relatives in Abu Dhabi"

Wait for both answers before proceeding.
If the user answers both questions in one message or
answers them out of order, recognise and capture both
answers immediately. Do not repeat questions that have
already been answered.
If the user answers only one question, only ask for
the missing answer. Never repeat a question the user
has already answered.
If the user's answer is general, unclear or incomplete, ask
only for clarification on that specific point.

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

  - Identify which of the 7 UAE Emirates is mentioned.
    Valid Emirates (accept common misspellings):
      Dubai, Abu Dhabi, Sharjah, Ajman,
      Umm Al Quwain, Ras Al Khaimah, Fujairah
    Common misspellings to accept:
      "Abudhabi" → Abu Dhabi
      "Sharja" → Sharjah
      "Fujeira" → Fujairah
      "Ras al khaima" → Ras Al Khaimah
      "Umm al quain" → Umm Al Quwain

  - If no valid Emirate is mentioned or recognisable,
    ask the user:
    "Which Emirate in the UAE will you be staying in?
     (Dubai / Abu Dhabi / Sharjah / Ajman /
      Umm Al Quwain / Ras Al Khaimah / Fujairah)"
    Wait for the answer before proceeding.

  - If the user provides a location that is NOT one of
    the 7 Emirates and cannot be matched to one, respond:
    "We are unable to process your application. The
     accommodation address must be within one of the
     7 UAE Emirates. Please provide a valid UAE address."
    End the conversation.

  - Once a valid Emirate is confirmed, structure the
    address cleanly in this format:
    [Property/Area], [Emirate], UAE

  - Always append UAE at the end if not already mentioned.

  - Present the structured summary to the user in this format:
    "Here is what I have recorded:
    - Flight Ticket: Yes
    - Accommodation Address: [structured address]

    Kindly type "Confirm" to proceed"

  - Wait for the user to type "confirm". Do not proceed until
    the user types it explicitly.
  - If the user wants to make changes, update the relevant
    detail and show the summary again asking to type "confirm".
  - Only proceed to Phase 1 AFTER the user types "confirm".

PHASE 1 — Document extraction:
  - Inform the user:
    "Thank you for confirming. I will now extract your
     documents for processing."
  - Call document_agent immediately.
  - Wait for document_agent to return the extracted data.
  - If document_agent returns INCOMPLETE, inform the user and stop.

PHASE 2 — Eligibility check:
  - Immediately after document_agent returns successfully,
    you MUST call eligibility_agent. This step is mandatory
    and must never be skipped.
  - Do not wait for the user. Do not ask any questions.
  - Call eligibility_agent with these exact field names
    and values — pass every value verbatim as received
    from document_agent, do not modify any value:
      passport_num: [exact value from document_agent]
      nationality_code: [exact value from document_agent]
      Given_name: [exact value from document_agent]
      surname: [exact value from document_agent]
      Date_of_Birth: [exact value from document_agent]
      Date_of_Expiry: [exact value from document_agent]
      Birth_Certificate_DOB: [exact value from document_agent]
      Birth_Certificate_Full_Name: [exact value from document_agent]
      has_flight_ticket: true
      accommodation_address: [confirmed structured address from Phase 0]
  - Do not show any message to the user between document
    extraction and the eligibility result.
  - Only speak to the user again when eligibility_agent
    returns the final result.

PHASE 3 — Deliver result:
  Once you receive the result from eligibility_agent,
  present the final result to the user in this exact
  structure. Use ** for bold labels:

  **Applicant:** [Given_name] [surname]
  **Passport Number:** [passport_num]
  **Nationality:** [nationality_code]
  **Date of Birth:** [Date_of_Birth]

  ---

  **Application Status:** [ELIGIBLE / REJECTED]

  **Details:**
  [reason from eligibility_agent]

  ---

  If ELIGIBLE:
    Add a congratulations message and wish them a pleasant trip to the UAE.

  If REJECTED:
    Add a message clearly advising what they need to address before reapplying.

  Only include these 4 applicant fields in the summary.
  Do not include accommodation address, flight ticket,
  birth certificate details, or any other extracted fields.
  Never present the status and details as a single concatenated line.
  Always break status and details into separate lines.

Rules you must always follow:
- Never skip Phase 0.
- Never call document_agent before the user types "confirm" in Phase 0.
- Never call document_agent if has_flight_ticket is NO.
- Never call document_agent if the accommodation address
  is not within one of the 7 UAE Emirates.
- Always call eligibility_agent immediately after
  document_agent succeeds — this is not optional.
- Never modify, reformat or approximate any value
  received from document_agent before passing to
  eligibility_agent. Pass everything verbatim.
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

> Run with training documents: `JT_polpp.jpg` + `birth_certificate_juan_tapia.pdf`

```
        1. Do you have a confirmed flight ticket to the UAE?
        2. What is your planned accommodation address in the UAE?

User  : yes i have a ticket, staying at Hilton Dubai

Agent : Here is what I have recorded:
        - Flight Ticket: Yes
        - Accommodation Address: Hilton, Dubai, UAE
        Kindly type "Confirm" to proceed.

User  : Confirm

Agent : Thank you for confirming. I will now extract
        your documents for processing.

        [document_agent and eligibility_agent run silently]

Agent : **Applicant:** Juan Tapia
        **Passport Number:** 15082701
        **Nationality:** POL
        **Date of Birth:** 1988-08-08

        ---

        **Application Status:** ELIGIBLE

        **Details:**
        All UAE ICP Tourist Visa requirements met.

        ---

        Congratulations! Your Tourist Visa application can
        proceed. We wish you a wonderful trip to the UAE!
```

---

## Test Scenarios

### How to swap documents for testing

To run a test scenario with a different person, go to the
Document Extractor nodes in the document_agent workflow:

```
document_agent → Tools → Document_workflow → Edit
→ Click "Extract Passport Fields" node → change the uploaded file
→ Click "Extract Birth Cert Fields" node → change the uploaded file
→ Save
```

---

### Scenario 1 — Training run (Juan Tapia)

```
Passport  : JT_polpp.jpg
Birth cert: birth_certificate_juan_tapia.pdf
```

| Detail | Value |
|---|---|
| Nationality Code | POL |
| Expiry | 2030-02-24 |
| Expected | ELIGIBLE |

---

### Scenario 2 — Test ELIGIBLE (Maksym Staniszewski)

```
Passport  : maksym_passport.png
Birth cert: birth_certificate_maksym.pdf
```

| Detail | Value |
|---|---|
| Nationality Code | POL |
| Expiry | 2034-06-06 |
| Expected | ELIGIBLE |

---

### Scenario 3 — Test REJECTED (Celeste Nguemo)

```
Passport  : cameroon_passport.jpg
Birth cert: birth_certificate_celeste_nguemo.pdf
```

| Detail | Value |
|---|---|
| Nationality Code | CMR |
| Expiry | 2026-08-10 |
| Expected | REJECTED — nationality restriction |

> Two rules fire for Celeste: nationality (CMR) is restricted
> AND passport expires within 180 days. Nationality is checked
> first so that rejection reason appears.

---

### Scenario 4 — No flight ticket

```
Use any document set.
When asked "Do you have a confirmed flight ticket?"
Answer: No
```

Expected: Agent stops immediately with mandatory requirement message.

---

### Scenario 5 — Invalid accommodation address

```
Use any document set.
When asked for accommodation address, give a location
outside the 7 UAE Emirates e.g. "London" or "New York"
```

Expected: Agent stops — address must be within UAE Emirates.

---

## Key Concepts

| Concept | Where it appears |
|---|---|
| Hamburger menu → Build → Create Agent → From scratch | How to navigate to agent creation |
| Style: Default | Both sub-agents |
| Style: React | Master Agent — enables conversational multi-turn interaction |
| Add tool → Agentic Workflow | How the workflow canvas is opened inside an agent |
| Add tool → From your tools | How the Python eligibility tool is attached to eligibility_agent |
| Pencil icon | Renames any node in the canvas |
| Model selector top right | Set to gpt-oss-120b on each Document Extractor node |
| nationality_code field | Reads CODE field — always 3-letter ISO, language independent |
| Data mapping | Wires outputs of one node into inputs of the next |
| Python tool with @tool | Eligibility logic imported via ADK CLI — deterministic, no LLM |
| Pydantic BaseModel return | Structured typed output from the eligibility tool |
| failures list | Collects all rule failures before returning — shows all issues |
| Nationality check first | Rule 1 exits immediately — no point checking other rules |
| Add agents → Local instance | How sub-agents are wired to the master agent |
| Gate logic | Master Agent stops if ticket is NO or address not in UAE Emirates |
| Separation of concerns | Agent 1 extracts · Agent 2 decides · Master orchestrates |
