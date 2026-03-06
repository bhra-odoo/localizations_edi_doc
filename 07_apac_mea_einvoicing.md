# 07. APAC & MEA E-Invoicing: APIs and Peppol Frameworks

The Asia-Pacific (APAC) and Middle East & Africa (MEA) regions are defined by heavy adoption of JSON wrappers around APIs and, particularly in Oceania, an embrace of the standard Peppol framework.

## 1. India (GST / NIC / Cleartax)
India's mandate applies to virtually all B2B transactions. Instead of pure XML, India uses a JSON payload sent to an Invoice Registration Portal (IRP), typically utilizing intermediate APIs like Cleartax to manage the throughput.
* **Modules:** `l10n_in_edi`
* **Configuration:**
  * **Master Configuration:** Go to *Settings > Companies*. Your **GSTIN** is paramount. Also, ensure the complete address (especially State Code and Pincode) is flawlessly formatted, as the API validates addressing strictly.
  * **API Credentials:** In *Accounting Settings > Indian Electronic Invoicing*, enter the Client ID, Client Secret, and Token provided by Cleartax.
  * **HSN/SAC Codes:** Path: *Sales > Products*. Every product must have an exact **HSN (Harmonized System Nomenclature)** or **SAC (Services Accounting Code)**. Without a valid 6 or 8 digit HSN snippet, the JSON will trigger a structural rejection by the NIC.
* **The IRN Flow:**
  1. The user clicks "Send & Print". Odoo compiles a JSON data map of the invoice.
  2. Odoo POSTs the JSON to the Cleartax/NIC API endpoint.
  3. The IRP receives the JSON, validates the internal calculations, and assigns an **IRN (Invoice Reference Number)** (a 64-character hash).
  4. The IRP structurally signs the JSON and returns a highly specific JSON Web Token (JWT) representing a **QR Code**.
  5. Odoo unpacks the response, stores the IRN, and renders the signed QR code on the PDF.

## 2. Saudi Arabia (ZATCA - Fatoorah)
Saudi Arabia's system is rolling out in waves and is hyper-complex due to strict cryptographic enforcement built directly into the local Odoo instance.
* **Modules:** `l10n_sa_edi`
* **Configuration & CSID Onboarding:**
  * Odoo cannot simply connect with an API key. You must generate Cryptographic Stamps.
  * **Step 1 (OTP):** You log into the ZATCA Fatoorah portal and generate an OTP (One-Time Password).
  * **Step 2 (Compliance CSID):** Inside Odoo (*Settings > Accounting > ZATCA*), enter the OTP. Odoo sends a CSR (Certificate Signing Request) to ZATCA to receive a temporary Compliance CSID.
  * **Step 3 (Production CSID):** Odoo runs test XMLs against ZATCA. If successful, ZATCA converts the temporary CSID into a permanent Production CSID. This is stored securely in the database.
* **The API Flow (Clearance Mode - B2B/B2G):**
  1. Odoo generates a heavily localized UBL 2.1 XML.
  2. Odoo generates an incredibly complex SHA-256 hash of the XML. Crucially, **the hash of Invoice N must include the hash of Invoice N-1** (creating a blockchain-like sequence).
  3. Odoo signs the XML with the local Production CSID.
  4. Odoo sends the XML to the ZATCA API.
  5. ZATCA validates and returns a status. Odoo embeds the specific ZATCA QR code.

## 3. Egypt (ETA - Egyptian Tax Authority)
Egypt mandates a robust JSON-based system, relying on either physical hardware tokens or cloud-based sealing services.
* **Modules:** `l10n_eg_edi`
* **Configuration:**
  * **Company:** Must have the Company Registry matching the ETA profile.
  * **API Credentials:** Enter the ETA Client ID and Client Secret in Accounting Settings (supports both Pre-Production and Production URLs).
  * **Product Coding:** Path: *Products*. Every product must be mapped to the **Global Standards 1 (GS1)** database or the **Egyptian Goods and Services (EGS)** registry.
* **The ETA Flow:**
  1. Odoo structures a massive JSON payload dictating the exact tax computations.
  2. **Signing:** By Egyptian law, production invoices must be digitally signed locally. Odoo integrates with a middleware component (often a local proxy running on a Windows machine attached to a physical USB e-Seal token) to digitally sign the JSON string before it leaves the local network. 
  3. The signed JSON is pushed via API to the ETA. Odoo polls to receive valid UUIDs.

## 4. Australia & New Zealand (Peppol)
Australia and NZ have avoided proprietary government APIs in favor of mandating the international Peppol standard for B2G, while heavily incentivizing it for B2B.
* **Modules:** `l10n_au`, `l10n_nz`, `account_peppol`
* **Configuration:**
  * Follows the exact same configuration flow as European Peppol.
  * **EAS & Endpoint:** 
    * Australia uses EAS `0151` linked to the **ABN (Australian Business Number)**.
    * New Zealand uses EAS `0088` or `0190` linked to the **NZBN (New Zealand Business Number)**.
* **Workflow:** Leverages Odoo's IAP Access point to generate Peppol BIS Billing 3.0 UBL XML and transmit it globally without requiring complex local signing certificates.
