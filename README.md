# UAE ICP Agentic Bootcamp

**IBM watsonx Orchestrate**

> **ICP — Visa Eligibility Processing | AI Agent Development Bootcamp**
>
> **Goal:** By the end of this lab, you will have built, deployed, and tested a UAE ICP Tourist Visa Eligibility system on watsonx Orchestrate — a 3-agent pipeline that reads identity documents, collects user information, applies real ICP eligibility rules, and delivers a visa decision.

---

## Table of Contents

- [What is watsonx Orchestrate?](#1-what-is-watsonx-orchestrate)
- [Architecture Overview](#2-architecture-overview)
- [What Gets Checked?](#3-what-gets-checked)
- [Prerequisites](#prerequisites)
- [Documents](#documents)
- [Part 1 — Document Agent](#part-1--build-sub-agent-1-document-agent)
- [Part 2 — Eligibility Agent](#part-2--build-sub-agent-2-eligibility-agent)
- [Part 3 — Master Agent](#part-3--build-the-master-agent)
- [Full Pipeline Test](#full-pipeline-test)
- [Test Scenarios](#test-scenarios)
- [Key Concepts](#key-concepts)

---

## 1. What is watsonx Orchestrate?

IBM watsonx Orchestrate is an open, hybrid enterprise platform for agentic AI. It lets you build intelligent agents that can:

- Reason and make decisions
- Call external tools and APIs
- Process documents automatically
- Run structured, multi-step workflows

### Development Approaches

| Approach | Description |
|---|---|
| No-code | Drag-and-drop UI agent builder |
| Chat to build | Create agents via natural language prompting |
| Pro-code (ADK) | Full control via the Agent Development Kit |
| Flow-builder | Visual agentic workflow builder |

> In this bootcamp we use **No-code UI** for agents and **ADK** for the eligibility tool.

---

## 2. Architecture Overview

We will build a **UAE ICP Tourist Visa Eligibility System** — a 3-agent pipeline that processes identity documents and returns a visa decision.

```
User
 │
 │  answers qualification questions
 │  uploads: Passport + Birth Certificate
 ▼
┌─────────────────────────────────────┐
│   ICP - Visa Eligibility Agent      │  ← Master Agent (watsonx Orchestrate)
│   Orchestrates the full flow        │
└────────────┬────────────────────────┘
             │
     ┌───────┴────────┐
     │                │
     ▼                ▼
┌────────────────┐  ┌──────────────────────────┐
│ document_agent │  │    eligibility_agent      │
│                │  │                          │
│ Agentic        │  │ Python tool (ADK)         │
│ Workflow (UI)  │  │ 6 deterministic rules     │
│                │  │                          │
│ · Upload node  │  │ · Nationality check       │
│ · Passport     │  │ · Passport validity       │
│   extractor    │  │ · DOB cross-check         │
│ · Birth cert   │  │ · Name cross-check        │
│   extractor    │  │ · Flight ticket           │
│ · Package JSON │  │ · Accommodation           │
└───────┬────────┘  └──────────────┬───────────┘
        │                          │
        └────────────┬─────────────┘
                     │
                     ▼
          ┌──────────────────────┐
          │   Visa Decision      │
          │   ELIGIBLE           │
          │   REJECTED + reason  │
          └──────────────────────┘
```

---

## 3. What Gets Checked?

| # | Check | Rule |
|---|---|---|
| 1 | **Nationality** | Restricted nationalities (AFG, LBY, YEM, SOM, SDN, CMR) → REJECTED immediately |
| 2 | **Passport validity** | Must be valid for 180+ days from today |
| 3 | **Date of birth** | DOB on passport must exactly match birth certificate |
| 4 | **Name** | Given name and surname must appear in birth certificate full name |
| 5 | **Flight ticket** | Applicant must have a confirmed flight ticket |
| 6 | **Accommodation** | Must provide a valid UAE address within one of the 7 Emirates |

---



## Prerequisites

Before starting, make sure you have:

- **watsonx Orchestrate SaaS environment** — no account? [Provision a free trial here](https://www.ibm.com/account/reg/us-en/signup?formid=urx-52753)
- **Python 3.11** installed on your machine
- **VS Code** (or any code editor)

### Get your API key and Service instance URL

1. Log in to your watsonx Orchestrate environment
2. Click your **Profile icon** (top-right corner)
3. Click **Settings** → **API details** tab
4. Click **Generate API key** — copy and save it
5. Copy the **Service instance URL** shown below

> ⚠️ Save both values — you will need them in [Part 2, Step 5](#step-5--install-the-adk-and-activate-your-environment)

---

## Documents

Three passports and matching birth certificates are provided. Each has a specific role:

| Role | Person | Passport | Birth Certificate |
|---|---|---|---|
| 🔵 Training — used while building | Juan Tapia | [JT_polpp_passport.jpg](ICP_Documents/Training/JT_polpp_passport.jpg) | [JT_polpp_birth certificate.png](ICP_Documents/Training/JT_polpp_birth%20certificate.png) |
| ✅ Test: ELIGIBLE | Maksym Staniszewski | [maksym_passport.png](ICP_Documents/Testing_True/maksym_passport.png) | [maksym_birth_certificate.png](ICP_Documents/Testing_True/maksym_birth_certificate.png) |
| ❌ Test: REJECTED | Celeste Nguemo | [cameroon_passport.jpg](ICP_Documents/Testing_False/cameroon_passport.jpg) | [cameroon_birth_certificate.png](ICP_Documents/Testing_False/cameroon_birth_certificate.png) |

> During **Part 1** upload the **training documents** (Juan Tapia) into the Document Extractor nodes.
> Swap to test documents during [Test Scenarios](#test-scenarios).

---

## Part 1 — Build Sub-Agent 1: Document Agent

> **Accessing your environment:**
> Open the watsonx Orchestrate instance URL from your welcome email, log in, and you will land on the home page.

---

### 1.1 Create the Agent

```
☰ Hamburger menu → Build → Create Agent → From scratch
```
<img width="1325" height="761" alt="image" src="https://github.com/user-attachments/assets/59fc395d-0c3c-4996-92b2-b124b81fbb36" />
<img width="1336" height="759" alt="image" src="https://github.com/user-attachments/assets/6b1ad2a8-9bd8-4b67-ba37-acf30bbb255a" />


| Field | Value |
|---|---|
| Name | `document_agent` |
| Description | Extracts and returns structured data from passport and birth certificate. Makes no decisions. |

Click **Create**.

Under **Style** select `Default`.
<img width="896" height="745" alt="image" src="https://github.com/user-attachments/assets/eeeec3da-cb39-4f99-a79a-5539c79cf150" />


---

### 1.2 Add the Behaviour

Click the **Behaviour** tab and paste:

```
You are a document extraction agent for UAE ICP visa processing.
When called, run the document extraction workflow.
The workflow will handle reading the passport and birth
certificate and returning the structured data.
Do not make any decisions or assessments about the documents.
Do not add any commentary or explanation to the output.
Return only what the workflow produces.
```
<img width="1142" height="757" alt="image" src="https://github.com/user-attachments/assets/6635d0f9-dfaa-430a-934e-c57beaace47e" />

---

### 1.3 Create the Agentic Workflow

Click the **Toolset** tab on the right side menu → Click **Add tool** → Select **Agentic Workflow**.
<img width="998" height="754" alt="image" src="https://github.com/user-attachments/assets/317c7e75-aab2-4ccd-8409-d439865285ef" />
<img width="1188" height="757" alt="image" src="https://github.com/user-attachments/assets/530b871d-1beb-4d1c-9ddc-09089a1e41bc" />

When prompted, enter a name for the workflow:

```
Document_workflow
```

Click **start building**. This opens the workflow canvas.

> **How to add nodes:**
> Hover over the arrow between two nodes → click the **+** button that appears → select the node type from the menu.

---

### 1.4 Build the Workflow

#### Node 1 & 2 — Collect from User (File Upload)

Click **+** on the arrow between START and END → select **Collect from user → Upload file**
<img width="1317" height="713" alt="image" src="https://github.com/user-attachments/assets/5e13935a-6e67-44a7-8093-181a7e9c960b" />

> **Rename:** Click the **pencil icon** (top-left of node) → type `Passport`
<img width="931" height="745" alt="image" src="https://github.com/user-attachments/assets/f14549ab-f4f5-4eaf-97ca-5d93a30651e4" />

Similarly 1 more node after the previous node by click **+**, Label it `Birth Certificate`

It should now look like this :
<img width="614" height="722" alt="image" src="https://github.com/user-attachments/assets/af4c6b69-b729-4f02-b013-272e39cdf802" />

| Label |
|---|
| `Passport` |
| `Birth Certificate` |

> There are no variable names here — just the label. The workflow waits until both files are uploaded before continuing.

---

#### Node 3 — Document Extractor (Passport)

Click **+** on the arrow between Node 1 and END → select **Add a flow activity → Document extractor**
<img width="1242" height="740" alt="image" src="https://github.com/user-attachments/assets/5cfb007b-ec30-4495-8a97-566bc136b7f5" />

Click on the node to open its configuration panel.

When prompted, select document type: `Unstructured`

> **Rename:** Click the **pencil icon** (top-left of node) → type `Extract Passport Fields`
>
> **Change model:** Click the model selector (top-right of node) → select `gpt-oss-120b`

Drag and drop the training passport file `JT_polpp_passport.jpg` into the document upload area of the node.
<img width="1287" height="758" alt="image" src="https://github.com/user-attachments/assets/80bb723e-265c-418d-ab8b-df124a87392d" />
> This is the training document. 

**Map the document source** — click **Input mapping** → click **`{x}`** on the document field:
<img width="1329" height="749" alt="image" src="https://github.com/user-attachments/assets/b5a4ee33-887a-4509-94d8-87fad71c8610" />

| Input | Source component | Variable to select |
|---|---|---|
| Document | Upload Documents | `Passport` |

Click **Add field** and add these fields:
<img width="1267" height="703" alt="image" src="https://github.com/user-attachments/assets/8611b1fc-8ff3-46f1-90df-6b6811d080b2" />
<img width="793" height="763" alt="image" src="https://github.com/user-attachments/assets/5a915a14-a2dd-4819-be51-50ebbaa5510a" />


| Field name | Type | Description |
|---|---|---|
| `passport number` | string | Passport number as printed on the document |
| `nationality` | string | Nationality as written e.g. POLISH, CAMEROONIAN |
| `Nationality code` | string | 3-letter ISO code from the CODE/KOD field at the top of the passport. NOT the nationality word. `CMR`=Cameroonian, `POL`=Polish, `ARE`=Emirati, `GBR`=British, `IND`=Indian, `USA`=American, `PAK`=Pakistani, `PHL`=Filipino, `EGY`=Egyptian, `AFG`=Afghan, `LBY`=Libyan, `SDN`=Sudanese, `YEM`=Yemeni, `SOM`=Somali |
| `given name` | string | Given name as printed |
| `surname` | string | Surname as printed |
| `Date of Birth` | date | Date of birth — format `YYYY-MM-DD` |
| `date of expiry` | date | Expiry date — format `YYYY-MM-DD` |

---

#### Node 4 — Document Extractor (Birth Certificate)

Click **+** between Node 2 and END → select **Add a flow activity → Document extractor**

Click on the node to open its configuration panel.

Select document type: `Unstructured`

> **Rename:** `Extract Birth Cert Fields`
>
> **Change model:** `gpt-oss-120b`

Drag and drop the training birth certificate file `birth_certificate_juan_tapia.pdf` into the document upload area of the node.

> This is the training document. You will swap it during test scenarios.

**Map the document source** — click **Input mapping** → click **`{x}`** on the document field:

| Input | Source component | Variable to select |
|---|---|---|
| Document | Upload Documents | `Birth Certificate` |

Click **Add field** and add:

| Field name | Type | Description |
|---|---|---|
| `Full name` | string | Full name as written in the Full Name field. Do not duplicate any part of the name. |
| `Date of Birth` | date | Date of birth — format `YYYY-MM-DD` |

---

#### Node 5 — Generative Prompt (Package Output)

Click **+** between Node 3 and END → select **Add a flow activity → Generative prompt**

Click on the node to open its configuration panel.

> **Rename:** `Package Output`

**Input variables** — click the **Input variables** tab → **Add variable**:
<img width="1299" height="757" alt="image" src="https://github.com/user-attachments/assets/4184d3ba-f10d-4a33-b3ed-5ff43dc7c4b2" />

| Variable name | Type |
|---|---|
| `passport_number` | string |
| `nationality` | string |
| `Nationality_code` | string |
| `given_name` | string |
| `surname` | string |
| `Date_of_Birth` | date |
| `date_of_expiry` | date |
| `Birth_Certificate_DOB` | date |
| `Birth_Certificate_Full_Name` | string |

**System Prompt:**

```
You are a data packaging assistant for UAE ICP visa processing.
Your only job is to combine extracted document data into a clean JSON object.
You must return only valid JSON. No explanation, no commentary, no extra text.
Never modify, correct, or interpret any field values.
Always preserve the exact values as given to you.
```

**User Prompt:**

```
Combine the following two documents into a single JSON object
with exactly two keys: "passport" and "birth_certificate".

Passport data:
- Passport Number: {self.input.passport_number}
- Nationality: {self.input.nationality}
- Nationality Code: {self.input.Nationality_code}
- Given Name: {self.input.given_name}
- Surname: {self.input.surname}
- Date of Birth: {self.input.Date_of_Birth}
- Expiry Date: {self.input.date_of_expiry}

Birth Certificate data:
- Date of Birth: {self.input.Birth_Certificate_DOB}
- Full Name: {self.input.Birth_Certificate_Full_Name}

Return only this structure and nothing else:
{
  "passport": {
    "passport_number": {self.input.passport_number},
    "nationality": {self.input.nationality},
    "Nationality_code": {self.input.Nationality_code},
    "given_name": {self.input.given_name},
    "surname": {self.input.surname},
    "Date_of_Birth": {self.input.Date_of_Birth},
    "date_of_expiry": {self.input.date_of_expiry}
  },
  "birth_certificate": {
    "Birth_Certificate_DOB": {self.input.Birth_Certificate_DOB},
    "Birth_Certificate_Full_Name": {self.input.Birth_Certificate_Full_Name}
  }
}
```
<img width="1741" height="977" alt="image" src="https://github.com/user-attachments/assets/7ca1f7a9-6816-4849-bd74-e53aad959e61" />


**Data Mapping** — click **Input mapping** tab → **Add mapping**:

For each input variable, use the variable picker instead of typing manually:

1. Click the **`{x}`** icon on the right side of the variable row
2. A panel opens — click the **source component** name on the left
3. Variables from that node appear on the right — click the one to map

| Input variable | Source component | Variable to select |
|---|---|---|
| `passport_number` | Extract passport | `passport_number` |
| `nationality` | Extract passport | `nationality` |
| `Nationality_code` | Extract passport | `nationality_code` |
| `given_name` | Extract passport | `given_name` |
| `surname` | Extract passport | `surname` |
| `Date_of_Birth` | Extract passport | `date_of_birth` |
| `date_of_expiry` | Extract passport | `date_of_expiry` |
| `Birth_Certificate_DOB` | Extract Birth Certificate | `date_of_birth` |
| `Birth_Certificate_Full_Name` | Extract Birth Certificate | `full_name` |

---

#### Final Canvas

```
START
  │
  ▼
Upload Documents  (Collect from user → Upload file)
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
<img width="274" height="918" alt="image" src="https://github.com/user-attachments/assets/85ce7c1a-f282-4001-a430-05d1a77de53b" />


---

#### END Node — Output Variables

Click the **END** node → **Add** these 9 output variables:

| Variable name | Type |
|---|---|
| `Birth_Certificate_DOB` | date |
| `Birth_Certificate_Full_Name` | string |
| `Date_of_Birth` | date |
| `date_of_expiry` | date |
| `given_name` | string |
| `nationality` | string |
| `Nationality_code` | string |
| `passport_number` | string |
| `surname` | string |

Click **Edit data mapping** and map each variable using the **`{x}`** variable picker:

| Output variable | Source component | Variable to select |
|---|---|---|
| `Birth_Certificate_DOB` | Extract Birth Certificate | `date_of_birth` |
| `Birth_Certificate_Full_Name` | Extract Birth Certificate | `full_name` |
| `Date_of_Birth` | Extract passport | `date_of_birth` |
| `date_of_expiry` | Extract passport | `date_of_expiry` |
| `given_name` | Extract passport | `given_name` |
| `nationality` | Extract passport | `nationality` |
| `Nationality_code` | Extract passport | `nationality_code` |
| `passport_number` | Extract passport | `passport_number` |
| `surname` | Extract passport | `surname` |
<img width="1350" height="1492" alt="image" src="https://github.com/user-attachments/assets/fe79b65d-d71a-4afe-b057-02a6f4208ebb" />

---

### 1.5 Save and Exit

Click **Done** (top-right) to return to the agent page.

---

## Part 2 — Build Sub-Agent 2: Eligibility Agent

### 2.1 Create the Agent

```
☰ Hamburger menu → Build → Create Agent → From scratch
```

| Field | Value |
|---|---|
| Name | `eligibility_agent` |
| Description | Receives extracted document data and user declaration. Runs all UAE ICP eligibility checks via the check_visa_eligibility tool and returns ELIGIBLE or REJECTED with a reason. |

Click **Create**.

Under **Style** select `Default`.

---

### 2.2 Add the Behaviour

Click the **Behaviour** tab and paste:

```
You are a UAE ICP visa eligibility checking agent.
When called, use the check_visa_eligibility tool with all
the inputs provided to you.
Do not modify any input values before passing them to the tool.
Do not add any commentary to the result.
Return only what the tool produces.
```

---

### 2.3 Setup — Import the Eligibility Tool

This step is done outside the browser in VS Code and a terminal.

---

#### Step 1 — Open your IDE

Open **VS Code**. Create a new folder on your desktop called `icp-bootcamp`.

```
File → Open Folder → select icp-bootcamp
```

---

#### Step 2 — Create the tool file

```
File → New File → name it: eligibility_check_tool.py
```

Paste this code and save (`Ctrl+S` / `Cmd+S`):

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
    nationality: str,
    nationality_code: str,
    given_name: str,
    surname: str,
    DOB: date,
    EXP_DATE: date,
    DOB_birth_certificate: date,
    full_name_birth_certificate: str,
    has_flight_ticket: bool,
    accommodation_address: str
) -> VisaEligibilityResult:
    """
    Checks UAE ICP Tourist Visa eligibility based on extracted
    document data and user declaration.

    Args:
        passport_num (str): Passport number
        nationality (str): Full nationality word from passport
        nationality_code (str): 3-letter ISO nationality code
        given_name (str): Given name from passport
        surname (str): Surname from passport
        DOB (date): Date of birth from passport
        EXP_DATE (date): Passport expiry date
        DOB_birth_certificate (date): Date of birth from birth certificate
        full_name_birth_certificate (str): Full name from birth certificate
        has_flight_ticket (bool): Whether applicant has a confirmed flight ticket
        accommodation_address (str): Planned accommodation address in UAE

    Returns:
        VisaEligibilityResult: status (ELIGIBLE or REJECTED) and reason
    """

    def to_date(val):
        if isinstance(val, datetime):
            return val.date()
        elif isinstance(val, str):
            return datetime.strptime(val.strip().split("T")[0], "%Y-%m-%d").date()
        else:
            return val

    exp = to_date(EXP_DATE)
    dob_passport = to_date(DOB)
    dob_cert = to_date(DOB_birth_certificate)

    failures = []

    # ── RULE 1: Nationality restriction — exits immediately ──
    restricted = ["AFG", "LBY", "YEM", "SOM", "SDN", "CMR"]
    if nationality_code.upper() in restricted:
        return VisaEligibilityResult(
            status="REJECTED",
            reason="Your application cannot be processed. Nationals of " + nationality_code +
                   " are currently subject to UAE ICP entry restrictions and are not eligible " +
                   "for a Tourist Visa at this time. Please contact your nearest UAE embassy for further guidance."
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
    if surname.upper() not in full_name_birth_certificate.upper() or \
       given_name.upper() not in full_name_birth_certificate.upper():
        failures.append(
            "Name mismatch — passport: " + (given_name + " " + surname).upper() +
            ", birth certificate: " + full_name_birth_certificate.upper()
        )

    # ── RULE 5: Flight ticket check ──
    if has_flight_ticket is not True:
        failures.append("A confirmed flight ticket is required for UAE Tourist Visa processing.")

    # ── RULE 6: Accommodation check ──
    if not accommodation_address or len(accommodation_address.strip()) < 10:
        failures.append("A valid accommodation address in the UAE is required.")

    if failures:
        return VisaEligibilityResult(
            status="REJECTED",
            reason=" | ".join(failures)
        )

    return VisaEligibilityResult(
        status="ELIGIBLE",
        reason="All UAE ICP Tourist Visa requirements met. Applicant: " +
               given_name + " " + surname +
               " | Passport: " + passport_num +
               " | Nationality: " + nationality
    )
```

---

#### Step 3 — Create the requirements file

```
File → New File → name it: requirement.txt
```

Paste and save:

```
ibm-watsonx-orchestrate
pydantic
```

Your folder should now look like:

```
icp-bootcamp/
├── eligibility_check_tool.py
└── requirement.txt
```
<img width="421" height="261" alt="image" src="https://github.com/user-attachments/assets/a3befb7a-9b69-4cb4-a0bb-3484ddd65922" />


---

#### Step 4 — Open the terminal in VS Code

```
Terminal → New Terminal
```

A terminal panel opens at the bottom of VS Code pointing to your `icp-bootcamp` folder.

---

#### Step 5 — Install the ADK and activate your environment

Install the ADK:

**Windows:**
```bash
pip install ibm-watsonx-orchestrate
```

**Mac:**
```bash
pip3 install ibm-watsonx-orchestrate
```


Add your environment — replace `<your-instance-url>` with the **Service instance URL** you copied in [Prerequisites](#prerequisites):

```bash
orchestrate env add -n ICP -u <your-instance-url>
```

> `-n ICP` is the name for this environment. You will use it every session.
<img width="2314" height="132" alt="image" src="https://github.com/user-attachments/assets/683332e5-0360-46e3-9ef8-755465510b95" />

Activate the environment:

```bash
orchestrate env activate ICP
```

When prompted, enter your **API key** and press Enter.
<img width="2172" height="176" alt="image" src="https://github.com/user-attachments/assets/45bc30ef-963b-4d32-a3b3-cd2d67a51d06" />

---

#### Step 6 — Import the tool

```bash
orchestrate tools import --kind python -r requirement.txt -f eligibility_check_tool.py
```
<img width="2180" height="120" alt="image" src="https://github.com/user-attachments/assets/df91e079-6888-491f-8fee-a2c021fd9445" />


Verify the tool was imported — go to your browser:

```
☰ Hamburger menu → Build → All Tools → check_visa_eligibility
```

If `check_visa_eligibility` appears in the list, the tool is ready. ✅
<img width="1311" height="645" alt="image" src="https://github.com/user-attachments/assets/5c270935-b64f-4cbd-8449-6dd037952e54" />

---

### 2.4 Add the Tool in the UI

```
☰ Hamburger menu → Build → All Agents → eligibility_agent
```

Click the **Toolset** tab on the left side menu → **Add tool** → **Local instance** → select `check_visa_eligibility` → **Add**.

---

### 2.5 Confirm Setup and Test

The `eligibility_agent` is now ready. It has:
- The behaviour instruction telling it to use the tool
- The `check_visa_eligibility` tool attached via Local instance

Click **Preview** → enter this test input in the chat:

```
passport_num: 15082701
nationality: POLISH
nationality_code: POL
given_name: JUAN
surname: TAPIA
DOB: 1988-08-08
EXP_DATE: 2030-02-24
DOB_birth_certificate: 1988-08-08
full_name_birth_certificate: JUAN TAPIA
has_flight_ticket: true
accommodation_address: Hilton Hotel, Sheikh Zayed Road, Dubai, UAE
```

**Expected result:** `All UAE ICP Tourist Visa requirements met. Applicant: JUAN TAPIA | Passport: 15082701 | Nationality: POLISH`

---

## Part 3 — Build the Master Agent

### 3.1 Create the Agent

```
☰ Hamburger menu → Build → Create Agent → From scratch
```

| Field | Value |
|---|---|
| Name | `ICP - Visa Eligibility Agent` |
| Description | UAE ICP Tourist Visa eligibility orchestrator. Qualifies the user upfront, extracts documents, runs eligibility checks and delivers the final visa decision. |

Click **Create**.

Under **Style** select `React`.

---

### 3.2 Welcome Message

Find the **Welcome message** field and paste:

```
Welcome to the UAE Tourist Visa Eligibility Agent!
```

---

### 3.3 Quick Start Prompts

Scroll to **Quick start prompts** → delete all existing questions (click **X** on each) → click **+** and add:

```
Check for Tourist Visa Eligibility
```

---

### 3.4 Add the Behaviour

Click the **Behaviour** tab and paste:

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

Examples:
- User says "im planning to stay in al ain" → capture
  accommodation as "Al Ain, Abu Dhabi, UAE" and only ask
  "Do you have a confirmed flight ticket to the UAE?"

- User says "yes i have a ticket, staying at Hilton Dubai"
  → capture both answers and proceed to confirmation
  without asking anything further.

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
    Examples:
      "Hilton on sheikh zayed road dubai" →
        "Hilton, Sheikh Zayed Road, Dubai, UAE"
      "relatives in Fujairah" → "Fujairah, UAE"
      "my friend's place in abu dhabi" → "Abu Dhabi, UAE"
      "staying in ajman" → "Ajman, UAE"

  - Always append UAE at the end if not already mentioned.

  - Present the structured summary to the user in this format:
    "Here is what I have recorded:
    - Flight Ticket: Yes
    - Accommodation Address: [structured address]

    Kindly type "Confirm" to proceed"

  - Wait for the user to type "confirm". Do not proceed until user types it properly
  - If the user wants to make changes, update the relevant
    detail and show the summary again asking to type "confirm" to proceed.
  - Only proceed to Phase 1 AFTER the user types "confirm" explicitly.

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
      nationality: [exact value from document_agent]
      nationality_code: [exact value from document_agent]
      given_name: [exact value from document_agent]
      surname: [exact value from document_agent]
      DOB: [exact value from document_agent]
      EXP_DATE: [exact value from document_agent]
      DOB_birth_certificate: [exact value from document_agent]
      full_name_birth_certificate: [exact value from document_agent]
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

  **Applicant:** [given_name] [surname]
  **Passport Number:** [passport_num]
  **Nationality:** [nationality]
  **Date of Birth:** [DOB]

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
- Never call document_agent before the user confirms in Phase 0.
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

Click the **Toolset** tab on the right side menu → scroll down to the **Agents** section → **Add agents** → **Local instance** → select:

- `document_agent`
- `eligibility_agent`

Click **Add**.

---

---

## Full Pipeline Test

On the Master Agent page, click the **refresh button** on the top left of the agent chat panel on the right side → click the quick start prompt:
<img width="1346" height="755" alt="image" src="https://github.com/user-attachments/assets/1f6f6b70-b336-4949-a9bd-1a12e87b833f" />

```
Check for Tourist Visa Eligibility
```

> Using training documents: `JT_polpp.jpg` + `birth_certificate_juan_tapia.pdf`

**Expected conversation:**

```
Agent : Welcome to the UAE Tourist Visa Eligibility Agent!
        1. Do you have a confirmed flight ticket to the UAE?
        2. What is your planned accommodation address in the UAE?

User  : yes i have a ticket, staying at Hilton Dubai

Agent : Here is what I have recorded:
        - Flight Ticket: Yes
        - Accommodation Address: Hilton, Dubai, UAE

        Kindly type "Confirm" to proceed

User  : Confirm

Agent : Thank you for confirming. I will now extract your documents for processing.

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

        Congratulations! Your Tourist Visa application can proceed.
        We wish you a wonderful trip to the UAE!
```

---

## Test Scenarios

When the Master Agent asks you to upload your documents, upload the relevant passport and birth certificate for each scenario below.

---

### Scenario 1 — Training Run ✅

Upload when prompted:

| | |
|---|---|
| Passport | [JT_polpp_passport.jpg](ICP_Documents/Training/JT_polpp_passport.jpg) |
| Birth cert | [JT_polpp_birth_certificate.png](ICP_Documents/Training/JT_polpp_birth%20certificate.png) |
| **Expected** | **ELIGIBLE** |

---

### Scenario 2 — Test ELIGIBLE ✅

Upload when prompted:

| | |
|---|---|
| Passport | [maksym_passport.png](ICP_Documents/Testing_True/maksym_passport.png) |
| Birth cert | [maksym_birth_certificate.png](ICP_Documents/Testing_True/maksym_birth_certificate.png) |
| **Expected** | **ELIGIBLE** |

---

### Scenario 3 — Test REJECTED ❌

Upload when prompted:

| | |
|---|---|
| Passport | [cameroon_passport.jpg](ICP_Documents/Testing_False/cameroon_passport.jpg) |
| Birth cert | [cameroon_birth_certificate.png](ICP_Documents/Testing_False/cameroon_birth_certificate.png) |
| **Expected** | **REJECTED — nationality restriction** |

> CMR is in the restricted list. Nationality is checked first so it fires before the expiry check.
---

### Scenario 4 — No Flight Ticket ❌

Use any documents. When asked about a flight ticket, answer **No**.

**Expected:** Agent stops immediately with mandatory requirement message.

---

### Scenario 5 — Invalid Address ❌

Use any documents. When asked for accommodation, give a location outside the UAE (e.g. "London").

**Expected:** Agent stops — address must be within the 7 UAE Emirates.

## Key Concepts

| Concept | Where it appears |
|---|---|
| `☰ → Build → Create Agent → From scratch` | How to create any agent |
| Style: `Default` | Both sub-agents |
| Style: `React` | Master Agent — enables multi-turn conversation |
| Add tool → Agentic Workflow | Opens the workflow canvas inside an agent |
| Add tool → From your tools | Attaches the imported Python tool to eligibility_agent |
| Pencil icon | Renames any node in the canvas |
| Model selector (top-right) | Set to `gpt-oss-120b` on Document Extractor nodes |
| `Nationality code` field | Reads CODE field — always 3-letter ISO regardless of language |
| Input variables + Data mapping | Wires outputs of one node into inputs of the next |
| `@tool` decorator | Registers the Python function as a watsonx Orchestrate tool |
| `Pydantic BaseModel` | Structured typed output from the eligibility tool |
| `failures` list | Collects all rule failures — returns all issues at once |
| Nationality checked first | Exits immediately for restricted nationalities |
| Add agents → Local instance | How sub-agents are wired to the master |
| Gate logic | Master stops if ticket is NO or address is not in UAE |
| Separation of concerns | Agent 1 extracts · Agent 2 decides · Master orchestrates |
