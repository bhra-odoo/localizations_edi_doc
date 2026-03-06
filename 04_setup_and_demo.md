# 04. Setup and Demo Flow: A Systematized Guide

When conducting a live session on Odoo Accounting, you must follow a strict sequence. Missing master data is the #1 cause of failing live demos when generating EDI.

Follow this exact script to ensure a flawless demonstration.

## Preparation Phase (Pre-Session)

1. **Database Setup:**
   * Create a new database with **Demo Data Enabled**.
   * Install the Accounting App (`account_accountant` in Enterprise).
   * Install the specific localization you want to demo (e.g., `l10n_be` for Belgium).
   * Note: The `account_edi` and `account_peppol` modules are usually auto-installed by European localizations.

2. **Verify Test Environment (CRITICAL):**
   * Path: **Settings > Technical > System Parameters** (Requires Developer Mode `?debug=1`).
   * Ensure `iap.endpoint` is pointing to the test server (if applicable) or that you are using a standard Odoo SaaS staging environment to avoid registering fake data on live networks.

---

## Live Demo Sequence

### Step 1: The Core Foundation (Company Setup)
*Narrative: "Before we can generate a legal e-invoice, Odoo needs to know exactly who we are. E-invoicing APIs reject anonymous or incomplete senders."*
1. Go to **Settings > Companies > Update Info**.
2. Show the critical fields: **Legal Name**, **Full Address** (emphasize ZIP and State).
3. Point to the **VAT Number**. Explain that this is the primary key used by governments and Peppol to identify the sender.
4. Go to **Accounting > Configuration > Settings**.
5. Scroll to the **Taxes** section. Show how taxes have "Tax Grids" or specific groupings. Explain: *"These grids aren't just for Odoo's reports; they map directly to UBL/XML tags so the government knows exactly how much VAT applies to which category."*

### Step 2: The Network Identity (Peppol Setup - If Applicable)
*Narrative: "Now that Odoo knows our legal data, we need to register our 'digital mailbox' on the Peppol network."*
1. Stay in **Accounting > Configuration > Settings**.
2. Scroll to the **Peppol** section.
3. Show the **EAS Code** dropdown (e.g., `0208` for Belgium CBE).
4. Show the **Endpoint ID** field (auto-populated from the company Registry/VAT).
5. Click **Verify / Register** (If in a safe test environment, show the SMS verification flow. If live, just explain the security mechanism).

### Step 3: The Target (Customer Setup)
*Narrative: "We send invoices differently depending on who the customer is. A government entity requires different XML tags than a private business."*
1. Go to **Customers > Contacts**. Create a new "Test Corp".
2. Enter a **Country** and a **VAT Number**.
3. Navigate to the **Accounting Tab**.
4. Show the **Electronic Invoicing Format** setting. Explain how Odoo can auto-detect this, but you can force specific formats depending on the customer's software capabilities.
5. If showing Peppol, show the Peppol EAS and Endpoint fields here. Explain the immediate background check Odoo performs to validate the ID against the Peppol Directory.

### Step 4: The Goods (Product Master Data)
*Narrative: "Products must be standardized for international trade."*
1. Go to **Sales > Products**. Open a product.
2. Under the General/Accounting tabs, highlight the **Income Account** and **Customer Taxes**.
3. Emphasize the **Unit of Measure**. Explain that "Pieces" or "Hours" are translated by Odoo into UNECE standard codes (`H87`, `HUR`) inside the XML payload.

### Step 5: Execution (Generating the E-Invoice)
*Narrative: "We have our foundation. Let's see Odoo's EDI engine in action."*
1. Go to **Accounting > Customers > Invoices**. Create a New Invoice for "Test Corp".
2. Add the configured product.
3. Click **Confirm**. (Explain that modifying the invoice is locked down once confirmed, which is a legal requirement in many EDI clearance models).
4. Click **Send & Print**.
5. The `account.move.send` wizard opens. Expand the options.
6. Check **Generate Factur-X** and/or **Send via Peppol** (depending on your setup).
7. Click **Send**.

### Step 6: Proof & Technical Verification (The "Wow" Moment)
*Narrative: "To the user, they just clicked send. But under the hood, Odoo just structured a complex data payload."*
1. Close the wizard and point to the **Chatter** (right side or bottom of the screen).
2. Show the communication log indicating the EDI was generated or transmitted.
3. Point to the **Attachments (Paperclip icon)**.
4. Click and download the `.xml` file (e.g., `factur-x.xml` or `INV_202X_001.xml`).
5. **Open the XML in a text editor (VS Code/Notepad) and show it to the audience on screen.**
6. *Crucial Demonstration Step:* Scroll through the XML and actively point out:
   * `<cbc:ID>`: "Here is our invoice number."
   * `<cac:AccountingSupplierParty>`: "Here is the company data we configured in Step 1."
   * `<cac:TaxCategory>`: "Here are the tax rates and UBL codes Odoo calculated automatically based on our product and fiscal position."

### Step 7: The Receiving Flow (Reverse Demo)
*Narrative: "E-invoicing goes both ways. What happens when a vendor sends us an XML?"*
1. Go to **Accounting > Vendors > Bills**.
2. Click **Upload**.
3. Upload a standard UBL/Factur-X XML file (have a test file ready on your desktop).
4. Watch as Odoo instantly parses the XML, identifies the Vendor, creates the lines, sets the amounts, and prepares a Draft Bill without any manual data entry.
