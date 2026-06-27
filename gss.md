# GoSweetSpot (GSS) Freight API Documentation

Source: https://api-docs.gosweetspot.com/

This document compiles the GoSweetSpot Freight API documentation including authentication, endpoints, models, webhooks, and shipping options.

---

## Table of Contents

1. [Introduction](#introduction)
2. [Authentication](#authentication)
3. [Rate Limiting](#rate-limiting)
4. [Sandbox Account](#sandbox-account)
5. [Data Types and Formats](#data-types-and-formats)
6. [Concepts](#concepts)
7. [Common Use Cases](#common-use-cases)
8. [Tracing Your Calls](#tracing-your-calls)
9. [FAQ](#faq)
10. [Endpoints](#endpoints)
    - [Available Services — GET /api/availableservices](#available-services--get-apiavailableservices)
    - [Customer Orders — GET /api/customerorders](#customer-orders--get-apicustomerorders)
    - [Customer Orders — PUT /api/customerorders](#customer-orders--put-apicustomerorders)
    - [Customer Orders — DELETE /api/customerorders](#customer-orders--delete-apicustomerorders)
    - [Order Status — GET /v2/order](#order-status--get-v2order)
    - [Pending Orders — GET /v2/pendingorders](#pending-orders--get-v2pendingorders)
    - [Rates — POST /api/rates](#rates--post-apirates)
    - [Shipments — GET /api/shipments](#shipments--get-apishipments)
    - [Shipments — POST /api/shipments](#shipments--post-apishipments)
    - [Shipments — DELETE /api/shipments](#shipments--delete-apishipments)
    - [Shipment Status — POST /v2/shipmentstatus](#shipment-status--post-v2shipmentstatus)
    - [Address Validation — POST /v2/addressvalidation](#address-validation--post-v2addressvalidation)
    - [Publish Manifest — POST /v2/publishmanifest](#publish-manifest--post-v2publishmanifest)
    - [Printers — GET /api/printers](#printers--get-apiprinters)
    - [Labels — GET /api/labels](#labels--get-apilabels)
    - [Labels — POST /api/labels](#labels--post-apilabels)
    - [Labels Enqueue — POST /api/labels/enqueue](#labels-enqueue--post-apilabelsenqueue)
    - [Pickup Booking — POST /api/bookpickup](#pickup-booking--post-apibookpickup)
    - [Stock Sizes — GET /api/stocksizes](#stock-sizes--get-apistocksizes)
11. [Webhooks](#webhooks)
12. [Shipping Options](#shipping-options)
13. [Models](#models)

---

## Introduction

The GoSweetSpot (GSS) Freight API is a JSON-based REST API for managing shipping orders, rates, shipments, labels, pickups, manifesting, address validation, and tracking across multiple carriers.

**Base URL:** `https://api.gosweetspot.com`

---

## Authentication

The API requires authenticated requests with specific headers:

- **`access_key`** — Your unique API key from GoSweetSpot.
- **`site_id`** — The site you're requesting action for.

**How to obtain credentials:**
1. Sign into GoSweetSpot at https://ship.gosweetspot.com
2. Navigate to **Administration > Integrations & Apps**
3. Locate the API integration and generate a new key

For accounts managing multiple sites, the API link must be added to each site separately.

---

## Rate Limiting

Currently no rate-limiting enforcement is active. However, GSS recommends staying below:

- **60 calls per minute**, or
- **1000 calls per hour**

Exceeding these thresholds may trigger usage review.

---

## Sandbox Account

A testing environment is available at:

```
https://ship.gosweetspot.com/opensandbox
```

---

## Data Types and Formats

- The API exclusively uses **JSON** for all requests and responses.
- When timezone information isn't explicitly specified, **New Zealand timezone** is applied by default.

---

## Concepts

Two distinct entities form the foundation of the API:

- **Customer Orders** — correspond one-to-one with e-commerce orders.
- **Shipments** — represent one or more packages destined for the same location via identical carriers.

A single order may generate multiple shipments based on item characteristics (weight, dimensions, nature). E-commerce integrations typically prioritize creating customer orders rather than shipments directly.

---

## Common Use Cases

**Case 1 — Custom E-Commerce Platform**
- Orders publish to GSS via `PUT /api/customerorders` when dispatch-ready.
- Operators process ticketing through the GSS web interface.
- Status updates retrieved periodically via `GET /api/customerorders`.

**Case 2 — Specialized Dispatch Workflow**
- `POST /api/rates` retrieves available freight options.
- Dispatchers review selections.
- `POST /api/shipments` generates shipments directly.

**Case 3 — Open Source Platforms**
- GSS integrates with popular open-source systems where business justified.

---

## Tracing Your Calls

Recent API activity viewing is available at:

```
https://ship.gosweetspot.com/developer/apitrace
```

---

## FAQ

**Connection:** Requires `access_key` header (Preferences & Settings). Multi-site accounts need `site_id` header. Standard HTTP methods used.

**Return Formats:** JSON is the primary format. XML compatibility exists but is not actively tested.

**Authentication Requirements:** Contact account manager for test access keys. Every request needs `access_key` and `support_email` headers. Multi-site setups require `site_id`.

**Rate Limits:** No current enforcement; reserves discretionary blocking rights. Recommended limit 60 calls per minute.

**Backwards Compatibility:** System prioritizes compatibility, but evolution may necessitate client updates.

---

# Endpoints

## Available Services — GET /api/availableservices

Retrieve shipping carriers and services associated with an API access key.

**URL:** `https://api.gosweetspot.com/api/availableservices`
**Method:** `GET`
**Content-Type:** `application/json`

### Headers
| Header | Description |
|---|---|
| `access_key` | Unique API key from GSS |
| `site_id` | Site identifier |

### Response Fields
| Field | Type | Description |
|---|---|---|
| `Carriers` | array | Array of carrier objects |
| `Carriers[].CarrierId` | integer | Unique carrier identifier |
| `Carriers[].CarrierName` | string | Display name of carrier |
| `Carriers[].CarrierType` | string | Carrier classification type |
| `Carriers[].Services` | string[] | Available shipping services |
| `Carriers[].AccountNumber` | string | Associated account identifier |

### cURL
```bash
curl --location 'https://api.gosweetspot.com/api/availableservices' \
  --header 'access_key;' \
  --header 'site_id;'
```

---

## Customer Orders — GET /api/customerorders

Retrieve published customer orders.

**URL:** `https://api.gosweetspot.com/api/customerorders`
**Method:** `GET`
**Content-Type:** `application/json`

### Headers
| Header | Description |
|---|---|
| `access_key` | Unique API key |
| `site_id` | Site identifier |

### Query Parameters
| Parameter | Type | Description |
|---|---|---|
| `packingslipno` | string | Comma-separated packing slip numbers for filtering |
| `createdfrom` | string | UTC date/time, earliest results |
| `createdto` | string | UTC date/time, latest results |
| `excludecompleted` | string | "true"/"false" — excludes completed slips when true |
| `page` | integer | Page index |
| `pagesize` | integer | Records per page (default 100) |
| `includeProducts` | boolean | Include product lines when true |

### Response Structure
| Field | Type | Description |
|---|---|---|
| `Page` | integer | Current page index |
| `PageSize` | integer | Records per page |
| `Pages` | integer | Total pages available |
| `Results` | array | List of Customer Order objects |

Each order in `Results` follows the [Customer Orders Model](#customer-orders-model).

### cURL
```bash
curl --location 'https://api.gosweetspot.com/api/customerorders?createdfrom=2023-08-01&page=2' \
  --header 'access_key;' \
  --header 'site_id;' \
  --header 'Content-Type: application/json; charset=utf-8'
```

---

## Customer Orders — PUT /api/customerorders

Saves new orders or updates a pending order. Already-processed orders are ignored. Supports batch submission.

**URL:** `https://api.gosweetspot.com/api/customerorders`
**Method:** `PUT`
**Content-Type:** `application/json`

### Headers
| Header | Description |
|---|---|
| `access_key` | Unique API key |
| `site_id` | Site identifier |

### Request Body
Accepts a JSON array of order objects.

| Parameter | Type | Notes |
|---|---|---|
| `packingslipno` | string | Unique order identifier |
| `consignee` | string | Recipient name |
| `address1` | string | Address line 1 (street number) |
| `address2` | string | Address line 2 (street name) |
| `suburb` | string | Suburb |
| `city` | string | City or state (abbreviation if applicable) |
| `postcode` | string | Postal code |
| `country` | string | ISO Alpha 2 (NZ, AU, US, etc.) |
| `delvref` | string | Customer reference number |
| `delvinstructions` | string | Label instructions |
| `contactname` | string | Contact name (optional) |
| `contactphone` | string | Contact phone (optional) |
| `email` | string | Recipient email (optional) |
| `products` | array | See [Product Model](#product-model) |
| `packages` | array | See [Customer Order Package Model](#customer-orders-package-model) |
| `dangerousgoods` | string[] | Array of Dangerous Goods Preset names |
| `customField1Value` | string | Custom field 1 |
| `customField2Value` | string | Custom field 2 |
| `shippingDescription` | string | Shipping reference (max 250 chars) |
| `isSignatureRequired` | boolean | Signature vs. Authority to Leave |

### Response
JSON array, one entry per submitted order:

```json
[
  { "packingslipno": "test1-14-07-01", "result": true, "msg": "" },
  { "packingslipno": "test2-14-07-01", "result": false, "msg": "already processed" }
]
```

---

## Customer Orders — DELETE /api/customerorders

Removes unprocessed customer orders.

**URL:** `https://api.gosweetspot.com/api/customerorders`
**Method:** `DELETE`
**Content-Type:** `application/json`

### Headers
| Header | Description |
|---|---|
| `access_key` | Unique API key |
| `site_id` | Site identifier |

### Query Parameters
| Parameter | Type | Description |
|---|---|---|
| `packingslipno` | string | Comma-separated packing slip numbers (max 50 per call) |

### Response
JSON object mapping order numbers to outcome messages:

- `"Invalid packing slip number."`
- `"Cannot be deleted. Already deleted."`
- `"Cannot be deleted. Already processed."`
- `"Max delete limit is 50 packing slip numbers."`
- `"Deleted."`

#### Example Response
```json
{
  "test#1234": "Cannot be deleted. Already processed.",
  "test#1235": "Cannot be deleted. Already deleted.",
  "test#1236": "Deleted."
}
```

---

## Order Status — GET /v2/order

Get the current status of a single published order.

**URL:** `https://api.gosweetspot.com/v2/order`
**Method:** `GET`
**Content-Type:** `application/json`

### Headers
| Header | Description |
|---|---|
| `access_key` | Unique API key |
| `site_id` | Site identifier |

### Query Parameters
| Parameter | Type | Description |
|---|---|---|
| `packingslipno` | string | Unique order/packing slip number from source system |

### Response Fields
| Field | Type | Description |
|---|---|---|
| `packingslipno` | string | Unique order/packing slip number |
| `consignee` | string | Recipient name |
| `status` | string | `PENDING`, `PICKED`, or `DELIVERED` |
| `ticketnumber` | string | Consignment number |
| `trackingurl` | string | Track and trace URL |
| `picked` | datetime | Pickup timestamp (local to origin) |
| `delivered` | datetime | Delivery timestamp (local to destination) |
| `ticketprinteddate` | datetime | When ticket was created |
| `ticketprintedby` | string | Username of ticket creator |
| `ticketdeleteddate` | datetime | When ticket was deleted (nullable) |
| `ticketdeletedby` | string | Username of deleter (nullable) |

#### Example Response
```json
{
  "packingslipno": "SSORDER5840",
  "consignee": "PHOTO DREAMS LIMITED",
  "status": "DELIVERED",
  "ticketnumber": "SSPOT012671",
  "trackingurl": "http://gosweetspot.com/track/1234-SSPOT012671",
  "picked": "2014-06-03T08:40:00",
  "delivered": "2014-06-03T14:46:45",
  "ticketprinteddate": "2014-06-03T08:30:00",
  "ticketprintedby": "[email protected]",
  "ticketdeleteddate": null,
  "ticketdeletedby": null
}
```

---

## Pending Orders — GET /v2/pendingorders

Retrieve a list of orders that have been published but not yet processed.

**URL:** `https://api.gosweetspot.com/v2/pendingorders`
**Method:** `GET`
**Content-Type:** `application/json`

### Headers
| Header | Description |
|---|---|
| `access_key` | Unique API key |
| `site_id` | Site identifier |

### Response
Array of pending customer order objects (similar shape to [Customer Orders Model](#customer-orders-model)).

### cURL
```bash
curl --location 'https://api.gosweetspot.com/v2/pendingorders' \
  --header 'access_key;' \
  --header 'site_id;' \
  --header 'Content-Type: application/json'
```

---

## Rates — POST /api/rates

Retrieve available shipping rates for a destination.

**URL:** `https://api.gosweetspot.com/api/rates`
**Method:** `POST`
**Content-Type:** `application/json`

### Headers
| Header | Description |
|---|---|
| `access_key` | Unique API key |
| `site_id` | Site identifier |

### Request Body
| Parameter | Type | Description |
|---|---|---|
| `origin` | object | Optional [Contact Model](#contact-model); blank = use site address |
| `destination` | object | Recipient ([Contact Model](#contact-model)) |
| `packages` | array | List of [Rate Package Model](#rate-package-model) |
| `issignaturerequired` | boolean | Signature requirement |
| `issaturdaydelivery` | boolean | Saturday delivery option |
| `deliveryreference` | string | Reference (max 60 chars) |
| `DeclarationType` | string | Optional: `merchandise`, `documents`, `returned`, `sample` |

### Response Structure
| Attribute | Type | Description |
|---|---|---|
| `Available` | array | [Available Rate Model](#available-rate-model) |
| `Rejected` | array | [Reject Rate Model](#reject-rate-model) |
| `ValidationErrors` | array | [Rate Validation Error Model](#rate-validation-error-model) |

### Request Example
```bash
curl --location 'https://api.gosweetspot.com/api/rates' \
  --header 'Content-Type: application/json' \
  --header 'access_key: YOUR_ACCESS_KEY' \
  --header 'site_id: 108633' \
  --data-raw '{
    "DeliveryReference": "ORDER123",
    "Destination": {
      "Name": "DestinationName",
      "Address": {
        "BuildingName": "",
        "StreetAddress": "DestinationStreetAddress",
        "Suburb": "Avonside",
        "City": "Christchurch",
        "PostCode": "8061",
        "CountryCode": "NZ"
      },
      "ContactPerson": "DestinationContact",
      "PhoneNumber": "123456789",
      "Email": "[email protected]",
      "DeliveryInstructions": "Destinationdeliveryinstructions"
    },
    "IsSaturdayDelivery": false,
    "IsSignatureRequired": true,
    "Packages": [
      {
        "Height": 10,
        "Length": 10,
        "Width": 10,
        "Kg": 1,
        "Name": "GSS-DLE SATCHEL",
        "PackageCode": "DLE",
        "Type": "Box",
        "HasDG": false
      }
    ]
  }'
```

### Response Example
```json
{
  "Available": [
    {
      "QuoteId": "3104eb7e-6354-4de4-a250-fa96297282d2",
      "CarrierId": 102,
      "CarrierName": "Post Haste",
      "DeliveryType": "Overnight",
      "Cost": 8.58,
      "Charge": 10.58,
      "ServiceStandard": "By 11am next business day",
      "Comments": "Satchel",
      "Route": "AKL- LOCAL->AKL- SI",
      "IsRuralDelivery": false,
      "IsSaturdayDelivery": false,
      "IsFreightForward": false,
      "CarrierServiceType": "DomesticCourier"
    }
  ],
  "Rejected": [
    {
      "Reason": "MS-AKL -> CHRISTCHURCH: Consignment undersize/weight",
      "Carrier": "Mainstream AKL",
      "DeliveryType": "Standard 1000kg+"
    }
  ],
  "ValidationErrors": {}
}
```

---

## Shipments — GET /api/shipments

Retrieve shipment status updates (historical shipments).

**URL:** `https://api.gosweetspot.com/api/shipments`
**Method:** `GET`
**Content-Type:** `application/json`

### Headers
| Header | Description |
|---|---|
| `access_key` | Unique API key |
| `site_id` | Site identifier |

### Query Parameters
| Parameter | Type | Description |
|---|---|---|
| `shipments` | string | Comma-separated consignment numbers |
| `ordernumbers` | string | Comma-separated order numbers |
| `lastupdateminutc` | string | UTC earliest update date |
| `lastupdatemaxutc` | string | UTC latest update date |
| `includeDangerousGoodsFlag` | boolean | Populate `IsDangerousGoods` in response |
| `excludeDeletedShipments` | boolean | Exclude deleted shipments |
| `page` | integer | Defaults to 1 |
| `pagesize` | integer | Max 250, defaults to 250 |

> When `shipments` or `ordernumbers` are provided, date filters are ignored.

### Response Structure
| Field | Type | Description |
|---|---|---|
| `Page` | integer | Current page |
| `Pages` | integer | Total pages |
| `PageSize` | integer | Records per page |
| `Results` | array | List of [Shipment Model](#shipment-model) objects |

### cURL
```bash
curl --location 'https://api.gosweetspot.com/api/shipments?lastupdateminutc=2015-08-13%2008:15:13&page=1&shipments=ABC0001,ABC0002,ABC20303' \
  --header 'access_key;' \
  --header 'site_id;' \
  --header 'Content-Type: application/json'
```

---

## Shipments — POST /api/shipments

Create a new shipment. You can either use a `QuoteId` (from `POST /api/rates`) or specify `Carrier` + `Service` directly.

**URL:** `https://api.gosweetspot.com/api/shipments`
**Method:** `POST`
**Content-Type:** `application/json`

### Headers
| Header | Description |
|---|---|
| `access_key` | Unique API key |
| `site_id` | Site identifier |

### Request Body — Core Fields
| Parameter | Type | Description |
|---|---|---|
| `QuoteId` | string | Identifier from `/api/rates`; exclusive with `Carrier`/`Service` |
| `Carrier` | string | Courier name or `*` for cheapest |
| `Service` | string | Carrier service (optional, defaults to cheapest) |
| `Origin` | object | [Contact Model](#contact-model). Leave null for site default |
| `Destination` | object | [Contact Model](#contact-model) |
| `Packages` | array | [Rate Package Model](#rate-package-model) |

### Delivery Options
| Parameter | Type | Description |
|---|---|---|
| `IsSaturdayDelivery` | boolean | Saturday delivery |
| `IsSignatureRequired` | boolean | Signature on delivery |
| `DutiesAndTaxesByReceiver` | boolean | Receiver pays customs duties |
| `DeliveryReference` | string | Order reference (max 50 chars) |
| `DeliveryInstructions` | string | Special handling notes |

### Customs & Insurance
| Parameter | Type | Description |
|---|---|---|
| `Commodities` | array | [Commodity Model](#commodity-model) (international) |
| `RecipientTaxId` | string | Tax ID for certain destinations |
| `IncludeInsurance` | boolean | Additional insurance coverage |
| `DeclarationType` | string | `merchandise`, `documents`, `returned`, or `sample` |

### Output & Printing
| Parameter | Type | Description |
|---|---|---|
| `PrintToPrinter` | string | "true"/"false" or printer name |
| `Outputs` | array | Output formats ([Print Output Formats](#print-output-formats)) |
| `CustomField1Value` | string | Custom field 1 |
| `CustomField2Value` | string | Custom field 2 |

### Hazardous Materials
| Parameter | Type | Description |
|---|---|---|
| `DangerousGoods` | object | [Dangerous Goods Model](#dangerous-goods-model) |

### Response Structure
```json
{
  "CarrierId": 0,
  "CarrierName": "string",
  "IsFreightForward": true,
  "Message": "string",
  "Errors": ["string"],
  "SiteId": 0,
  "Consignments": [
    {
      "Connote": "string",
      "TrackingUrl": "string",
      "Cost": 0,
      "Charge": 0,
      "ConsignmentId": 0,
      "OutputFiles": {},
      "IsSaturdayDelivery": false,
      "IsRural": false
    }
  ]
}
```

### Key Response Fields
- **`Connote`** — Tracking number assigned by carrier
- **`TrackingUrl`** — Link to track shipment
- **`Cost`** — Freight cost
- **`Charge`** — Billable amount
- **`OutputFiles`** — Generated label/document data (base64 encoded)
- **`Message`** — Operation status indication

### Notes
- When `QuoteId` is supplied, `Carrier` and `Service` are ignored.
- Printing defaults to enabled if a printer is configured; disable via `PrintToPrinter="false"`.
- Custom fields require site configuration.

---

## Shipments — DELETE /api/shipments

Removes unprocessed consignments.

**URL:** `https://api.gosweetspot.com/api/shipments`
**Method:** `DELETE`
**Content-Type:** `application/json`

### Headers
| Header | Description |
|---|---|
| `access_key` | Unique API key |
| `site_id` | Site identifier |

### Query Parameters
| Parameter | Type | Description |
|---|---|---|
| `id` | string | Comma-separated consignment numbers (max 50 per call) |

### Response
JSON object mapping consignment numbers to status. Possible statuses:

- `Deleted`
- `Invalid ticket number`
- `Cannot be deleted. Already deleted.`
- `Cannot be deleted. Already delivered.`
- `Cannot be deleted. Already in transit.`
- `Cannot be deleted. Already manifested.`

**HTTP 409** if max consignment limit 50 exceeded.

#### Example
```bash
curl --location --request DELETE 'https://api.gosweetspot.com/api/shipments?id=SSPOT014115,SSPOT014114,SSPOT014113,SSPOT014112' \
  --header 'access_key;' \
  --header 'site_id;' \
  --header 'Content-Type: application/json'
```

```json
{
  "SSPOT014112": "Deleted",
  "SSPOT014113": "Deleted",
  "SSPOT014114": "Cannot be deleted. Already deleted.",
  "SSPOT014115": "Cannot be deleted. Already deleted."
}
```

---

## Shipment Status — POST /v2/shipmentstatus

Returns the current status of one or more shipments, including line-by-line events.

**URL:** `https://api.gosweetspot.com/v2/shipmentstatus`
**Method:** `POST`
**Content-Type:** `application/json`

### Headers
| Header | Description |
|---|---|
| `access_key` | Unique API key |
| `site_id` | Site identifier |

### Request Body
A JSON array of consignment numbers:

```json
[
  "WRR1406658",
  "WRR1406659"
]
```

### Response Fields
| Field | Type | Description |
|---|---|---|
| `ConsignmentNo` | string | Queried shipment identifier |
| `Status` | string | Current shipment status |
| `Picked` | datetime (nullable) | Pickup date/time |
| `Delivered` | datetime (nullable) | Delivery date/time |
| `Tracking` | string | Tracking URL |
| `Events` | array | [V2 Shipment Status Event Model](#v2-shipment-status-event-model) |
| `TotalCost` | number | Shipment cost |
| `CreatedUtc` | datetime | Created timestamp |

---

## Address Validation — POST /v2/addressvalidation

Validates customer address details. For NZ addresses, also assesses rural, Saturday, and residential delivery eligibility.

**URL:** `https://api.gosweetspot.com/v2/addressvalidation`
**Method:** `POST`
**Content-Type:** `application/json`

### Headers
| Header | Description |
|---|---|
| `access_key` | Unique API key |
| `site_id` | Site identifier |

### Request Body
| Attribute | Type | Description |
|---|---|---|
| `consignee` | string | Company/person name (max 50) |
| `contactperson` | string | Optional contact (max 50) |
| `phonenumber` | string | Optional phone (max 50) |
| `deliveryinstructions` | string | Optional instructions (max 120) |
| `email` | string | Optional email (max 200) |
| `address` | object | [V2 Validation Address Model](#v2-validation-address-model) |

### Response
| Attribute | Type | Description |
|---|---|---|
| `validated` | boolean | State/suburb/postcode/country validation result |
| `address` | object | [V2 Address Validation Response Model](#v2-address-validation-response-model) |
| `errors` | string[] | Validation messages |

---

## Publish Manifest — POST /v2/publishmanifest

Submits pending consignments to their carriers for manifesting and returns the summary manifest report as a PDF.

**URL:** `https://api.gosweetspot.com/v2/publishmanifest`
**Method:** `POST`
**Content-Type:** `application/json`

### Headers
| Header | Description |
|---|---|
| `access_key` | Unique API key |
| `site_id` | Site identifier |

### Request Body
JSON array of consignment numbers:

```json
[
  "4WD0002186",
  "4WD0002187",
  "4WD0002188"
]
```

### Response
JSON array of base64-encoded PDF binary manifests.

```json
[
  "iVBORw0KGgoAAAANSUhEUgAABUYAAAMHCAYAAAD1lY2SAAAAAXNS...",
  "iVBORw0KGgoAAAANSUhEUgAABUYAAAMHCAYAAAD1lY2SMAAA7DAc..."
]
```

**HTTP 400** if no consignments are supplied or any are invalid.

---

## Printers — GET /api/printers

List printers available to your account.

**URL:** `https://api.gosweetspot.com/api/printers`
**Method:** `GET`
**Content-Type:** `application/json`

### Headers
| Header | Description |
|---|---|
| `access_key` | Unique API key |
| `site_id` | Site identifier |

### Response — Printer Object
| Field | Type | Description |
|---|---|---|
| `Printer` | string | Print agent name to reference when creating shipments |
| `IsLabelPrinter` | boolean | Whether device handles label printing |
| `IsOnline` | boolean | Current online status |
| `LabelType` | integer | Format type: `1`=EPL_LP2844, `2`=EPL_INTERMEC, `3`=ZPL_GK420D, `4`=PNG, `5`=PDF |
| `Name` | string | Display name |
| `PrinterPath` | string | Hardware path |
| `FullName` | string | Complete hierarchical printer name |

### Response Example
```json
[
  {
    "Printer": "Warehouse >> NPIA1A447 (HP COLOR LASERJET MFP M277DW) (11111)",
    "IsLabelPrinter": true,
    "IsOnline": true,
    "LabelType": 4,
    "Name": "NPIA1A447 (HP Color LaserJet MFP M277dw)",
    "PrinterPath": "NPIA1A447 (HP Color LaserJet MFP M277dw)",
    "FullName": "SWEET SPOT GROUP LTD >> Warehouse >> NPIA1A447 (HP COLOR LASERJET MFP M277DW) (11111)"
  }
]
```

---

## Labels — GET /api/labels

Download labels for a shipment.

**URL:** `https://api.gosweetspot.com/api/labels`
**Method:** `GET`
**Content-Type:** `application/json`

### Headers
| Header | Description |
|---|---|
| `access_key` | Unique API key |
| `site_id` | Site identifier |

### Query Parameters
| Parameter | Type | Description |
|---|---|---|
| `format` | string | Label format. Options: `LABEL_PDF`, `LABEL_PNG_100X175`, `LABEL_PNG_100X150`, `LABEL_PDF_100X175`, `LABEL_PDF_100X150`, `LABEL_ZPL_100X150`, `LABEL_ZPL_100X150_300DPI`, `LABEL_PDF_LABELOPE`, `USER_CONFIGURED`, `GOPRINT_PRN`. Default `LABEL_PNG_100X175`. |
| `rotate` | boolean | 180-degree rotation |
| `connote` | string | Consignment number |

### Response
Array of base64-encoded binary strings. Multi-part PNG shipments are split; PDF combines parts into one document.

```json
[
  "iVBORw0KGgoAAAANSUhEUgAABUYAAAMHCAYAAAD1lY2SAAAAAXNS...",
  "iVBORw0KGgoAAAANSUhEUgAABUYAAAMHCAYAAAD1lY2SMAAA7DAc..."
]
```

### cURL
```bash
curl --location 'https://api.gosweetspot.com/api/labels?format=label_png&connote=ABA0010054' \
  --header 'access_key;' \
  --header 'site_id;' \
  --header 'Content-Type: application/json'
```

---

## Labels — POST /api/labels

Requeue a shipment for label printing.

**URL:** `https://api.gosweetspot.com/api/labels`
**Method:** `POST`
**Content-Type:** `application/json`

### Query Parameters
| Parameter | Type | Description |
|---|---|---|
| `connote` | string | Consignment number to print |
| `printtoprinter` | string | Optional; uses access_key profile printer if omitted |

### Response (plain text)
- `"Not a valid consignment number"`
- `"Print job sent"`
- `"Print job queued"`
- `"API Key not associated with a valid printer"`

### cURL
```bash
curl --location --request POST 'https://api.gosweetspot.com/api/labels?connote=ABC123' \
  --header 'access_key;' \
  --header 'site_id;' \
  --header 'Content-Type: application/json'
```

---

## Labels Enqueue — POST /api/labels/enqueue

Queue a raw image for printing via the print application.

**URL:** `https://api.gosweetspot.com/api/labels/enqueue`
**Method:** `POST`
**Content-Type:** `application/json`

### Headers
| Header | Description |
|---|---|
| `access_key` | Unique API key |
| `site_id` | Site identifier |

### Request Body
| Parameter | Type | Description |
|---|---|---|
| `Copies` | integer | Number of copies to print |
| `Image` | string | Base64-encoded PNG, **1350px × 1175px** |
| `PrintToPrinter` | string | Optional printer; uses profile default if omitted |

### Response (plain text)
- `"Supplied document stream does not represent an image."`
- `"Print job sent."`
- `"Print job queued."`
- `"API Key not associated with a valid printer."`

### cURL
```bash
curl --location 'https://api.gosweetspot.com/api/labels/enqueue' \
  --header 'access_key;' \
  --header 'site_id;' \
  --header 'Content-Type: application/json' \
  --data '{
    "Copies": 1,
    "Image": "LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0t...",
    "PrintToPrinter": "SMITH-PC >> ZDESIGNER LP 2844 (3227)"
  }'
```

---

## Pickup Booking — POST /api/bookpickup

Book a driver pickup with a supported carrier.

**URL:** `https://api.gosweetspot.com/api/bookpickup`
**Method:** `POST`
**Content-Type:** `application/json`

### Headers
| Header | Description |
|---|---|
| `access_key` | Unique API key |
| `site_id` | Site identifier |

### Request Body
| Parameter | Type | Required For | Description |
|---|---|---|---|
| `Carrier` | string | All requests | One of: `CastleParcel`, `PostHaste`, `NZCouriers`, `Mainstream`, `FedEx`, `NZPost`, `FirstGlobal` |
| `Consignments` | string[] | Mainstream, FedEx, NZPost | Consignment identifiers |
| `TotalKg` | decimal | FirstGlobal | Total parcels weight |
| `Parts` | integer | FirstGlobal | Number of parcels |

### Response (plain text)
```
Success. Your booking has been accepted. #NC12345678
```

### Examples

**FedEx / NZ Post / Mainstream:**
```bash
curl --location 'https://api.gosweetspot.com/api/bookpickup' \
  --header 'access_key;' \
  --header 'site_id;' \
  --header 'Content-Type: application/json' \
  --data '{"Carrier": "FedEx", "Consignments": ["789663088020"]}'
```

**First Global:**
```bash
curl --location 'https://api.gosweetspot.com/api/bookpickup' \
  --header 'access_key;' \
  --header 'site_id;' \
  --header 'Content-Type: application/json' \
  --data '{"Carrier": "FirstGlobal", "TotalKg": 1.5, "Parts": 1}'
```

**Post Haste / Castle Parcels / NZ Couriers:**
```bash
curl --location 'https://api.gosweetspot.com/api/bookpickup' \
  --header 'access_key;' \
  --header 'site_id;' \
  --header 'Content-Type: application/json' \
  --data '{"Carrier": "PostHaste"}'
```

---

## Stock Sizes — GET /api/stocksizes

List available stock sizes for your account.

**URL:** `https://api.gosweetspot.com/api/stocksizes`
**Method:** `GET`
**Content-Type:** `application/json`

### Headers
| Header | Description |
|---|---|
| `access_key` | Unique API key |
| `site_id` | Site identifier |

### Response
Array of [Stock Size Model](#stock-size-model) objects.

### cURL
```bash
curl --location 'https://api.gosweetspot.com/api/stocksizes' \
  --header 'access_key;' \
  --header 'site_id;' \
  --header 'Content-Type: application/json'
```

---

# Webhooks

Webhooks provide real-time feedback for shipment events.

**Configuration URL:** `https://ship.gosweetspot.com/developer/webhooks`

### Subscribable Events
- Shipment creation
- Courier pickup registration
- Delivery confirmation
- Shipment tracking activity update (coming soon)

### Technical Requirements

- **Timeout:** 10 seconds. Failed/timeout requests are queued for retry.
- **Retry logic:** Starts at 10 minutes, extends by 10 minutes each retry, aborts after 10 attempts.
- **Idempotency:** Your endpoint must handle the same shipment posting multiple times in any order.
- **Batch size:** Up to 50 shipments per webhook batch.
- Respond with valid HTTP codes promptly.

### Payload Structure
Each POST is a JSON array of shipment objects:

| Field | Type | Description |
|---|---|---|
| `ConsignmentNo` | string | Shipment ID |
| `Consignee` | string | Recipient name |
| `Status` | string | Latest tracking status |
| `Picked` | datetime | Pickup timestamp |
| `Delivered` | datetime | Delivery timestamp |
| `Tracking` | string | Track and trace URL |
| `TotalCost` | decimal | Shipment cost (ex tax) |
| `ManualTicket` | boolean | `true` = UI-created, `false` = integrated |
| `PackingSlipNo` | string | Order/delivery reference |
| `CreatedUtc` | datetime | Shipment creation timestamp |

### Debugging
Tools like `requestb.in` can help capture and inspect webhook payloads.

---

# Shipping Options

Enables merchants to retrieve shipping rates for their websites after setting up the GoSweetSpot Shipping Options App.

**URL Format:** `https://checkout.gosweetspot.com/CustomApi/Rates/{your shipping options app ID}`
**Method:** `POST`
**Content-Type:** `application/json`

### Authentication
HMAC-SHA256 via the `X-GSS-Hmac-Sha256` header, generated from your request body and an assigned secret.

### Request Body
| Parameter | Type | Description |
|---|---|---|
| `weight` | decimal | Total shipment weight in kg |
| `cartvalue` | decimal | Combined item value |
| `destination` | object | [Shipping Options Address Model](#shipping-options-address-model) |

### Response
JSON array of rate objects:

| Field | Type | Description |
|---|---|---|
| `description` | string | Rate label |
| `shortcode` | string | Abbreviated rate identifier |
| `rate` | decimal | Decimal price |

> This API is distinct from `POST /api/rates` and requires preliminary app configuration.

---

# Models

## Contact Model

| Field | Type | Description |
|---|---|---|
| `name` | string | Company or person name (max 50) |
| `contactperson` | string | Contact person (max 50, optional) |
| `phonenumber` | string | Phone number (max 50, optional) |
| `deliveryinstructions` | string | Label instructions (max 120). Ignored for origin |
| `sendtrackingemail` | boolean | Optional, send tracking email |
| `costcentreid` | integer | Optional, cost centre to use |
| `address` | object | [Contact Address Model](#contact-address-model) |

## Contact Address Model

| Field | Type | Description |
|---|---|---|
| `buildingname` | string | Unit, Level, etc. (max 50) |
| `streetaddress` | string | Street number and name (max 50) |
| `suburb` | string | Suburb (max 50) |
| `city` | string | City or state (use state abbreviations) (max 50) |
| `postcode` | string | Postal code (max 10) |
| `countrycode` | string | ISO Alpha 2 (max 2) |

## Rate Package Model

| Field | Type | Description |
|---|---|---|
| `name` | string | Configured Stock Size name; otherwise `"custom"` (max 50) |
| `length` | integer | Length in cm |
| `width` | integer | Width in cm |
| `height` | integer | Height in cm |
| `kg` | decimal | Weight in kg |
| `type` | string | Box, Carton, Satchel, Bag, Pallet, etc. (max 10) |
| `hasDG` | boolean | Contains dangerous goods? |

## Customer Orders Model

| Field | Type | Description |
|---|---|---|
| `packingslipno` | string | Unique order/packing slip number |
| `consignee` | string | Recipient name |
| `address1` | string | Street number |
| `address2` | string | Street name |
| `suburb` | string | Suburb |
| `city` | string | City or state (abbreviation) |
| `postcode` | string | Postal code |
| `country` | string | ISO Alpha 2 |
| `delvref` | string | Customer reference |
| `delvinstructions` | string | Label instructions |
| `contactname` | string | Optional |
| `contactphone` | string | Optional |
| `email` | string | Email address |
| `isrural` | string | Rural address flag |
| `notrural` | string | Override non-rural |
| `costcentre` | string | Cost centre to use |
| `status` | object | [Status Model](#status-model) |
| `products` | array | [Product Model](#product-model) |
| `customField1Value` | string | Custom field 1 |
| `customField2Value` | string | Custom field 2 |
| `shippingDescription` | string | Shipping reference |

## Customer Orders Package Model

| Field | Type | Description |
|---|---|---|
| `packageStockName` | string | Optional; matches pre-configured package or `--Custom--` |
| `quantity` | int | Optional, number of packages |
| `weightKg` | float | Optional, overrides default if provided |
| `lengthCm` | float | Optional; ignored for some SATCHEL types |
| `widthCm` | float | Optional; ignored for some SATCHEL types |
| `heightCm` | float | Optional; ignored for some SATCHEL types |

## Product Model

| Field | Type | Description |
|---|---|---|
| `productcode` | string | SKU |
| `description` | string | Name |
| `units` | integer | Units ordered |
| `unitvalue` | decimal | Value per unit |
| `countryofManufacture` | string | ISO Alpha 2 |
| `imageurl` | string | Product image |
| `currency` | string | 3-letter currency code |
| `alreadySent` | integer | Quantity fulfilled prior |
| `fulfilledQty` | integer | Quantity fulfilled on this order |
| `linetotal` | decimal | Total value of this line |
| `bin` | string | Picking location |

## Dangerous Goods Model

| Field | Type | Description |
|---|---|---|
| `additionalhandinginfo` | string | Additional handling info (max 100) |
| `hazchemcode` | string | Hazchem code |
| `isDGLQ` | boolean | Limited quantities? |
| `totalQuantity` | string | Total quantity, required (max 200) |
| `totalKg` | decimal | Total weight kg, required |
| `signoffName` | string | Optional (max 50) |
| `signoffRole` | string | Optional (max 50) |
| `lineitems` | array | [Dangerous Goods Item Model](#dangerous-goods-item-model) |

## Dangerous Goods Item Model

| Field | Type | Description |
|---|---|---|
| `description` | string | Max 100 |
| `classordivision` | string | Max 50 |
| `unoridno` | string | Max 50 |
| `packinggroup` | string | Max 50 |
| `subsidaryrisk` | string | Max 50 |
| `packing` | string | Max 50 |

## Commodity Model

| Field | Type | Description |
|---|---|---|
| `harmonizedCode` | string | Optional, 6-digit HS tariff code |
| `description` | string | Commodity description |
| `unitvalue` | decimal | Value per unit |
| `units` | int | Number of commodities |
| `unitKg` | decimal | Weight per unit (kg) |
| `currency` | string | Currency, e.g. NZD, AUD, USD (max 3) |
| `country` | string | ISO Alpha 2 country of manufacture (max 2) |

## Available Rate Model

| Field | Type | Description |
|---|---|---|
| `quoteId` | GUID | Unique rate identifier |
| `carrierId` | integer | Carrier ID |
| `carriername` | string | Display name |
| `deliverytype` | string | Courier service type |
| `cost` | decimal | Freight cost |
| `charge` | decimal | Marked up cost |
| `servicestandard` | string | Service wording |
| `comments` | string | Extra rate comments |
| `route` | string | Provider routing |
| `isruraldelivery` | boolean | Rural delivery? |
| `issaturdaydelivery` | boolean | Saturday delivery? |
| `isresidential` | boolean | Residential delivery? |
| `carrierservicetype` | string | Provider service type |

## Reject Rate Model

| Field | Type | Description |
|---|---|---|
| `carriername` | string | Carrier name |
| `deliverytype` | string | Service type |
| `reason` | string | Why rate was rejected |

## Rate Validation Error Model

| Field | Type | Description |
|---|---|---|
| `key` | string | Field name |
| `value` | string | Reason of validation failure |

## Consignment Model

| Field | Type | Description |
|---|---|---|
| `connote` | string | Carrier connote number |
| `trackingurl` | string | Track and trace URL |
| `cost` | decimal | Total cost (ex tax) |
| `charge` | decimal | Marked-up cost |
| `carriertype` | string | Internal carrier classification |
| `issaturdaydelivery` | boolean | Saturday delivery applied? |
| `isrural` | boolean | Rural? |
| `isovernight` | boolean | Overnight (inter-island)? |
| `hastrackpaks` | boolean | Has trackpaks? |
| `outputs` | string[] | Output files as base64 strings |

## Shipment Model

| Field | Type | Description |
|---|---|---|
| `ConsignmentNo` | string | Consignment number |
| `Consignee` | string | Consignee name |
| `ManualTicket` | boolean | `false` = integrated; `true` = GSS UI |
| `PackingSlipNo` | string | Order/packing slip / delivery reference |
| `Picked` | datetime (nullable) | Pickup time (local) |
| `Delivered` | datetime (nullable) | Delivery time (local) |
| `Status` | string | Latest tracking status |
| `TotalCost` | decimal | Total cost (ex tax) |
| `TotalCharge` | decimal | Marked-up cost |
| `Tracking` | string | Tracking URL |
| `OriginZone` | string | Origin zone short code |
| `DestinationZone` | string | Destination zone short code |
| `CostCentre` | string | Cost centre name |
| `Carrier` | string | Carrier name |
| `DeliveryInstructions` | string | Driver instructions |
| `IsSaturdayDelivery` | bool | Saturday delivery? |
| `IsRuralDelivery` | bool | Rural? |
| `IsPOBox` | bool | PO Box / ParcelPod? |
| `CustomerRef` | string | Reference |
| `TotalCubic` | decimal | Total cubic volume (m³) |
| `CreatedUtc` | datetime | Created |
| `CreatedBy` | string | Creator |
| `TotalKg` | decimal | Total weight |
| `Parts` | int | Number of items |
| `IsSignatureRequired` | bool | Signature required? |
| `IsFreightForward` | bool | Freight-forward? |
| `ManifestedAt` | datetime (nullable) | Manifest time |
| `ManifestNumber` | string | Manifest number |
| `Origin` | object | [Contact Model](#contact-model) |
| `Destination` | object | [Contact Model](#contact-model) |
| `Items` | array | [Shipment Package Model](#shipment-package-model) |
| `CustomField1Name` | string | Custom field 1 name |
| `CustomField1Value` | string | Custom field 1 value |
| `CustomField2Name` | string | Custom field 2 name |
| `CustomField2Value` | string | Custom field 2 value |

## Shipment Package Model

| Field | Type | Description |
|---|---|---|
| `PartNo` | integer | Part number (1, 2, 3, ...) |
| `LengthCm` | decimal | Length in cm |
| `WidthCm` | decimal | Width in cm |
| `HeightCm` | decimal | Height in cm |
| `WeightKg` | decimal | Weight in kg |
| `PackageName` | string | e.g. GSS A4 Satchel |
| `Charge_LineTotal` | decimal | Charge at consignment creation |
| `Charge_MarkedUpLineTotal` | decimal | Charge with markup |
| `PickedAt` | datetime (nullable) | Pickup time |
| `DeliveredAt` | datetime (nullable) | Delivery time |
| `RatingCode` | string | Rating code |
| `Events` | array | [Shipment Event Model](#shipment-event-model) |

## Shipment Event Model

| Field | Type | Description |
|---|---|---|
| `Part` | string | Part number |
| `Code` | string | Milestone code: `CR` Created, `PUP` Picked up, `UPD` Status update, `EXP` Exception/service update, `DEL` Delivered |
| `Description` | string | Event description |
| `eventDt` | datetime | Event time (local) |
| `Location` | string | Event locality |

## V2 Shipment Status Event Model

| Field | Type | Description |
|---|---|---|
| `eventdate` | datetime | Event date/time (local) |
| `code` | string | `CR` Created, `INTL` International transit, `CUST` Customs cleared, `COUR` Courier picked up, `COURU` Courier update, `DEL` Delivered |
| `description` | string | Event description |
| `location` | string | Locality |
| `part` | string | Shipment item identifier |

## Status Model

| Field | Type | Description |
|---|---|---|
| `status` | string | Current status |
| `ticketnumber` | string | Ticket number if printed |
| `trackingurl` | string | Tracking URL |
| `picked` | datetime | Pickup time (local) |
| `delivered` | datetime | Delivery time (local) |
| `manualticket` | boolean | Created manually vs via channel feed |

## Print Output Formats

| Type | Description |
|---|---|
| `LABEL_PNG_100X175` | PNG image, 100mm × 175mm |
| `LABEL_PNG_100X150` | PNG image, 100mm × 150mm |
| `LABEL_PDF_100X175` | PDF, 100mm × 175mm |
| `LABEL_PDF_100X150` | PDF, 100mm × 150mm |
| `LABEL_ZPL_100X150` | 200 DPI ZPL, 100mm × 150mm |
| `LABEL_ZPL_100X150_300DPI` | 300 DPI ZPL, 100mm × 150mm |
| `LABEL_PDF_LABELOPE` | PDF Labelope. Rotation not supported |
| `USER_CONFIGURED` | Adapts per user's print settings (PDF, PRN, or Print Agent trigger) |
| `GOPRINT_PRN` | Base64 PRN file. Decode and save to GoPrint monitored folder |

## Stock Size Model

| Field | Type | Description |
|---|---|---|
| `PackageStockId` | integer | Unique identifier |
| `Name` | string | Stock size name |
| `Height` | decimal | Height |
| `Length` | decimal | Length |
| `Width` | decimal | Width |
| `Cubic` | decimal | Cubic m³ |
| `Weight` | decimal | Weight |
| `Sort` | integer | UI ordering |
| `IsTrackPak` | boolean | Trackable? |
| `HeightAdjustable` | boolean | Can adjust height? |
| `Availability` | string | `Me_Only`, `This_Site_Only`, `Entire_Group`, `Locked` |

## V2 Address Validation Response Model

| Field | Type | Description |
|---|---|---|
| `buildingname` | string | Property identifier (max 50) |
| `streetaddress` | string | Street number and name (max 50) |
| `suburb` | string | Suburb (max 50) |
| `city` | string | City/state (max 50) |
| `postcode` | string | Postal code (max 50) |
| `countrycode` | string | ISO Alpha 2 (max 2) |
| `AvailableServices` | array | [V2 Address Validation Available Service Model](#v2-address-validation-available-service-model) |

## V2 Address Validation Available Service Model

| Field | Type | Description |
|---|---|---|
| `carrier` | string | Carrier name |
| `isresidential` | boolean | Residential service? |
| `isrural` | boolean | Rural service? |
| `hassaturdayservice` | boolean | Saturday delivery possible? |
| `branchcode` | string | Serving branch code |
| `runnumber` | string | Run number |

## V2 Validation Address Model

| Field | Type | Description |
|---|---|---|
| `buildingname` | string | Property identifier (max 50) |
| `streetaddress` | string | Street number and name (max 50) |
| `suburb` | string | Suburb (max 50) |
| `city` | string | City/state (max 50) |
| `postcode` | string | Postal code (max 50) |
| `countrycode` | string | ISO Alpha 2 (max 2) |

## Shipping Options Address Model

| Field | Type | Description |
|---|---|---|
| `Contact` | string | Contact name |
| `street` | string | Street number and name |
| `suburb` | string | Suburb |
| `city` | string | City/state (use abbreviation in countries with states) |
| `postcode` | string | Postal code |
| `countrycode` | string | ISO Alpha 2 |

---

## Quick Reference: Endpoint Summary

| Method | Endpoint | Purpose |
|---|---|---|
| GET | `/api/availableservices` | List carriers + services |
| GET | `/api/customerorders` | Retrieve published orders |
| PUT | `/api/customerorders` | Publish/update orders |
| DELETE | `/api/customerorders` | Delete unprocessed orders |
| GET | `/v2/order` | Get single order status |
| GET | `/v2/pendingorders` | List pending orders |
| POST | `/api/rates` | Get shipping rates |
| GET | `/api/shipments` | Retrieve shipments |
| POST | `/api/shipments` | Create shipment |
| DELETE | `/api/shipments` | Delete shipments |
| POST | `/v2/shipmentstatus` | Get shipment status + events |
| POST | `/v2/addressvalidation` | Validate destination address |
| POST | `/v2/publishmanifest` | Batch and manifest shipments |
| GET | `/api/printers` | List printers |
| GET | `/api/labels` | Download labels (base64) |
| POST | `/api/labels` | Queue shipment for printing |
| POST | `/api/labels/enqueue` | Queue raw image for printing |
| POST | `/api/bookpickup` | Book driver pickup |
| GET | `/api/stocksizes` | List stock sizes |

**All endpoints require headers: `access_key` and `site_id`.**
