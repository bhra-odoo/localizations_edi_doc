# 06. Latin American E-Invoicing: Clearance Models

Latin American e-invoicing is defined by the **Clearance Model**: an invoice has zero legal validity until the tax authority (or their authorized proxy) signs it or returns an authorization code. 

Odoo handles this via complex XML signing libraries, specific document types, and asynchronous API polling via backend cron jobs.

## 1. Colombia (DIAN)
All electronic invoices, credit notes, and waybills must be cleared by the *Dirección de Impuestos y Aduanas Nacionales (DIAN)*.
* **Modules:** `l10n_co_dian`
* **Configuration:**
  * **Digital Certificate:** Go to *Settings > Companies*. Upload a `.p12` or `.pfx` certificate and enter the password.
  * **Software Data:** From the DIAN Muisca portal, you must obtain a **Software PIN**, **Software ID**, and **Test Set ID** (if in test mode) and enter them in Odoo's Accounting Settings.
  * **Document Types:** You cannot use standard Odoo invoice journals. You must configure the journal to use the `l10n_latam_document_type` specifically mapped to DIAN codes (e.g., *Factura Electrónica de Venta*).
* **The CUFE Algorithm & API Flow:**
  1. Odoo generates a UBL 2.1 XML containing highly specific Colombian extensions.
  2. Odoo calculates the **CUFE** (Código Único de Facturación Electrónica). This is a complex SHA-384 hash using the invoice number, date, amounts, taxes, the company's NIT, the customer's NIT, and an internal DIAN key.
  3. Odoo uses the `.p12` certificate to digitally sign the XML (XAdES standard).
  4. Odoo sends the signed XML symmetrically to the DIAN API.
  5. The DIAN returns a `.ZIP` file containing an `ApplicationResponse` XML. If the response says "Accepted", the invoice is valid. Odoo then generates the QR code containing the CUFE and prints the PDF.

## 2. Mexico (SAT)
Mexico uses the **CFDI 4.0** (Comprobante Fiscal Digital por Internet) standard. Rather than sending directly to the SAT, companies send XML to a PAC (Proveedor Autorizado de Certificación), an authorized third party.
* **Modules:** `l10n_mx_edi`
* **Configuration:**
  * **RFC & CSD:** The company must have an **RFC** (Tax ID) and upload a **CSD** (Certificado de Sello Digital). The CSD consists of a `.cer` key, a `.key` file, and a password.
  * **PAC Credentials:** In Accounting Settings, enter the username/password for your PAC (e.g., Finkok, Solución Factible).
  * **The Three Catalogs:** Mexico relies on rigid SAT catalogs.
    * *Product Master:* Every product requires a SAT Product Code (e.g., `01010101` for generic).
    * *UoM:* Every unit requires a SAT Unit Code (e.g., `H87` for piece).
    * *Customer Data:* The contact requires a specific **Regimen Fiscal** (Fiscal Regime) and **Uso CFDI** (CFDI Usage, e.g., `G03` for General Expenses).
* **The Timbre Flow:**
  1. Odoo generates the CFDI XML.
  2. Odoo signs the "Cadena Original" (original string) of the XML using the local CSD.
  3. Odoo connects via SOAP to the PAC WebService. 
  4. The PAC validates the XML against SAT rules. If valid, the PAC injects a **Timbre Fiscal Digital** (Digital Stamp) back into the XML, containing a massive **UUID** (e.g., `85B4...-....`).
  5. Odoo saves the UUID and generates a QR code mapped to the SAT validation URL.

## 3. Peru (SUNAT / OSE)
Peru validates XML directly via SUNAT or via third-party OSEs.
* **Modules:** `l10n_pe_edi`
* **Configuration:**
  * **Company/Customer:** Requires an active **RUC** (Registro Único de Contribuyentes).
  * **SOL Credentials:** In Accounting Settings, enter the **User SOL** and **Password** provided by SUNAT to establish the SOAP connection.
  * **Certificate:** A digital certificate `.p12` must be uploaded to sign the XML.
  * **Catalogs:** Ensure taxes use the correct SUNAT catalogs (e.g., `IGV` / `ISC`).
* **The CDR Flow:**
  1. Odoo generates a UBL 2.1 XML.
  2. Odoo signs the XML locally. 
  3. Odoo sends the XML to the SUNAT/OSE endpoint.
  4. SUNAT returns a **CDR (Constancia de Recepción)**. This is an XML receipt proving the invoice was accepted. Odoo attaches this CDR to the invoice chatter.

## 4. Argentina (AFIP)
Argentina uses AFIP's Web Services (WSFE for domestic, WSFEX for export).
* **Modules:** `l10n_ar`, `l10n_ar_edi`
* **Configuration:**
  * **AFIP Certificate:** Upload the AFIP-issued connection certificate.
  * **Point of Sale (Punto de Venta):** Odoo Journals must be configured strictly as AFIP Points of Sale, mapped to exact API endpoints (e.g., Web Services).
  * **Responsibility Types:** The AFIP Responsibility Type (e.g., *Responsable Inscripto*) must be set on the company and the customer to determine which Document Type (Factura A vs Factura B) Odoo will generate.
* **The CAE Flow:**
  1. User confirms the invoice.
  2. Odoo opens a connection to the WSFE API, requesting authorization for the specific invoice amounts and taxes.
  3. AFIP returns a **CAE (Código de Autorización Electrónico)** and a CAE Expiration Date.
  4. Odoo saves the CAE to `<field name="l10n_ar_afip_auth_code">` and prints it as a barcode on the layout.

## 5. Chile (SII)
Chile uses standard XML DTEs (Documentos Tributarios Electrónicos) sent to the SII.
* **Modules:** `l10n_cl_edi`
* **Configuration:**
  * **CAF (Código de Autorización de Folios):** Unlike other localizations that use standard Odoo sequences, Chile requires you to upload an XML file (the CAF) provided by the SII directly into the Invoice Journal. The CAF contains the pre-approved sequence ranges and cryptographic keys to sign the DTE.
  * **Digital Certificate:** Requires a personal digital certificate representative of the company.
* **Flow:** Odoo generates the XML -> Signs with CAF and personal certificate -> Submits to SII -> SII provides a track ID -> Odoo runs a cron job polling the track ID until the DTE is accepted.
