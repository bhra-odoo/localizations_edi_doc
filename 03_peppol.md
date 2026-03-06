# 03. Peppol Architecture & Configuration in Odoo

To understand Peppol in Odoo, you must understand the difference between the **Format** and the **Network**.
* **The Format:** A standardized XML structure (usually Peppol BIS Billing 3.0, a subset of UBL 2.1).
* **The Network:** The secure, closed transport infrastructure (the 4-corner model) used to transmit that XML.

Odoo handles *both*: generating the compliant XML and natively transmitting it over the network via its IAP (In-App Purchase) Access Point service.

---

## The 4-Corner Model: Technical Flow in Odoo

1. **Corner 1 (Sender - Your Odoo DB):** You confirm an invoice and hit "Send via Peppol". Odoo generates the Peppol BIS 3.0 XML `ir.attachment`.
2. **Corner 2 (Sender's Access Point - Odoo IAP):** Odoo sends the XML payload securely via RPC to Odoo's Peppol Access Point servers (`iap-services.odoo.com`).
3. **Corner 3 (Receiver's Access Point):** The Odoo AP uses the Peppol SMP (Service Metadata Publisher - basically a DNS for Peppol) to look up the Receiver's Endpoint ID (e.g., `0208:0403448140`). The SMP returns the URL of the Receiver's Access Point. The Odoo AP securely transmits the XML via AS4 protocol to Corner 3.
4. **Corner 4 (Receiver - Customer's Accounting System):** The customer's software pulls the XML from their Access Point and imports it.

---

## Deep Dive: Configuration & Setup

### 1. Company Registration (Becoming Corner 1)
Path: **Settings > Accounting > Peppol**
* **Peppol EAS (Endpoint Accounting Scheme):** A 4-digit code denoting the *type* of identifier. Examples:
    * `0208` = Belgian Enterprise Number
    * `0184` = Danish CVR number
    * `9901` = Danish UBL (Often used for general EU VATs)
    * `0151` = Australian ABN
* **Peppol Endpoint ID:** The actual ID (e.g., `BE0403448140`).
* **Registration Process:** When you click "Register", Odoo pings its IAP server. To prevent spam/fraud on the Peppol network, Odoo requires verification (usually an SMS code sent to the company's registered phone number) before it officially publishes your Endpoint to the Peppol Directory.

### 2. Customer Configuration (Targeting Corner 4)
Path: **Contacts > (Select Customer) > Accounting Tab**
* **Peppol Details:** You must specify the EAS and Endpoint ID for the customer.
* **Format Preference:** Ensure their preferred electronic invoice format is set to `Peppol BIS Billing 3.0`.
* **The `peppol_endpoint_valid` Field:** Odoo runs a background check against the Peppol Directory (using the `account_peppol` module's API calls) to see if the customer's Endpoint ID actually exists. A green checkmark indicates safety to send.

---

## Sending and Receiving Workflows

### Sending an Invoice
1. Create and confirm an `account.move`.
2. Click **Send & Print** (`account.move.send`).
3. Check **Send via Peppol**. (This checkbox is only available if both you and the customer are validly configured).
4. Odoo queues the message (`account.edi.document` state moves to `to_send`).
5. A cron job (or immediate RPC call depending on Odoo version) dispatches the payload to the IAP.
6. The invoice chatter logs the AS4 Message ID for tracking.

### Receiving a Vendor Bill (Automated Import)
Because your Odoo DB is registered as an Endpoint on the network, vendors can send XMLs to you.
1. **The Cron Job:** `account_peppol.ir_cron_peppol_get_new_documents` runs (usually daily/hourly).
2. It asks the Odoo IAP AP: "Are there any pending messages for my Endpoint ID?"
3. The AP sends down the incoming Peppol XML files.
4. Odoo intercepts these, parses the UBL data, and automatically creates a Draft `account.move` of type `in_invoice` (Vendor Bill).
5. **Smart Matching:** Odoo uses the supplier's `<cac:AccountingSupplierParty>` ID to map to an existing Contact, finds PO numbers in the XML `<cac:OrderReference>` to auto-fill purchase order lines, and links the original XML attachment.

---

## Key Technical Tables & Models
If you need to debug or build views around Peppol during your session, reference these models:
* `res.partner`: Houses the `peppol_eas` and `peppol_endpoint`.
* `account.edi.document`: Tracks the state (`to_send`, `sent`, `error`) of specific EDI transmissions attached to a move.
* `peppol.registration`: (Internal/IAP side) manages the state of your company's registration on the network.
