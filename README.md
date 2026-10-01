# ZATCA Phase-2 E-Invoicing API Integration Specification
### Developer Integration Guide for Batch ERP

---

## 1. API Endpoint & Authentication

```http
POST /api/method/your_custom_app.api.create_zatca_invoice
Host: <your-site>.batcherp.net
Content-Type: application/json
Authorization: token <api_key>:<api_secret>
```

> **Note on Batch ERP Endpoint:** Your Batch ERP backend handles internal accounting ledgers, cost centers, payment means, item creation, and tax templates automatically. The external system only transmits the business data detailed below.

---

## 2. Concrete JSON Payload Examples

### Example A: Standard Tax Invoice (B2B)
> **Use Case:** Sale to a corporate client or registered business. Full buyer identity, Commercial Registration (`CRN`), VAT number, and verified Saudi address are **mandatory**.

```json
{
  "is_b2c": false,
  "date": "2026-10-01",
  "due_date": "2026-10-01",
  "currency": "SAR",
  "customer": {
    "name": "Saudi Business Solutions Co.",
    "name_ar": "شركة حلول الأعمال السعودية",
    "vat_number": "310123456700003",
    "id_type": "CRN",
    "id_number": "1010987654",
    "address": {
      "street": "King Fahd Road",
      "district": "Al Olaya",
      "building_no": "3456",
      "postal_code": "12214",
      "city": "Riyadh",
      "country": "SA"
    }
  },
  "items": [
    {
      "name": "IT Infrastructure Consulting",
      "qty": 2,
      "unit_price": 1500.00,
      "uom": "Unit",
      "discount": 0.0,
      "vat_rate": 15
    },
    {
      "name": "Network Firewall Gateway",
      "qty": 1,
      "unit_price": 4500.00,
      "uom": "Nos",
      "discount": 450.00,
      "vat_rate": 15
    }
  ],
  "notes": "Contract reference: PO-2026-089"
}
```

---

### Example B: Simplified Tax Invoice (B2C) — Anonymous / Retail Consumer
> **Use Case:** Walk-in cash customer, fast retail, or standard eCommerce order. **Address, VAT number, and personal ID are omitted.**

```json
{
  "is_b2c": true,
  "date": "2026-10-01",
  "due_date": "2026-10-01",
  "currency": "SAR",
  "customer": {
    "name": "Cash Customer",
    "name_ar": "عميل نقدي"
  },
  "items": [
    {
      "name": "Wireless Optical Mouse",
      "qty": 1,
      "unit_price": 65.00,
      "uom": "Nos",
      "discount": 5.00,
      "vat_rate": 15
    }
  ]
}
```

---

## 3. Complete Field Specification Table

### 3.1 Header Level Attributes

| Field | Type | Required in B2B? | Required in B2C? | Format / Allowed Values | Description & Validation Rule |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `is_b2c` | Boolean | **Yes** (`false`) | **Yes** (`true`) | `true` \| `false` | Distinguishes between Standard (B2B Clearance) and Simplified (B2C Reporting). Maps to Batch ERP `custom_b2c`. |
| `date` | String | **Yes** | **Yes** | `YYYY-MM-DD` (e.g. `"2026-10-01"`) | Invoice issue date (`cbc:IssueDate`). Must be the transaction date. |
| `due_date` | String | **Yes** | **Yes** | `YYYY-MM-DD` | Due date or actual date of supply (`cbc:ActualDeliveryDate`). For cash sales, equal to `date`. |
| `currency` | String | No | No | `ISO 4217` (e.g. `"SAR"`, `"USD"`) | Default: `"SAR"`. For foreign currencies, VAT is converted to SAR per ZATCA regulation. |
| `notes` | String | No | No | Freeform text | Optional document notes or purchase order references. |

---

### 3.2 Customer Object (`customer`)

