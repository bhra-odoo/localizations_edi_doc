# Test Environments & Credentials Configuration

Demonstrating or developing e-invoicing requires test environments so you don't inadvertently send false fiscal data to government agencies.

Odoo handles testing through two primary mechanisms: "Test Mode" booleans and specific Demo Data.

## 1. Global EDI Test Mode
For many countries, you must enable a "Test Environment" toggle before sending documents.
* **How to enable:**
  1. Go to **Accounting > Configuration > Settings**.
  2. Scroll down or search for the specific country's EDI section (e.g., "Mexican Electronic Invoicing", "Colombian EDI").
  3. Look for a checkbox saying **"Test Environment"** or **"Demo Mode"** and check it off.
* **Effect:** This forces the `account.move.send` wizard to route API calls to the government's/provider's sandbox URLs instead of production URLs.

## 2. Using Odoo's IAP Test Mode (Peppol, Italy SDI)
For services managed by Odoo's IAP (like Peppol or the Italian SDI), the database must typically be in test mode to use test credentials.
* **Developer Mode:** Turn on debug mode (`?debug=1`).
* **Settings:** Go to **Settings > Technical > System Parameters**.
* Add or verify the key `iap.endpoint` or checking `odoo.conf` parameters. Note: Odoo standard SaaS databases have a built-in test mock for IAP on staging branches.

### Peppol-Specific Demo Mode
Odoo features a dedicated sandbox specifically for Peppol to allow fake transmissions.
* **Configuration:** In **Accounting > Configuration > Settings**, under Peppol, check the **"Demo Mode"** box (often requires developer mode).
* **Registration:** Registers your company on a mock IAP server without SMS or business verification.
* **Customer Credentials:** Set the test customer's Peppol EAS to `9999` and Endpoint ID to `TEST`. The mock server auto-validates this.
* **Loopback Testing for Sessions:** By sending a Peppol invoice to a vendor configured with your own registered Demo Endpoint ID, you can demonstrate Odoo's automated Vendor Bill creation when the Peppol fetch cron job runs.

## 3. Country-Specific Test Credentials

If setting up a local or staging database for a session, you can use these standard test methodologies:

### Mexico (CFDI)
When you install `l10n_mx_edi` with Demo Data checked upon database creation, Odoo automatically injects Finkok Test Credentials.
* **Provider:** Finkok
* **Username:** Found automatically loaded in settings if Demo Data is on (e.g., generic test emails provided by Finkok).
* *Manual entry:* Use standard Finkok test RFCs and CSD files available from SAT's public test developer site.

### Colombia (DIAN)
You need a specific "Software Test Set ID" (TestSetId) from the DIAN Muisca portal.
* **TestSetID:** You must generate this yourself on the DIAN website using a valid company profile; Odoo cannot provide universal test credentials for this because DIAN tracks test submissions against the specific company NIT.
* **Certificate:** You can use a self-signed generic `.p12` test certificate for structural validation if you just want to see the XML build, but DIAN will reject it without a real test signature.

### Italy (SDI)
* Just enable "Test Environment" in settings. You can use any valid layout of an Italian VAT.
* Odoo's Italian IAP proxy automatically accepts test invoices and returns a simulated "Accepted" receipt if the test environment is on.

### India (Cleartax/NIC)
* Odoo integrates usually via Cleartax.
* **Test Credentials:** You can sign up for a free Cleartax Sandbox account online to get a Client ID and Secret, which you enter in Odoo's Accounting Settings.

## 4. Best Practices for Sessions
When giving a training session:
1. **Prepare the DB:** Create a DB *with Demo Data* + install the target localization.
2. **Review Settings:** Point out the "Test Mode" checkboxes to the audience. This proves the system separates sandbox from prod.
3. **Show the XML, not just the API success:** If a test credential fails (which happens often due to temp Sandbox outages from governments), you can still demonstrate Odoo's power by successfully generating the XML. 
4. Explain: *"The XML structured perfectly. The final step is just transmission, which in Test Mode routes to a sandbox API."*
5. Download the XML from the attachments and open it in VS Code to explain the tags.
