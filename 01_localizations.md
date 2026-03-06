# 01. Odoo Fiscal Localizations: In-Depth Architecture

Fiscal localizations are the foundational layers required before any e-invoicing can happen. An e-invoice is essentially a structured representation of an accounting move; if the underlying accounting data (taxes, accounts) is incorrect for that country, the XML will fail government validation.

## Technical Structure of a Localization

A localization module (e.g., `l10n_in` for India) typically contains the following core data structures located in `data/` or generated via Python `_get_...` methods in `models/template_...`:

### 1. Chart of Accounts (CoA)
* **Model:** `account.account.template` (Legacy/Data) or `account.chart.template`
* **Purpose:** Defines the legal ledger accounts.
* **Why it matters for EDI:** While the specific debit/credit account numbers aren't usually in an e-invoice, the *type* of account (e.g., `asset_receivable`, `liability_payable`) dictates how Odoo treats the invoice lines internally.

### 2. Taxes (`account.tax`)
* **Purpose:** The most critical component for e-invoicing. Every country calculates VAT/GST differently.
* **Fields critical for EDI:**
  * `amount`: The percentage (e.g., 20%).
  * `tax_group_id`: Groups similar taxes together (e.g., "State GST 9%" and "Central GST 9%" grouped under "GST 18%"). EDI frameworks sum taxes by Tax Group.
  * **Tags (Tax Grids):** (`account.account.tag`). These are used to populate the Tax Report (e.g., Box 81 on the Belgian VAT return). In many EDI formats, these tags or specific Tax Groups are mapped directly to UBL Category Codes (e.g., Standard rate `S`, Zero rated `Z`, Exempt `E`).

### 3. Fiscal Positions (`account.fiscal.position`)
* **Purpose:** Rules that swap default accounts or taxes on an invoice based on the customer's country, state, or specific Tax ID.
* **Example:** Selling from France to Germany to a B2B customer. The Fiscal Position swaps the domestic 20% French VAT to an "Intra-Community Exempt" tax.
* **EDI Impact:** The resulting "Exempt" tax will carry a specific Reason Code (e.g., `VATEX-EU-IC`) which the UBL framework injects into the `<cac:TaxCategory><cbc:TaxExemptionReasonCode>` node. Without the fiscal position mapping the correct tax, the invoice XML is structurally invalid for cross-border trade.

### 4. Document Types (`l10n_latam.document.type`)
* **Purpose (Primarily LatAm):** Countries like Argentina, Chile, and Peru have strictly defined statutory document types (e.g., "Factura A", "Nota de Crédito B").
* **EDI Impact:** Instead of Odoo's generic `out_invoice` / `out_refund`, the XML generation specifically looks at `move.l10n_latam_document_type_id.code` to populate the UBL `<cbc:InvoiceTypeCode>`.

---

## Configuration Checklist for a Localization Session

When setting up a database for a demonstration, you must systematically configure the following records. Miss one, and the resulting EDI will likely fail validation.

### Step 1: Company Legal Data
Path: **Settings > Users & Companies > Companies**
* **Name:** Must exactly match the registered legal name.
* **Address:** Street, City, Zip, State, Country. *Crucial:* Many EDI APIs reject invoices if the State or Zip code doesn't match the national registry postal standard.
* **VAT / Tax ID:** The most important field. Odoo uses this to identify your company as the "Sender" in the XML.
* **Company Registry:** (e.g., Siret in France, KBO in Belgium, CIN in India). Often mapped to the `CompanyID` node in UBL.

### Step 2: Customer Legal Data
Path: **Contacts > (Select Customer)**
* **Country:** Determines the Fiscal Position applies.
* **VAT:** Determines B2B vs B2C rules. In Europe, Odoo can validate this against the VIES database.
* **Accounting Tab:** Look for country-specific fields added by the localization (e.g., "RUC" in Peru, "O.C." in Spain).

### Step 3: Product Master Data
Path: **Sales/Accounting > Products > Products**
* **UNSPSC / Commodity Codes:** Many EDI formats require products to be classified using standard international codes. In Odoo, this is often the `unspsc_code_id` or `l10n_in_hsn_code`.
* **UoM (Unit of Measure):** UNECE recommendation 20 defines standard codes for units (e.g., `H87` for Piece, `KGM` for Kilogram). Odoo's Unit of Measure categories map to these standard UNECE codes in the background for UBL generation.