| Field | Type | Required in B2B? | Required in B2C? | Format / Allowed Values | Description & Validation Rule |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `customer.name` | String | **Yes** | **Yes** | Legal Entity Name | Buyer business name (B2B) or individual name / `"Cash Customer"` (B2C). Maps to `cbc:RegistrationName`. |
| `customer.name_ar` | String | Recommended | Recommended | Arabic Name string | Buyer name in Arabic. Recommended for bilingual invoices in Saudi Arabia. |
| `customer.vat_number` | String | **Yes** | **Do Not Include** | Exactly 15 digits (`^3[0-9]{13}3$`) | Buyer VAT registration number (`cbc:CompanyID`). **Must start with `3` and end with `3`**. Individual consumers do not have VAT numbers. |
| `customer.id_type` | String | **Yes** | Optional | See [Section 4.1](#41-buyer-id-scheme-codes-id_type) for codes | Scheme identifier for the buyer's official registration. For B2B, use `"CRN"` or `"700"`. For B2C, use `"NAT"`, `"IQA"`, or `"PAS"`. |
| `customer.id_number` | String | **Yes** | Optional | Identifier String (e.g. `"1010987654"`) | Buyer registration number. **Mandatory in B2B** (Batch ERP enforces CR number for corporate buyers). Optional in B2C. |
| `customer.address` | Object | **Yes** | **Do Not Include** | Address Object | National Address details. **Strictly required for B2B; omitted for B2C.** |

---

### 3.3 Customer Address Object (`customer.address`)
> **Important:** This entire object is **mandatory for B2B** and **omitted for B2C**.

| Field | Type | Required in B2B? | Format / Constraints | Description & ZATCA Mapping |
| :--- | :--- | :--- | :--- | :--- |
| `address.street` | String | **Yes** | Street name (e.g. `"King Fahd Road"`) | Street Name (`cbc:StreetName`). Cannot be blank. |
| `address.district` | String | **Yes** | District / Quarter (e.g. `"Al Olaya"`) | City Subdivision Name (`cbc:CitySubdivisionName`). Cannot be blank. |
| `address.building_no` | String | **Yes** | **Exactly 4 digits** (`^[0-9]{4}$`, e.g. `"3456"`) | Building Number (`cbc:BuildingNumber`). Strictly enforced. |
| `address.postal_code` | String | **Yes** | **Exactly 5 digits** (`^[0-9]{5}$`, e.g. `"12214"`) | Postal Zone (`cbc:PostalZone`). Strictly enforced. |
| `address.city` | String | **Yes** | City name (e.g. `"Riyadh"`, `"Jeddah"`) | City Name (`cbc:CityName`). Cannot be blank. |
| `address.country` | String | No | ISO 3166-1 alpha-2 (Default: `"SA"`) | Country Identification Code (`cac:Country/cbc:IdentificationCode`). |

---

### 3.4 Line Items Array (`items`)

| Field | Type | Required? | Format / Allowed Values | Description & Calculation Rule |
| :--- | :--- | :--- | :--- | :--- |
| `items[].name` | String | **Yes** | Product or service name | Description of supplied items (`cac:Item/cbc:Name`). |
| `items[].qty` | Number | **Yes** | Float / Integer > 0 (e.g. `2`) | Invoiced quantity (`cbc:InvoicedQuantity`). |
| `items[].unit_price` | Number | **Yes** | Non-negative Float (e.g. `1500.00`) | **Net Unit Price excluding VAT** (`cac:Price/cbc:PriceAmount`). |
| `items[].uom` | String | No | e.g. `"Nos"`, `"Unit"`, `"PCE"`, `"KGM"` | Unit of Measure (`unitCode`). Defaults to `"Unit"`. |
| `items[].discount` | Number | No | Float >= 0 (Default: `0.0`) | Line item discount amount (deducted from taxable amount prior to VAT). |
| `items[].vat_rate` | Number | No | Float (`15`, `0`) | Applicable VAT percentage. Default is `15` (Standard KSA rate). |

---

## 4. Code Lists & Reference Enums

### 4.1 Buyer ID Scheme Codes (`id_type`)

| Code | Scheme (English) | Scheme (Arabic) | Target Entity | Expected Format |
| :--- | :--- | :--- | :--- | :--- |
| `CRN` | Commercial Registration | السجل التجاري | **B2B Companies** | 10 digits (Standard for private companies) |
| `700` | 700 Unified Number | الرقم الموحد 700 | **B2B Ministries & Semi-Gov** | 10 digits starting with `700` |
| `TIN` | Tax Identification Number | الرقم الضريبي | **B2B Non-CR Entities** | 15 digits |
| `NAT` | National ID | الهوية الوطنية | **B2C Saudi Citizens** | 10 digits starting with `1` |
| `IQA` | Iqama Number | هوية مقيم (الإقامة) | **B2C Resident Expats** | 10 digits starting with `2` |
| `PAS` | Passport | جواز السفر | **B2C Foreign Visitors** | Alphanumeric |

---

## 5. Critical ZATCA Business Rules & Validations

The Batch ERP integration backend automatically verifies the following rules before signing and sending the invoice to ZATCA:

1. **Buyer VAT Number Validation:**
   * In B2B transactions, if the customer is VAT registered, the `vat_number` must be exactly 15 numerical digits, starting with `3` and ending with `3`.
2. **Saudi National Address Constraint:**
   * In B2B transactions, the `building_no` must be **exactly 4 digits** and the `postal_code` must be **exactly 5 digits**. Alphanumeric or mismatched lengths cause automatic clearance rejection.
3. **B2B Commercial Registration Rule:**
   * Batch ERP enforces that every B2B customer record must include a valid Commercial Registration (`id_number` under `CRN` or `700`).
4. **B2C Address Exemption:**
   * B2C invoices must **never** contain mandatory address validations. The XML structure generated will omit `<cac:PostalAddress>` to comply with ZATCA Simplified invoice specifications.
5. **Net Unit Price Requirement:**
   * `unit_price` must always represent the price **before VAT**. The VAT amount is computed as:
     ```text
     VAT Amount = (qty * unit_price - discount) * (vat_rate / 100)
     ```

---

## 6. Sample API Response

### Success Response (HTTP 200 / 201)

```json
{
  "success": true,
  "invoice_number": "ACC-SINV-2026-00142"
}
```

### Error Response (HTTP 400 / 422)

```json
{
  "success": false,
  "message": "Building Number must be exactly 4 digits in customer address."
}
```
