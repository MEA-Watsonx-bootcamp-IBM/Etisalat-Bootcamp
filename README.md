# Etisalat Postpaid Plan Eligibility Bootcamp

**IBM watsonx Orchestrate**

> **Postpaid Plan Eligibility Processing | AI Agent Development Bootcamp**
>
> **Goal:** By the end of this lab, you will have built, deployed, and tested an Etisalat Postpaid Plan Eligibility system on watsonx Orchestrate — a 3-agent pipeline that reads Emirates ID and payslip documents, validates eligibility requirements, recommends eligible plan tiers, and processes payment via Stripe.

---

## Table of Contents

- [What is watsonx Orchestrate?](#1-what-is-watsonx-orchestrate)
- [Architecture Overview](#2-architecture-overview)
- [What Gets Checked?](#3-what-gets-checked)
- [Prerequisites](#prerequisites)
- [Documents](#documents)
- [Part 1 — Document Agent](#part-1--build-sub-agent-1-document-agent)
- [Part 2 — Payment Agent](#part-2--build-sub-agent-2-payment-agent)
- [Part 3 — Master Agent](#part-3--build-the-master-agent)
- [Part 4 — Full Pipeline Test](#part-4--full-pipeline-test)
- [Part 5 — Test Scenarios](#part-5--test-scenarios)

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

> In this bootcamp we use **No-code UI** for agents, **Flow-builder** for document extraction workflow, and **ADK** for the payment tool.

---

## 2. Architecture Overview

We will build an **Etisalat Postpaid Plan Eligibility System** — a 3-agent pipeline that processes identity documents, validates eligibility, and processes payment.

```
User
 │
 │  uploads: Emirates ID + Payslip
 ▼
┌─────────────────────────────────────┐
│   Postpaid Eligibility Agent         │  ← Master Agent (watsonx Orchestrate)
│   Orchestrates the full flow        │     Style: React
└────────────┬────────────────────────┘
             │
     ┌───────┴────────┐
     │                │
     ▼                ▼
┌────────────────┐  ┌──────────────────────────┐
│ document_agent │  │    payment_agent          │
│                │  │                          │
│ Agentic        │  │ Python tool (ADK)         │
│ Workflow (UI)  │  │ Stripe integration        │
│ Style: Default │  │ Style: Default            │
│                │  │                          │
│ · User upload  │  │ · create_payment_link     │
│ · Emirates ID  │  │ · Test mode checkout      │
│   extractor    │  │ · Returns payment URL     │
│ · Payslip      │  │                          │
│   extractor    │  └──────────────────────────┘
│ · Eligibility  │
│   check script │
│ · Package JSON │
│ · Knowledge    │
│   base lookup  │
└────────────────┘
         │
         ▼
  ┌──────────────────────┐
  │   Eligibility Result │
  │   PASS → Plan tiers  │
  │   FAIL → Reason      │
  └──────────────────────┘
         │
         ▼
  ┌──────────────────────┐
  │   Payment Link       │
  │   Stripe Checkout    │
  └──────────────────────┘
```

---

## 3. What Gets Checked?

| # | Check | Rule |
|---|---|---|
| 1 | **Emirates ID validity** | Must be valid for 90+ days from today |
| 2 | **Name cross-check** | Name on Emirates ID must match payslip employee name |
| 3 | **Salary threshold** | Gross salary must be ≥ 4000 AED |

If all checks pass, the system retrieves eligible plan tiers from the knowledge base based on the customer's salary and presents them for selection.

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

Three sets of Emirates ID and payslip documents are provided. Each has a specific role:

| Role | Files | Location |
|---|---|---|
| 🔵 Training — used while building | EID_Train.png, Payslip_Train.png | [documents/training/](documents/training/) |
| ✅ Test: PASS | EID_Pass.png, Payslip_Pass.png | [documents/Pass Test/](documents/Pass%20Test/) |
| ❌ Test: FAIL | EID_Fail.png, Payslip_Fail.png | [documents/Fail Test/](documents/Fail%20Test/) |

> During **Part 1** upload the **training documents** into the Document Extractor nodes.
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

<img width="1325" height="761" alt="image" src="https://github.com/user-attachments/assets/81257616-56a0-40b8-9ef4-fa8937218d4f" />
<img width="1336" height="759" alt="image" src="https://github.com/user-attachments/assets/2c7784c5-890d-445c-b5dc-4559231f9997" />

| Field | Value |
|---|---|
| Name | `document_agent` |
| Description | Extracts structured information from uploaded Emirates ID documents and payslips for postpaid eligibility verification. Reads Emirates ID and extracts full name, ID number, date of birth, expiry date. Reads payslip and extracts employee name, company name, gross salary, pay period. Returns all extracted fields in structured format for downstream eligibility validation. |

Click **Create**.

Under **Style** select `Default`.

<img width="468" height="408" alt="image" src="https://github.com/user-attachments/assets/95e27da2-61d0-4668-9d2c-26d47f3c55eb" />

---

### 1.2 Add the Behaviour

Click the **Behaviour** tab and paste:

```
call document_extract_tool.

After the tool finishes, return the complete tool output to the user.

Do not only say the documents were processed.
Do not summarize unless the tool output is shown.
Always show the extracted fields and eligibility status.

If eligibility status is PASS:
Retrieve postpaid plans from the knowledge base.
Compare the customer salary against each plan minimum salary.
Return all eligible plans with plan name, rental amount, and credit limit.

If eligibility status is FAIL:
Do not retrieve plans.
Only explain the rejection reason.
```

<img width="468" height="418" alt="image" src="https://github.com/user-attachments/assets/2352bb0f-667e-4f2b-8244-720ad60405ee" />

---

### 1.3 Create the Agentic Workflow

Click the **Toolset** tab on the right side menu → Click **Add tool** → Select **Agentic Workflow**.

<img width="468" height="420" alt="image" src="https://github.com/user-attachments/assets/a600fb02-69be-45ca-8a5d-bfabeece3335" />
<img width="1188" height="757" alt="image" src="https://github.com/user-attachments/assets/3fc0fab4-fce3-4997-99b3-807f2943f64b" />

When prompted, enter a name for the workflow:

```
document_extract_tool
```

Click **start building**. This opens the workflow canvas.

> **How to add nodes:**
> Hover over the arrow between two nodes → click the **+** button that appears → select the node type from the menu.

---

### 1.4 Build the Workflow

#### Node 1 & 2 — Collect from User (File Upload)

Click **+** on the arrow between START and END → select **Collect from user → Upload file**

<img width="1317" height="713" alt="image" src="https://github.com/user-attachments/assets/5369bbce-0902-4928-844e-d4a3a05a8b04" />

> **Rename:** Click the **pencil icon** (top-left of node) → type `Emirates ID`

<img width="468" height="289" alt="image" src="https://github.com/user-attachments/assets/2ebabfe8-a5f4-4c81-ae90-9f6dea867921" />

Similarly add 1 more node after the previous node by clicking **+**, Label it `Payslip`

It should now look like this with two upload nodes:

<img width="468" height="619" alt="image" src="https://github.com/user-attachments/assets/a80414d0-52e0-46fc-9908-2682d8e5fdef" />


| Label |
|---|
| `Emirates ID` |
| `Payslip` |

> There are no variable names here — just the label. The workflow waits until both files are uploaded before continuing.

---

#### Node 3 — Document Extractor (Emirates ID)

Click **+** on the arrow between Node 1 and END → select **Add a flow activity → Document extractor**

<img width="440" height="267" alt="image" src="https://github.com/user-attachments/assets/9d1b65a2-f901-4a38-8237-7d85b498fcf0" />

Click on the node to open its configuration panel.

When prompted, select document type: `Unstructured`

<img width="438" height="285" alt="image" src="https://github.com/user-attachments/assets/0da8020e-9f02-4e87-8f4e-4d55a770db59" />

> **Rename:** Click the **pencil icon** (top-left of node) → type `Extract emirates ID fields`
>
> **Change model:** Click the model selector (top-right of node) → select `gpt-oss-120b`

Drag and drop the training Emirates ID file `EID_Train.png` into the document upload area of the node.

<img width="468" height="274" alt="image" src="https://github.com/user-attachments/assets/fdd034e8-2002-4160-a3d2-d425dcdc272c" />

> This is the training document.

Click **Add field** and add these fields:

| Field name | Type | Description |
|---|---|---|
| `ID Number` | string | Extract the Emirates ID number exactly as shown on the card, usually in the format 784-XXXX-XXXXXXX-X. |
| `Full Name` | string | Extract the card holder's full name exactly as written in English on the Emirates ID. |
| `Date of Birth` | date | Extract the card holder's date of birth exactly as shown on the Emirates ID. Return in YYYY-MM-DD format if possible. |
| `Nationality` | string | Extract the card holder's nationality from the Emirates ID. Look for the label "Nationality" or "الجنسية". Return only the nationality value, not the label. The value may be a country name such as United Arab Emirates, Saudi Arabia, India, Pakistan, Egypt, Philippines, Jordan, Syria, or another nationality. If both English and Arabic are shown, return the English nationality. |
| `Expiry Date` | date | Extract the expiry date of the Emirates ID exactly as shown on the card. This field will be used later for eligibility validation. |
| `Date of Issue` | string | Extract the issue date of the Emirates ID exactly as shown on the card. |

<img width="468" height="258" alt="image" src="https://github.com/user-attachments/assets/fed473e2-de25-44fa-a4d0-600c24a209e2" />

<img width="468" height="261" alt="image" src="https://github.com/user-attachments/assets/a075dceb-5a3f-48c8-ba3b-ec29139da798" />

**Map the document source:**

1. Click **X** (top-right of the panel) to close it
2. Click the `Extract emirates ID fields` node again to reopen it
3. At the bottom of the panel, click the **settings icon** (⚙) next to **Edit data mapping**
4. Click **`{x}`** on the `document_ref` field
5. Under **User activity 1**, select `Emirates ID`
6. On the right side, select `value`

---

#### Node 4 — Document Extractor (Payslip)

Click **+** between Node 2 and END → select **Add a flow activity → Document extractor**

Click on the node to open its configuration panel.

Select document type: `Unstructured`

> **Rename:** `Extract payslip fields`
>
> **Change model:** `gpt-oss-120b`

Drag and drop the training payslip file `Payslip_Train.png` into the document upload area of the node.

> This is the training document. You will swap it during test scenarios.

Click **Add field** and add:

| Field name | Type | Description |
|---|---|---|
| `Employee Name` | string | Extract the employee's full name exactly as written in the payslip, usually found next to the label "Employee Name" or "Name" |
| `gross salary` | string | Extract the gross salary amount from the payslip exactly as shown next to "Gross Salary". Return only the numeric value without currency symbols or commas. |

**Map the document source:**

1. Click **X** (top-right of the panel) to close it
2. Click the `Extract payslip fields` node again to reopen it
3. At the bottom of the panel, click the **settings icon** (⚙) next to **Edit data mapping**
4. Click **`{x}`** on the `document_ref` field
5. Under **User activity 1**, select `Payslip`
6. On the right side, select `value`

---

#### Node 5 — Logic Block (Eligibility Check)

Click **+** between Node 4 and END → select **Add a flow activity → Logic block**

<img width="468" height="235" alt="image" src="https://github.com/user-attachments/assets/1b288b54-ba99-4f07-aded-b18dc4f78677" />

Click on the node to open its configuration panel.

> **Rename:** `Eligibility Check`

<img width="468" height="235" alt="image" src="https://github.com/user-attachments/assets/d9eb3f04-36c2-4eb6-b923-813d2f039c1b" />

**Logic block code** — paste this Python code:

<img width="468" height="236" alt="image" src="https://github.com/user-attachments/assets/9ee3b9f8-15f6-4b49-92eb-cb858fd5fe61" />

```python
# ---- Pull extracted fields from the two upstream document extractor nodes ----
id_fields = parent["Extract emirates ID fields"].output
payslip_fields = parent["Extract payslip fields"].output

id_name_raw = id_fields.get("Full Name", "")
payslip_name_raw = payslip_fields.get("Employee Name", "")
expiry_str = id_fields.get("Expiry Date", "")
gross_salary = float(payslip_fields.get("gross_salary", 0))

# ---- Check 1: Cross-name validation ----
def normalize_name(name):
    name = name.lower().strip()
    name = re.sub(r"[^a-z\s]", "", name)
    name = re.sub(r"\s+", " ", name)
    return name

id_name_norm = normalize_name(id_name_raw)
payslip_name_norm = normalize_name(payslip_name_raw)

id_tokens = set(id_name_norm.split())
payslip_tokens = set(payslip_name_norm.split())

if len(id_tokens) == 0 or len(payslip_tokens) == 0:
    name_match = False
elif id_tokens.issubset(payslip_tokens) or payslip_tokens.issubset(id_tokens):
    name_match = True
else:
    common = id_tokens.intersection(payslip_tokens)
    name_match = len(common) >= 2

# ---- Check 2: Expiry date (fail if missing, expired, or expiring within 3 months) ----
today = datetime.date.today()
three_months_out = today + datetime.timedelta(days=90)

expiry_str_clean = expiry_str.strip() if expiry_str else ""
expiry_date = None

if expiry_str_clean != "":
    # ISO 8601 is the platform's native date format; others are fallbacks
    # in case the extractor returns a differently formatted string.
    date_formats = [
        "%Y-%m-%d",    # ISO 8601 - native flow date format
        "%d/%m/%Y",
        "%d-%m-%Y",
        "%m/%d/%Y",
        "%d %B %Y",
        "%d %b %Y",
    ]
    for fmt in date_formats:
        try:
            expiry_date = datetime.datetime.strptime(expiry_str_clean, fmt).date()
            break
        except ValueError:
            continue

if expiry_date is None:
    id_valid = False
elif expiry_date < today:
    id_valid = False
elif expiry_date <= three_months_out:
    id_valid = False
else:
    id_valid = True

# ---- Check 3: Salary threshold ----
if gross_salary >= 4000:
    salary_pass = True
else:
    salary_pass = False

# ---- Overall result: ALL checks must pass ----
if name_match and id_valid and salary_pass:
    status = "PASS"
    reason = "All checks passed"
else:
    if not name_match:
        reason = "Name on Emirates ID does not match payslip"
    elif not id_valid:
        if expiry_str_clean == "":
            reason = "Emirates ID expiry date missing or unreadable"
        else:
            reason = "Emirates ID expired or expiring within 3 months"
    else:
        reason = "Salary below 4000"
    status = "FAIL"

# ---- Outputs for downstream nodes ----
self.output.status = status
self.output.reason = reason
```

**Output schema** — click **Output variables** tab → **Add variable**:

| Variable name | Type | Description |
|---|---|---|
| `status` | string | Final eligibility status (PASS or FAIL) |
| `reason` | string | Reason for approval or rejection |

<img width="468" height="236" alt="image" src="https://github.com/user-attachments/assets/9fb2ebe8-0c42-4e50-81c3-d1bf34eccf55" />

<img width="468" height="236" alt="image" src="https://github.com/user-attachments/assets/c6a217f9-0090-4d69-92f9-9369c058f138" />


---

#### Node 6 — Generative Prompt (Package Output)

Click **+** between Node 5 and END → select **Add a flow activity → Generative prompt**

Click on the node to open its configuration panel.

> **Rename:** `Generative prompt`

**Input variables** — click the **Input variables** tab → **Add variable**:

| Variable name | Type |
|---|---|
| `full_name` | string |
| `id_number` | string |
| `expiry_date` | string |
| `nationality` | string |
| `gross_salary` | string |
| `date_of_birth` | string |
| `employee_name` | string |
| `status` | string |
| `reason` | string |

**System Prompt:**

```
You are a data packaging assistant for telecom eligibility processing.
Your only job is to combine extracted document data into a clean JSON object.
You must return only valid JSON. No explanation, no commentary, no extra text.
Never modify, correct, or interpret any field values.
Always preserve the exact values as given to you.
```

**User Prompt:**

```
Combine the following two documents into a single JSON object
with exactly two keys: "emirates_id" and "payslip".

Emirates ID data:
- ID Number: {self.input.id_number}
- Full Name: {self.input.full_name}
- Date of Birth: {self.input.date_of_birth}
- Nationality: {self.input.nationality}
- Expiry Date: {self.input.expiry_date}

Payslip data:
- Employee Name: {self.input.employee_name}
- Gross Salary: {self.input.gross_salary}

Return only this structure and nothing else:

{
  "emirates_id": {
    "id_number": {self.input.id_number},
    "full_name": {self.input.full_name},
    "date_of_birth": {self.input.date_of_birth},
    "nationality": {self.input.nationality},
    "expiry_date": {self.input.expiry_date}
  },
  "payslip": {
    "employee_name": {self.input.employee_name},
    "gross_salary": {self.input.gross_salary}
  },
"eligibility": {
    "status": status,
    "reason": reason
  }
}
```

**Data Mapping:**

1. Click **X** (top-right of the panel) to close it
2. Click the `Generative prompt` node again to reopen it
3. At the bottom of the panel, click the **settings icon** (⚙) next to **Edit data mapping**

For each input variable, click **`{x}`** and use the variable picker:

| Input variable | Source component | Variable to select |
|---|---|---|
| `full_name` | Extract emirates ID fields | `full_name` |
| `id_number` | Extract emirates ID fields | `id_number` |
| `expiry_date` | Extract emirates ID fields | `expiry_date` |
| `nationality` | Extract emirates ID fields | `nationality` |
| `date_of_birth` | Extract emirates ID fields | `date_of_birth` |
| `employee_name` | Extract payslip fields | `employee_name` |
| `gross_salary` | Extract payslip fields | `gross_salary` |
| `status` | Eligibility Check | `status` |
| `reason` | Eligibility Check | `reason` |

---

#### Final Canvas

```
START
  │
  ▼
User activity 1  (Collect from user → Upload files: Emirates ID, Payslip)
  │
  ▼
Extract emirates ID fields  (Document Extractor)
  │
  ▼
Extract payslip fields  (Document Extractor)
  │
  ▼
Eligibility Check  (Logic block)
  │
  ▼
Generative prompt  (Package Output)
  │
  ▼
END
```

---

### 1.5 Save and Exit

Click **Done** (top-right) to return to the agent page.

---
### 1.6 Add Knowledge Base

Download the plans CSV file from the lab export folder:

```
Etisalat/knowledge-bases/plans.csv
```

This file contains the postpaid plan tiers with their eligibility requirements:

| tier | plan_name | min_salary_aed | rental_aed | credit_limit_aed |
|---|---|---|---|---|
| 1 | Smart 55 | 4000 | 55 | 500 |
| 2 | Smart 150 | 6000 | 150 | 1200 |
| 3 | Smart 350 | 10000 | 350 | 2500 |
| 4 | Smart 500+ | 15000 | 500 | 4000 |

While still on the document_agent page, click the **Toolset** tab on the left side menu → scroll down to **Knowledge** section → **Add source** → select **New knowledge**.

Scroll down and choose **Upload files** → click **Next**.

Drag and drop the `plans.csv` file you downloaded into the upload area → click **Next**.

Enter the knowledge base details:

| Field | Value |
|---|---|
| Name | `Postpaid Plans Knowledge Base` |
| Description | This knowledge base contains telecom postpaid plans and their eligibility requirements, including minimum salary thresholds, plan names, monthly fees, benefits, and available tiers. Use this knowledge source after eligibility validation to retrieve all plans the customer qualifies for based on gross salary and return matching plans ranked by eligibility. |

Click **Save**.

---


## Part 2 — Build Sub-Agent 2: Payment Agent

### 2.1 Create the Agent

```
☰ Hamburger menu → Build → Create Agent → From scratch
```

| Field | Value |
|---|---|
| Name | `payment_agent` |
| Description | Handles payment collection for a postpaid plan the user has selected. Creates a Stripe test-mode checkout link for the chosen plan's monthly rental amount and shares it with the user to complete payment. |

Click **Create**.

Under **Style** select `Default`.

---

### 2.2 Add the Behaviour

Click the **Behaviour** tab and paste:

```
You handle payment collection once a user has selected a postpaid plan.

When you receive a plan name and its monthly rental amount in AED:

1. Call create_payment_link with the plan_name and rental_aed.
2. If status is CREATED, respond with only the payment_url as a markdown hyperlink with clear link text — no greeting, no confirmation phrase, no introductory sentence of your own. The master agent adds its own framing before relaying your response, so your entire response should be just the link itself. The URL itself is a long opaque token — it may contain many characters after a "#" symbol that look like random text or encoded data. This is normal and expected. You must copy the entire payment_url exactly as returned by the tool, character for character, with nothing shortened, summarized, truncated, "cleaned up," or rewritten. Never drop, trim, or simplify any part of it, including everything after the "#".

   Format it like this, where [the actual payment_url value returned by the tool] is replaced with the real, complete URL:
   [Click here to pay for Smart 150]([the actual payment_url value returned by the tool])
3. If status is FAILED, tell the user clearly that the payment link could not be created and share the reason. Do not retry silently — ask the user if they'd like to try again.

Never ask the user for card details directly. Payment is always completed on Stripe's hosted checkout page via the link you provide.

Never paste the raw URL as plain visible text outside the markdown link syntax — but the URL inside the parentheses of the markdown link must always be the complete, unmodified payment_url value. A shortened, truncated, or partially reproduced URL will not work and will break the payment flow.
```

---

### 2.3 Setup — Import the Payment Tool

This step is done outside the browser in VS Code and a terminal.

---

#### Step 1 — Open your IDE

Open **VS Code**. Create a new folder on your desktop called `etisalat-bootcamp`.

```
File → Open Folder → select etisalat-bootcamp
```

---

#### Step 2 — Create the tool file

```
File → New File → name it: create_payment_link.py
```

Paste this code and save (`Ctrl+S` / `Cmd+S`):

```python
from ibm_watsonx_orchestrate.agent_builder.tools import tool
from pydantic import BaseModel, Field
import stripe

stripe.api_key = "sk_test_51Tls7GRriMoAjNdV3T7efWnrIBRQLDuGUEcfXk4oJHQhVAVxjpMv8FDstHZ0qHonFTmqjI9bT5OJ8pwmbnfRmgEj00iPaeMGpY"

SUCCESS_URL = "https://6a3bf686c40af40b0f021c31--dashing-pothos-4e1f45.netlify.app/"
CANCEL_URL = "https://6a3bf7055d04772eedef4cc0--animated-entremet-8a202b.netlify.app/"


class PaymentLinkResult(BaseModel):
    status: str = Field(description="CREATED or FAILED")
    payment_url: str = Field(description="Stripe Checkout URL the user opens to pay, empty if creation failed")
    session_id: str = Field(description="Stripe Checkout Session ID, empty if creation failed")
    reason: str = Field(description="Explanation of the result")


@tool(
    name="create_payment_link",
    description="""Creates a Stripe Checkout payment link (test mode) for
    a selected postpaid plan's monthly rental amount.

    Use this tool once the user has chosen a specific plan tier from the
    eligible options. It creates a Stripe-hosted Checkout Session for
    that plan's rental amount and returns a payment URL.

    The user must open the returned payment_url in a browser to complete
    payment on Stripe's hosted page using a test card (e.g.
    4242 4242 4242 4242, any future expiry, any CVC). No real charge is
    made — this runs in Stripe test mode.

    Returns status (CREATED or FAILED), the payment_url, the Stripe
    session_id, and a reason."""
)
def create_payment_link(
    plan_name: str,
    rental_aed: float
) -> PaymentLinkResult:
    """
    Creates a Stripe Checkout Session (test mode) for the selected
    postpaid plan's monthly rental amount.

    Args:
        plan_name (str): Name of the selected plan, e.g. "Smart 150"
        rental_aed (float): Monthly rental amount in AED, e.g. 150

    Returns:
        PaymentLinkResult: status, payment_url, session_id, and reason
    """

    # ── Basic input validation — exits immediately ──
    if not plan_name or not plan_name.strip():
        return PaymentLinkResult(
            status="FAILED",
            payment_url="",
            session_id="",
            reason="Plan name is required to create a payment link."
        )

    if rental_aed is None or rental_aed <= 0:
        return PaymentLinkResult(
            status="FAILED",
            payment_url="",
            session_id="",
            reason="Rental amount must be a positive number."
        )

    # ── Create the Stripe Checkout Session ──
    try:
        session = stripe.checkout.Session.create(
            payment_method_types=["card"],
            mode="payment",
            line_items=[
                {
                    "price_data": {
                        "currency": "aed",
                        "product_data": {
                            "name": plan_name + " — Monthly rental"
                        },
                        "unit_amount": int(round(rental_aed * 100)),
                    },
                    "quantity": 1,
                }
            ],
            success_url=SUCCESS_URL,
            cancel_url=CANCEL_URL,
        )
    except Exception as e:
        return PaymentLinkResult(
            status="FAILED",
            payment_url="",
            session_id="",
            reason="Stripe checkout session could not be created: " + str(e)
        )

    return PaymentLinkResult(
        status="CREATED",
        payment_url=session.url,
        session_id=session.id,
        reason="Checkout session created for " + plan_name + " at " + str(rental_aed) + " AED/month."
    )
```

---

#### Step 3 — Create the requirements file

```
File → New File → name it: requirements.txt
```

Paste and save:

```
stripe>=11.0.0
ibm-watsonx-orchestrate==2.5.1
```

Your folder should now look like:

```
etisalat-bootcamp/
├── create_payment_link.py
└── requirements.txt
```

---

#### Step 4 — Open the terminal in VS Code

```
Terminal → New Terminal
```

A terminal panel opens at the bottom of VS Code pointing to your `etisalat-bootcamp` folder.

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
orchestrate env add -n EtisalatBootcamp -u <your-instance-url>
```

> `-n EtisalatBootcamp` is the name for this environment. You will use it every session.

Activate the environment:

```bash
orchestrate env activate EtisalatBootcamp
```

When prompted, enter your **API key** and press Enter.

---

#### Step 6 — Import the tool

```bash
orchestrate tools import --kind python -r requirements.txt -f create_payment_link.py
```

Verify the tool was imported — go to your browser:

```
☰ Hamburger menu → Build → All Tools → create_payment_link
```

If `create_payment_link` appears in the list, the tool is ready. ✅

---

### 2.4 Add the Tool in the UI

```
☰ Hamburger menu → Build → All Agents → payment_agent
```

Click the **Toolset** tab on the left side menu → **Add tool** → **Local instance** → select `create_payment_link` → **Add**.

---

## Part 3 — Build the Master Agent

### 3.1 Create the Agent

```
☰ Hamburger menu → Build → Create Agent → From scratch
```

| Field | Value |
|---|---|
| Name | `Postpaid Eligibility Agent` |
| Description | Helps users check eligibility for Etisalat postpaid plans and complete sign-up. Collects Emirates ID and payslip, verifies identity and income requirements, recommends eligible plan tiers, and processes payment for the plan the user selects. |

Click **Create**.

Under **Style** select `React`.

---

### 3.2 Welcome Message

Find the **Welcome message** field and paste:

```
Hi, Welcome to e& Plan Eligibility Agent
```

---

### 3.3 Quick Start Prompts

Scroll to **Quick start prompts** → delete all existing questions (click **X** on each) → click **+** and add:

```
Check my postpaid plan eligibility
```

---

### 3.4 Add the Behaviour

Click the **Behaviour** tab and paste:

```
You are the Postpaid Plan Eligibility assistant.
You are the only agent that communicates with the user.
You manage the full postpaid eligibility and sign-up flow step by step.

PHASE 0 — Kickoff:
Call document_agent immediately at the very start of every conversation, with no greeting or message of your own first. document_agent owns the document upload, extraction, eligibility checking, and knowledge base plan lookup entirely through its own workflow and behavior.

PHASE 1 — Document extraction and eligibility check:
  - Triggered immediately and automatically at the very start of the conversation. Do not send any message of your own first, and do not wait for any input before calling document_agent.
  - Call document_agent immediately. document_agent will handle asking the user to upload their Emirates ID and payslip, run its own extraction workflow, check eligibility, and — if eligible — retrieve matching plans from its knowledge base.
  - Wait for document_agent to return its full result. Do not interject, ask your own questions, or duplicate any upload prompts while document_agent is handling this phase.
  - As soon as document_agent returns a result, proceed immediately to Phase 2 in the same turn — do not pause and do not wait for any additional input.
  - If document_agent returns INCOMPLETE or fails to extract a required field, relay that back to the user clearly so they know what to re-upload. Do not proceed to Phase 2 until extraction succeeds.

PHASE 2 — Deliver result:
  Once you receive the result from document_agent:

  If the result is a decline:
    - Clearly explain the decline reason to the user in plain language.
    - Do not proceed further. Do not call payment_agent. Do not ask the user to pick a plan.

  If the result is a list of eligible tiers:
    - Present every eligible tier as a markdown table with columns: Plan, Monthly Rental (AED), Credit Limit (AED).
    - Beneath the table, add one short recommendation line suggesting the highest eligible tier as the best value (e.g. "Based on your eligibility, [highest plan name] offers the most data and the highest credit limit for your budget."). This is a suggestion only — it must not replace or skip showing the full table of all eligible tiers above it.
    - Ask the user which plan they would like to proceed with.
    - Never pre-select, assume, or recommend only the highest tier as if it were the only option — all eligible tiers must be shown.
    - Wait for the user to name one specific plan before proceeding. If their answer doesn't clearly match one of the listed plans, ask them to choose again from the listed options.

PHASE 3 — Plan confirmation:
  - Once the user names one specific eligible plan, do NOT call payment_agent yet. First present a confirmation summary of that plan, in this format:

    "Here is the plan you've selected:
    - Plan: [plan name]
    - Monthly Rental: [rental amount] AED
    - Credit Limit: [credit limit] AED

    Kindly type "Confirm" to proceed with payment."

  - Wait for the user to type "confirm" (accept reasonable case variations, e.g. "Confirm", "confirm", "CONFIRM"). Do not call payment_agent until the user has typed it.
  - If the user wants to switch to a different eligible plan instead of confirming, update the selection and show the confirmation summary again for the newly chosen plan. Always re-confirm before proceeding.
  - As soon as the user types "confirm", proceed directly and immediately to Phase 4 in the same turn. Do not pause, do not ask any further questions, and do not wait for any additional user input before calling payment_agent.

PHASE 4 — Payment:
  - Triggered immediately and automatically the moment the user types "confirm" in Phase 3. Do not wait for any other input.
  - Call payment_agent immediately. Pass the plan details as a single labeled string in this exact format:

    "Plan: [plan name]
    Monthly Rental: [monthly rental amount] AED"

    Example:

    "Plan: Smart 150
    Monthly Rental: 150 AED"

  - Use the exact plan name and exact rental amount as returned by document_agent. Never invent, round, or alter the rental amount. Never include the credit limit in this string — only the plan name and the monthly rental amount with the AED unit.
  - Once payment_agent responds, if status is CREATED, begin your message with exactly this line: "Thank you, please proceed with the payment using the following link:" — then on the next line, output payment_agent's response exactly as-is, with no changes whatsoever. Do not summarize it, do not rephrase it, do not regenerate the link, and do not add your own wording around it beyond the exact opening line specified above.
  - CRITICAL: Treat payment_agent's entire response as an opaque block of text that you copy, not text that you read, understand, or reason about. Do not attempt to parse, interpret, decode, or analyze the URL inside it. Do not try to determine what the URL "means" or whether it "looks complete." Your only job with this block of text is character-for-character copying — never regeneration. If you find yourself reasoning step by step about the structure or content of the URL, stop that reasoning immediately and instead copy the original text directly. The correct process is: take payment_agent's response, copy it, paste it into your reply unchanged. Do not retype it from memory, do not summarize it and then expand it back out, do not describe it and then reconstruct it. Treat it the same way you would treat copying a block of text without reading it.
  - Never ask the user "would you like me to generate a payment link" or any similar confirmatory question before calling payment_agent. Once the user has typed "confirm" in Phase 3, proceed directly to generating and presenting the payment link — do not ask again.
  - The URL inside the payment link is long and may look unfamiliar or repetitive after a "#" character. This is normal, expected, and not a display error to fix. You must never insert an ellipsis ("...", "…"), never insert any "and so on" style abbreviation, never insert any invisible or zero-width characters, and never substitute any portion of the real URL with a shorter placeholder, a made-up fragment, or anything that merely resembles the original pattern. The URL must be reproduced as one single unbroken sequence of the exact characters payment_agent provided, with nothing added, nothing removed, and nothing replaced — including inside the part of the URL after the "#" symbol. A URL that is shorter than what payment_agent returned is always wrong, with no exceptions.
  - If the user changes their mind after receiving a payment link and names a different eligible plan instead, treat this as a new selection: return to Phase 3 and present the exact confirmation summary template again for the newly chosen plan (do not phrase this as a casual question like "would you like me to generate a new link"). Wait for a fresh "confirm" from the user, then repeat Phase 4 and call payment_agent again. Do not refer back to the previous link.

FORMATTING:
Always format your messages clearly and readably. When presenting eligible plans, list them as bullet points, not as a single paragraph. Use line breaks generously so the message is easy to scan rather than a wall of text.

Rules you must always follow:
- Never skip Phase 0. Always call document_agent immediately at the very start of the conversation, with no message of your own first.
- Never ask the user to upload documents yourself, list document requirements, or duplicate document_agent's upload workflow — document_agent owns that conversation entirely, including eligibility checks and plan lookup.
- Never call payment_agent before the user has explicitly named one specific plan from the eligible list AND typed "confirm" in Phase 3. Both steps are required, in that order.
- Never call payment_agent if document_agent returned a decline.
- Never present only one eligible tier if document_agent returned more than one — always show the full list.
- Always pass plan details to payment_agent as a labeled string in the exact format "Plan: [plan name]" followed by "Monthly Rental: [monthly rental amount] AED" on the next line — for example "Plan: Smart 150 / Monthly Rental: 150 AED". Never pass them as a single unlabeled comma-separated value, a sentence, or any other format.
- Once payment_agent responds, output its response unchanged as your message to the user — no summarizing, rephrasing, or regenerating any part of it, including the link. Treat payment_agent's output as an opaque block to copy, never as text to read and retype from understanding. A reproduced URL that is shorter than, different from, or only resembles the original is always a failure, with no exceptions.
- Never narrate or list data being passed between agents.
- Never make up or assume any information not provided by the user or returned by an agent.
- Never perform document extraction, eligibility checks, or payment creation yourself — always delegate to the relevant specialist agent.
- Always be polite and professional throughout.
```

---

### 3.5 Add Sub-Agents

Click the **Toolset** tab on the right side menu → scroll down to the **Agents** section → **Add agents** → **Local instance** → select:

- `document_agent`
- `payment_agent`

Click **Add**.

---

## Part 4 — Full Pipeline Test

On the Master Agent page, click the **refresh button** on the top left of the agent chat panel on the right side → click the quick start prompt:

```
Check my postpaid plan eligibility
```

> Using training documents: `EID_Train.png` + `Payslip_Train.png`

**Expected conversation flow:**

```
Agent : [Immediately calls document_agent without greeting]

        [Upload prompt: Emirates ID]
User  : EID_Train.png

        [Upload prompt: Payslip]
User  : Payslip_Train.png

Agent : [Document extraction and eligibility check happens]
        [Review document extraction → Submitted]

Agent : [Presents eligible plans in a table]
        
        | Plan | Monthly Rental (AED) | Credit Limit (AED) |
        |---|---|---|
        | Smart 55 | 55 | 500 |
        | Smart 150 | 150 | 1200 |
        
        Based on your eligibility, Smart 150 offers the most data and the highest credit limit for your budget.
        
        Which plan would you like to proceed with?

User  : Smart 150

Agent : Here is the plan you've selected:
        - Plan: Smart 150
        - Monthly Rental: 150 AED
        - Credit Limit: 1200 AED
        
        Kindly type "Confirm" to proceed with payment.

User  : confirm

Agent : Thank you, please proceed with the payment using the following link:
        [Click here to pay for Smart 150](https://checkout.stripe.com/c/pay/cs_test_...)
```

---

## Part 5 — Test Scenarios

When the Master Agent asks you to upload your documents, upload the relevant Emirates ID and payslip for each scenario below.

---

### Scenario 1 — Test PASS (Eligible for multiple tiers) ✅

Upload when prompted:

| | |
|---|---|
| Emirates ID | [EID_Pass.png](documents/Pass%20Test/EID_Pass.png) |
| Payslip | [Payslip_Pass.png](documents/Pass%20Test/Payslip_Pass.png) |
| **Expected** | **PASS — Multiple eligible tiers presented** |

The system should:
1. Extract data successfully
2. Pass all eligibility checks (name match, ID validity, salary ≥ 4000)
3. Present eligible plan tiers based on salary
4. Allow user to select a plan
5. Generate Stripe payment link

---

### Scenario 2 — Test FAIL (Eligibility rejection) ❌

Upload when prompted:

| | |
|---|---|
| Emirates ID | [EID_Fail.png](documents/Fail%20Test/EID_Fail.png) |
| Payslip | [Payslip_Fail.png](documents/Fail%20Test/Payslip_Fail.png) |
| **Expected** | **FAIL — Rejection with reason** |

The system should:
1. Extract data successfully
2. Fail one or more eligibility checks (name mismatch, expired ID, or salary < 4000)
3. Present clear rejection reason
4. NOT proceed to plan selection or payment

---

### Scenario 3 — Test Payment Flow (Complete end-to-end) 💳

Use the PASS scenario documents and complete the full flow:

1. Upload documents
2. Review eligible plans
3. Select a plan (e.g., "Smart 150")
4. Confirm selection
5. Receive Stripe payment link
6. Click the link to open Stripe Checkout
7. Use test card: `4242 4242 4242 4242`, any future expiry, any CVC
8. Complete test payment

**Expected:** Payment succeeds and redirects to success page.

---

## 🎉 Congratulations!

You have successfully built and tested a fully functional 3-agent Etisalat Postpaid Plan Eligibility pipeline on IBM watsonx Orchestrate — powered by document extraction, eligibility validation, knowledge base plan lookup, and Stripe payment integration.

### What You Built

- **document_agent**: Agentic workflow with Emirates ID and payslip extraction, eligibility validation script, and knowledge base integration
- **payment_agent**: ADK Python tool with Stripe test mode integration
- **Master agent**: React-style orchestrator managing the full user journey
- **Knowledge base**: CSV-based plan catalog with salary-based eligibility

### Key Concepts Covered

- Multi-agent orchestration with React and Default styles
- Agentic workflows with document extraction nodes
- Python script nodes for business logic
- Knowledge base integration for dynamic data retrieval
- ADK tool development with external API integration (Stripe)
- Agent collaboration patterns
- Test mode payment processing
