# 05. European E-Invoicing: Deep Technical Dive

Europe is heavily driven by the VAT in the Digital Age (ViDA) initiative. While Peppol establishes a standard network, many countries still maintain complex, proprietary B2G/B2B clearance models that Odoo supports natively.

## 1. Italy (SDI - Sistema di Interscambio)
Italy mandates B2B, B2C, and B2G e-invoicing via the SDI platform using the proprietary **FatturaPA** XML format.
* **Module:** `l10n_it_edi`
* **Configuration:**
  * **Company setup:** Go to *Settings > Companies*. You must configure the **Partita IVA** (VAT) and **Codice Fiscale** (Tax ID). Also define the **Regime Fiscale** (Fiscal Regime) under Accounting Settings.
  * **Customer Setup:** You must configure the target destination. This means entering a **Codice Destinatario** (Recipient Code, exactly 7 characters) or a **PEC** (Certified Email Address) on the contact's Accounting tab. If dealing with an international customer outside the SDI network, use `XXXXXXX` (seven Xs) to signal an export.
  * **Tax Configuration (`account.tax`):** This is the most complex part of Italian EDI. Standard rates (e.g., 22%) are straightforward. However, for 0% rates (exemptions, exports, reverse charges), you must assign a **Natura** code (e.g., `N1`, `N2.1`, `N3.2`). Without the correct Natura code mapped to the 0% tax, the SDI will reject the invoice immediately with a schema error.
  * **Document Types:** Invoice (`TD01`), Down Payment (`TD02`), Credit Note (`TD04`), Self-Invoice (`TD16-TD19`). Odoo automatically computes the TD code based on the move type and taxes applied.
* **Workflow & IAP Endpoint:** 
  * Odoo generates the `FatturaPA` XML.
  * It transmits via RPC to Odoo's IAP server. 
  * The IAP operates securely as an SDI intermediary. The SDI validates the XML (verifying the VAT exists, Natura codes match) and forwards it to the Customer's *Codice Destinatario*.
  * The SDI asynchronously returns a Receipt (Accettata/Scartata). A daily cron job (`ir.cron` "Italian EDI: Check invoice status") polls the IAP to update the `<field name="l10n_it_edi_state">` on the invoice.

## 2. Spain (Facturae & TicketBAI)
Spain manages a dual system: a national standard (Facturae) and deeply localized provincial systems (TicketBAI).
* **Modules:** `l10n_es_edi_facturae` (National), `l10n_es_edi_tbai` (Basque Country).
* **Facturae (National B2G):**
  * **Format:** Custom XML format `Facturae 3.2.2`.
  * **Configuration:** Sending to public entities requires exact routing information. On the Customer Contact form, you must configure three distinct **DIR3** codes:
    * *Oficina Contable* (Accounting Office)
    * *Órgano Gestor* (Managing Body)
    * *Unidad Tramitadora* (Processing Unit)
  * Without these three tags populated in the XML, the FACe (government portal) will reject the invoice.
* **TicketBAI (Basque Country B2B/B2C):**
  * Affects companies registered in Álava, Bizkaia, and Gipuzkoa.
  * **Configuration:** Operates as an immediate supply of information. You must upload a digital `.p12` or `.pfx` certificate to the `res.company` record. 
  * **Technical Flow:** The moment the invoice is posted, Odoo generates the XML -> Signs the XML locally using the uploaded certificate -> Synchronously fires an API request to the respective provincial council -> Expects a successful 200 OK response with a signature. Odoo then prints a specific QR code on the PDF report linking back to the TicketBAI registry.

## 3. France (Chorus Pro & Factur-X)
France mandates B2G via Chorus Pro and is rolling out B2B via the hybrid Factur-X standard.
* **Modules:** `l10n_fr_edi_facturx` (often relies on the base `account_edi_ubl_cii`).
* **Configuration:** 
  * **Company/Customer:** Requires accurate **SIREN** (9 digits) or **SIRET** (14 digits) numbers in the Company Registry field.
  * **Chorus Pro (B2G):** Requires defining a **Code Service Exécutant** (Routing Code) on the customer contact to route the invoice to the exact department within the French administration.
* **Output:** Odoo uses Factur-X (based on the CII standard, identical payload structure to ZUGFeRD). Odoo generates a PDF and embeds a `factur-x.xml` file within the PDF/A-3 envelope.

## 4. Germany (ZUGFeRD & XRechnung)
Germany maintains strict structural requirements based on the CIUS (Core Invoice Usage Specifications).
* **Modules:** `l10n_de`
* **XRechnung (Pure XML for B2G):** 
  * **Configuration:** To send to the German government via Peppol or direct portal, you must have a **Leitweg-ID** configured on the customer contact. This acts as the routing mechanism indicating the exact government agency.
* **ZUGFeRD (Hybrid PDF/XML for B2B):**
  * Default behavior in Odoo. Very similar to Factur-X.
  * Odoo automatically generates standard UBL/CII taxonomies for tax types and unit mapping.

## 5. Poland (KSeF - Krajowy System e-Faktur)
Poland utilizes a strict, real-time centralized clearance model (similar in architecture to LatAm).
* **Modules:** `l10n_pl_edi`
* **Configuration:**
  * To connect to the KSeF Ministry of Finance API, you must generate an **API Token** directly from the government portal and paste it into the *Accounting Settings > KSeF* section.
* **Workflow:** 
  * Odoo generates the Polish FA(2) XML format.
  * Connecting to KSeF requires a two-step handshake using the token to establish an encrypted session.
  * Odoo uploads the XML. KSeF performs synchronous structural validation. 
  * If valid, KSeF immediately returns a unique **KSeF Reference Number**, which Odoo logs on the invoice form.

## 6. Belgium (Peppol & UBL)
Belgium relies almost exclusively on the Peppol network, heavily pushing B2B and B2G over standard BIS Billing 3.0.
* **Modules:** `l10n_be`, `account_peppol`.
* **Configuration:** 
  * Primary identifier is the CBE (Crossroads Bank for Enterprises) number. Look in *Settings > Companies*. The Company Registry field should hold the CBE (e.g., `0403.448.140`).
  * In Peppol settings, select EAS `0208`. The endpoint is the CBE number without formatting.
  * Taxes rely heavily on Tax Grids mapping to standard UI categories (Exempt, IC-Delivery) which map to UBL `TaxCategory` codes (`S`, `Z`, `E`, `K`).
