# 02. EDI and e-Invoicing: Deep Dive into Odoo's Architecture

Electronic Data Interchange (EDI) in Odoo is not just a simple export script. It is a robust, extensible framework designed to handle the complexity of hundreds of different national formats, asynchronous API communication, and hybrid document generation.

## The `account.edi.format` Framework (The Standard Engine)

Odoo's EDI framework revolves around the `account.edi.format` model. Every specific EDI type (e.g., UBL 2.1, Factur-X, Peppol BIS 3) is a record in this model.

When you create an invoice (`account.move`), Odoo computes the applicable EDI formats based on the company's localization, the customer's country, and specific settings.

### 1. Generating the XML
Instead of hardcoding XML strings, Odoo uses **QWeb templates** to render the XML data.
* **Data Preparation:** Python methods like `_export_invoice_vals` gather all the data needed (taxes grouped by rate, lines grouped by product, company identifiers).
* **Rendering:** A QWeb template (e.g., `account_edi_ubl_cii.ubl_21_Invoice`) uses this data dictionary to render the actual `<cbc:Invoice>` XML structure. This makes it highly customizable using standard Odoo view inheritance (`<xpath>`).

### 2. The `account.move.send` Wizard (v17+)

In recent Odoo versions, the sending process was unified into the `account.move.send` wizard. This is the popup you see when you click "Send & Print".

**How it works technically:**
1. **Compute Available Formats:** The wizard checks `res.partner` settings (what format does the customer prefer?) and `res.company` settings (what is legally required?).
2. **Checkbox Toggles:** It presents checkboxes to the user (e.g., "Send via Peppol", "Generate Factur-X").
3. **Execution (`action_send`):** 
    * It calls `_generate_and_send_invoices()`.
    * This triggers the specific `_export_as_xml` or API methods of the selected EDI formats.
    * It updates the chatter (`mail.message`) and attaches the resulting files (`ir.attachment`).

---

## EDI Formats: A Technical Distinction

### 1. Pure XML Formats (e.g., UBL BIS 3.0, XRechnung)
* The output is a raw `.xml` file.
* Typically sent via an API (like SDI in Italy or KSeF in Poland) or over a network (Peppol).
* In Odoo, these are often governed by the `account_edi_ubl_cii` module which provides a base class for UBL and CII generation that specific localizations extend.

### 2. Hybrid Formats (Factur-X / ZUGFeRD)
* Factur-X is a standard that creates a **PDF/A-3** document.
* PDF/A-3 allows for file attachments *inside* the PDF payload itself.
* Odoo generates the Factur-X XML (based on CII) -> Takes standard Odoo PDF report -> Uses a Python library (like `PyPDF2` or `pikepdf`) to embed the `factur-x.xml` byte stream directly into the PDF metadata.
* **Why this is powerful:** A human opens the PDF and reads it normally. Software opens the PDF, extracts the embedded XML, and imports the data automatically.

---

## The Asynchronous Clearance Model (Webservices)

Many government APIs (DIAN in Colombia, AFIP in Argentina, SDI in Italy) do not respond instantly.

1. Odoo sends the XML to the government API.
2. The government returns a "Transaction ID" or "Ticket Number", not a final approval.
3. Odoo marks the EDI state on the invoice as **To Check** (`to_check`) or **Sent** (`sent`), but not definitively accepted.
4. **The Cron Job:** An automated action (`ir.cron`) runs periodically in the background (e.g., "Account EDI: Update Document Status"). It pings the government API with the Ticket Number to ask "Is the invoice approved yet?"
5. Once the government API returns "Accepted" (along with a digital signature or unique code like a CUFE or CAE), Odoo finalizes the invoice state and generates the final PDF displaying the official government signature/QR code.

---

## Technical Troubleshooting Flow for Session Demos

When demonstrating, XML generation errors are common due to missing master data. Here is how to trace them:

1. **User clicks "Send & Print", error pops up:** *"Missing Customer VAT string"*.
    * **Cause:** The QWeb template hit an assertion or `_export_invoice_vals` found a mandatory field empty.
    * **Fix:** Go to the Customer Contact, fill in the VAT, and retry.
2. **Invoice sent, but EDI state is "Error":**
    * **Cause:** The XML was generated successfully, but the *Transmission* failed (either the API rejected it, or network timeout).
    * **Fix:** Check the invoice chatter. Odoo logs the exact API response (e.g., *"Error 400: Zip code does not match state code format"*). Fix the data on the contact/company and click "Retry" on the EDI banner atop the invoice.
