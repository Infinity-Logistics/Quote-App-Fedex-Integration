# DHL Integration Analysis - Complete System Understanding

**Date**: October 22, 2025
**Purpose**: Baseline documentation of existing DHL integration to inform FedEx integration architecture

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [System Architecture Overview](#system-architecture-overview)
3. [Integration Point 1: Validations](#integration-point-1-validations)
4. [Integration Point 2: Rate Fetching](#integration-point-2-rate-fetching)
5. [Integration Point 3: Booking Shipments](#integration-point-3-booking-shipments)
6. [Integration Point 4: Shipping Labels](#integration-point-4-shipping-labels)
7. [Supporting Systems](#supporting-systems)
8. [Critical Observations](#critical-observations)
9. [Next Steps](#next-steps)

---

## Executive Summary

The current JobOrder module has DHL Express tightly coupled throughout the entire workflow. There are **4 primary integration points** where DHL-specific business logic and API calls are embedded:

1. **Validations** - Frontend rules enforcing DHL API constraints
2. **Rate Fetching** - Real-time rate comparison with DHL APIs
3. **Booking Shipments** - Creating shipments and generating AWBs via DHL
4. **Shipping Labels** - Receiving and storing waybill documents from DHL

Additionally, there's an **internal workflow** for ERP synchronization that uses AWB data but is carrier-agnostic in design.

---

## System Architecture Overview

### Technology Stack

**Backend**:
- Node.js / Express
- MongoDB (Mongoose ODM)
- Axios (HTTP client for DHL APIs)
- Location: `Backend-Quotation-Mgmt/src/logic/jobOrder/`

**Frontend**:
- React.js
- Ant Design components
- Location: `Frontend-Quotation-Mgmt-App/src/screens/dashboard/jobOrders/`

### Data Flow

```
Quote Created → JobOrder Generated → Review Initiated →
Rate Fetch (DHL API) → Review Completed →
AWB Creation (DHL API) → ERP Sync → AWB Sent (Email)
```

### Key Backend Files

| File | Purpose | DHL Coupling |
|------|---------|--------------|
| `JobOrder.model.js` | MongoDB schema | Medium - stores agent type |
| `JobOrderCRUD.service.js` | Core business logic | High - orchestrates DHL workflow |
| `JobOrderRateFetching.service.js` | **DHL API client** | **Critical - 100% DHL** |
| `JobOrderOptions.service.js` | DHL metadata | High - 2700+ lines of DHL constants |
| `HandleAWBAutomationFlow.service.js` | AWB generation workflow | High - calls DHL APIs |
| `CreateJobOrderInERP.service.js` | ERP synchronization | Low - carrier agnostic |
| `dhlConstants.js` | Static DHL metadata | Critical - countries, postal formats |

### Key Frontend Files

| File | Purpose | DHL Coupling |
|------|---------|--------------|
| `JobOrderReviewModal.jsx` | Main review interface | High - DHL-specific validations |
| `FetchDHLRatesModal.jsx` | Rate display modal | High - DHL-only UI |
| `jobOrderApis.js` | API client wrappers | Medium - DHL-specific endpoints |

---

## Integration Point 1: Validations

### Location
`Frontend-Quotation-Mgmt-App/src/screens/dashboard/jobOrders/JobOrderReviewModal.jsx` (Lines 67-377)

### Purpose
Enforce DHL API constraints before allowing users to proceed with booking. These are **pre-flight validations** to prevent API errors.

### DHL-Specific Rules

#### 1. Special Instructions (Line 79-82)
```javascript
if (isDHLAgent && specialInstructions && specialInstructions.length > 20) {
    errors['pickup.specialInstructions'] = "Must be less than 20 characters (DHL requirement)";
}
```
- **Constraint**: Max 20 characters
- **Reason**: DHL API field limit
- **Field**: `pickup.specialInstructionsForPickup`

#### 2. Weekend Pickup Restriction (Line 87-95)
```javascript
if (isDHLAgent) {
    const pickupDate = new Date(details.pickupDateAndTime);
    if (isWeekend(pickupDate)) {
        errors['pickupDateAndTime'] = "DHL pickup cannot be scheduled on weekends";
    }
}
```
- **Constraint**: No Saturday or Sunday pickups
- **Reason**: DHL operational policy (automated booking not available)
- **Field**: `pickupDateAndTime`

#### 3. Address Line Character Limits (Line 134-146, 202-214)
```javascript
if (isDHLAgent) {
    if (locationDetails?.addressLine1?.length > 45) {
        errors['shipper.addressLine1'] = "Too long (max 45 - DHL limit)";
    }
    // Same for addressLine2, addressLine3
}
```
- **Constraint**: Max 45 characters per address line
- **Reason**: DHL API field limit
- **Fields**: `addressLine1`, `addressLine2`, `addressLine3` (shipper & receiver)

#### 4. City Name Format (Line 151-154, 219-222)
```javascript
if (locationDetails.cityName.includes(',')) {
    errors['shipper.cityName'] = "Cannot contain commas";
}
```
- **Constraint**: No commas allowed
- **Reason**: DHL API parsing issue
- **Fields**: `cityName` (shipper & receiver)

#### 5. Package Dimensions (Line 260-274)
```javascript
if (isDHLAgent) {
    if (!dims.width || dims.width <= 0) {
        errors[`packaging.${index}.width`] = "Required (DHL)";
    }
    // Same for height, length
}
```
- **Constraint**: Width, height, length are mandatory
- **Reason**: DHL rating API requires all dimensions
- **Fields**: `containerInfo.packaging[].dimensions.{width, height, length}`
- **Non-DHL**: Dimensions are optional (manual processing)

#### 6. Item Description (Line 282-286)
```javascript
if (isDHLAgent && containerInfo.itemDescription.length > 70) {
    errors['itemDescription'] = "Too long (max 70 - DHL limit)";
}
```
- **Constraint**: Max 70 characters
- **Reason**: DHL API field limit
- **Field**: `containerInfo.itemDescription`

#### 7. Customs Declaration (Line 305-347)
```javascript
if (isDHLAgent) {
    // Strict requirements for DHL
    if (!containerInfo.exportDeclaration?.lineItems || lineItems.length === 0) {
        errors['exportDeclaration'] = "Required for DHL customs";
    }
    // Validate each line item: description, price, quantity, weight (gross & net)
    if (!containerInfo.exportDeclaration?.invoice?.number) {
        errors['invoiceNumber'] = "Required for DHL customs";
    }
    if (!containerInfo.exportDeclaration?.invoice?.date) {
        errors['invoiceDate'] = "Required for DHL customs";
    }
    if (!containerInfo.exportDeclaration?.exportReasonType) {
        errors['exportReasonType'] = "Purpose of shipment is required";
    }
}
```
- **Constraint**: All customs fields mandatory for international shipments
- **Reason**: DHL customs clearance API requirements
- **Fields**: `exportDeclaration.{lineItems, invoice, exportReasonType}`
- **Non-DHL**: Basic customs info only

### Validation Summary Table

| Field | DHL Constraint | Non-DHL | Impact on FedEx |
|-------|----------------|---------|-----------------|
| Special Instructions | ≤ 20 chars | None | **Need to check FedEx limit** |
| Pickup Date | No weekends | None | **Need to check FedEx policy** |
| Address Lines | ≤ 45 chars | None | **Need to check FedEx limit** |
| City Name | No commas | None | **Need to check FedEx parsing** |
| Package Dimensions | Required | Optional | **Check if FedEx requires** |
| Item Description | ≤ 70 chars | None | **Need to check FedEx limit** |
| Customs Fields | All required | Basic | **Check FedEx customs requirements** |

### Code Pattern
```javascript
const isDHLAgent = details?.agentConfirmedByCustomer === "DHL";

if (isDHLAgent) {
    // Apply DHL-specific validation
} else {
    // Relaxed validation for manual processing (FEDEX/UPS)
}
```

---

## Integration Point 2: Rate Fetching

### Location
`Backend-Quotation-Mgmt/src/logic/jobOrder/service/JobOrderRateFetching.service.js` (Lines 5-40)

### Purpose
Fetch live shipping rates from DHL and compare them against the rate quoted to the customer earlier.

### DHL API Endpoints

#### Production
```
GET/POST https://express.api.dhl.com/mydhlapi/rates
```

#### Test/Sandbox
```
GET/POST https://express.api.dhl.com/mydhlapi/test/rates
```

### Authentication
```javascript
headers: {
    'Authorization': 'Basic YXBVN3RQN3dENHhQOGs6VSExdE8kOXdSIzloRkAyYw==',
    'Content-Type': 'application/json'
}
```
**Note**: This is a hardcoded Base64-encoded credential (`apU7tP7wD4xP8k:U!1tO$9wR#9hF@2c`)

### 2.1 Single Package Rate Fetching

**Function**: `getDhlRatesFromAPI(params, isTestServer)`

**HTTP Method**: `GET`

**Query Parameters**:
```javascript
{
    accountNumber: "454466892",           // Hardcoded DHL account
    originCountryCode: "AE",              // ISO 2-char code
    originCityName: "Dubai",
    destinationCountryCode: "US",
    destinationCityName: "New York",
    weight: 5.0,                          // Numeric
    length: 30,                           // cm
    width: 20,                            // cm
    height: 10,                           // cm
    plannedShippingDate: "2025-10-25",    // YYYY-MM-DD format
    isCustomsDeclarable: true,
    unitOfMeasurement: "metric"           // or "imperial"
}
```

**Response Structure**:
```javascript
{
    products: [
        {
            productName: "EXPRESS WORLDWIDE",
            productCode: "P",
            localProductCode: "P",
            totalPrice: [{
                price: 350.50,
                priceCurrency: "AED"
            }],
            weight: { volumetric: 6.0 },
            totalPriceBreakdown: [
                { typeCode: "STDIS", price: 300.00 },
                { typeCode: "FUEL", price: 50.50 }
            ],
            deliveryCapabilities: {
                deliveryTypeCode: "QDDC",
                estimatedDeliveryDateAndTime: "2025-10-27T18:00:00"
            }
        }
    ]
}
```

### 2.2 Multiple Package Rate Fetching

**Function**: `getDhlRatesForMultiplePackages(bodyPayload, isTestServer)`

**HTTP Method**: `POST`

**Request Body**:
```javascript
{
    customerDetails: {
        shipperDetails: {
            cityName: "Dubai",
            countryCode: "AE",
            postalCode: "12345"
        },
        receiverDetails: {
            cityName: "New York",
            countryCode: "US",
            postalCode: "10001"
        }
    },
    accounts: [{
        typeCode: "shipper",
        number: "454466892"
    }],
    productCode: "P",
    packages: [
        {
            weight: 5.0,
            dimensions: { length: 30, width: 20, height: 10 }
        },
        {
            weight: 3.5,
            dimensions: { length: 25, width: 15, height: 8 }
        }
    ],
    plannedShippingDateAndTime: "2025-10-25T14:30:00 GMT+04:00",  // ISO with timezone
    isCustomsDeclarable: true,
    unitOfMeasurement: "metric"
}
```

**Response**: Same structure as single package (aggregated totals)

### Frontend Flow

**Location**: `JobOrderReviewModal.jsx` (Lines 536-718)

**Sequence**:
1. User clicks **"Fetch Rates"** button → `handleFetchRates()`
2. **Validation** checks required fields (Lines 547-594):
   - Shipper/receiver country codes & city names
   - Pickup date
   - Package dimensions (DHL requires all)
3. **Confirmation modal** shown → `DetailsConfirmationModal.jsx`
4. User confirms → `handleConfirmAndFetchRates()`
5. **API call logic** (Lines 619-686):
   - If single package: Call `fetchSingleContainerRatesAPI()`
   - If multiple packages: Call `fetchMultipleContainerRatesAPI()`
6. **Response processing** (Lines 689-710):
   - Filter for "EXPRESS WORLDWIDE" product
   - Extract `dhlRateInQuote` from response (the rate quoted earlier)
   - Display in `FetchDHLRatesModal.jsx`

### Rate Comparison Display

**Component**: `FetchDHLRatesModal.jsx` (Lines 56-73)

```javascript
{rates && isDHL ? (
    <ShipmentOptions fetchRates={rates} />  // Shows live vs quoted comparison
) : (
    <div>
        Rates can only be fetched from DHL. You can simply proceed from here to create an entry into the ERP.
    </div>
)}
```

### Key Field Mapping

| Frontend Field | Backend Parameter | DHL API Field |
|----------------|-------------------|---------------|
| `shipperDetails.locationDetails.countryCode` | `originCountryCode` | `originCountryCode` |
| `shipperDetails.locationDetails.cityName` | `originCityName` | `originCityName` |
| `receiverDetails.locationDetails.countryCode` | `destinationCountryCode` | `destinationCountryCode` |
| `receiverDetails.locationDetails.cityName` | `destinationCityName` | `destinationCityName` |
| `pickupDateAndTime` (UTC ISO) | `plannedShippingDate` | `plannedShippingDate` (YYYY-MM-DD) |
| `containerInfo.packaging[].weight` | `weight` | `weight` |
| `containerInfo.packaging[].dimensions.*` | `length/width/height` | `length/width/height` |

### Non-DHL Behavior
```javascript
if (!isDHLAgent) {
    setShowRatesModal(true)  // Show modal directly without API call
    return;
}
```
For non-DHL agents (FEDEX/UPS), the rate modal opens immediately without fetching live rates (manual processing).

---

## Integration Point 3: Booking Shipments

### Location
`Backend-Quotation-Mgmt/src/logic/jobOrder/service/JobOrderRateFetching.service.js` (Lines 42-317)

### Purpose
Create a shipment booking with DHL and generate an Air Waybill (AWB) number. This is the **most critical integration point** as errors here can result in financial loss.

### DHL API Endpoint

#### Production
```
POST https://express.api.dhl.com/mydhlapi/shipments
```

#### Test/Sandbox
```
POST https://express.api.dhl.com/mydhlapi/test/shipments
```

### Authentication
Same as rate fetching (hardcoded Basic Auth)

### Request Structure

**Function**: `makeBookingUsingDHL(jobOrderInfo, isTestServer)`

#### Core Payload Structure
```javascript
{
    // === PRODUCT CONFIGURATION ===
    productCode: "P",  // EXPRESS WORLDWIDE (fixed)
    accounts: [{
        typeCode: "shipper",
        number: "454466892"  // Hardcoded DHL account number
    }],

    // === PICKUP CONFIGURATION ===
    pickup: {
        isRequested: true,
        location: "Reception Area",               // Optional
        specialInstructions: [{
            value: "Call before arrival",
            typeCode: "TBD"
        }]
    },
    plannedShippingDateAndTime: "2025-10-25T14:30:00 GMT+04:00",

    // === REFERENCE TRACKING ===
    customerReferences: [{
        value: "JO_670000000000000000000001",    // MongoDB _id
        typeCode: "CU"
    }],

    // === OUTPUT CONFIGURATION ===
    outputImageProperties: {
        imageOptions: [
            {
                typeCode: "invoice",
                templateName: "COMMERCIAL_INVOICE_P_10",
                isRequested: true
            },
            {
                typeCode: "waybillDoc",
                templateName: "ECOM26_84_001",
                isRequested: true,
                hideAccountNumber: true
            }
        ]
    },

    // === CUSTOMER DETAILS ===
    customerDetails: {
        shipperDetails: { /* See below */ },
        receiverDetails: { /* See below */ }
    },

    // === CONTENT/PACKAGES ===
    content: { /* See below */ },

    // === VALUE ADDED SERVICES ===
    valueAddedServices: [
        { serviceCode: "DS" }  // or "DD"
    ],

    // === DOCUMENT ATTACHMENTS (optional) ===
    documentImages: [{ /* See below */ }]
}
```

### Detailed Field Mappings

#### 3.1 Customer Details - Shipper/Receiver

**Code**: Lines 58-106

```javascript
customerDetails: {
    shipperDetails: {
        postalAddress: {
            postalCode: "12345",
            cityName: "Dubai",
            countryCode: "AE",
            addressLine1: "Building 123, Street 45",
            addressLine2: "Near City Center",       // Optional
            addressLine3: "Business Bay Area"       // Optional
        },
        contactInformation: {
            companyName: "ABC Trading LLC",
            fullName: "John Doe",
            email: "john@abc.com",                 // Optional
            phone: "+971501234567"                 // First phone only
        }
    },
    receiverDetails: {
        // Same structure as shipperDetails
    }
}
```

**Field Mapping**:
| JobOrder Field | DHL API Path | Notes |
|----------------|--------------|-------|
| `shipperDetails.locationDetails.postalCode` | `customerDetails.shipperDetails.postalAddress.postalCode` | Required |
| `shipperDetails.locationDetails.cityName` | `customerDetails.shipperDetails.postalAddress.cityName` | Required, max 45 chars |
| `shipperDetails.locationDetails.countryCode` | `customerDetails.shipperDetails.postalAddress.countryCode` | ISO 2-char |
| `shipperDetails.locationDetails.addressLine1` | `customerDetails.shipperDetails.postalAddress.addressLine1` | Required, max 45 chars |
| `shipperDetails.locationDetails.addressLine2` | `customerDetails.shipperDetails.postalAddress.addressLine2` | Optional, max 45 chars |
| `shipperDetails.locationDetails.addressLine3` | `customerDetails.shipperDetails.postalAddress.addressLine3` | Optional, max 45 chars |
| `shipperDetails.customerDetails.companyName` | `customerDetails.shipperDetails.contactInformation.companyName` | Required |
| `shipperDetails.customerDetails.fullName` | `customerDetails.shipperDetails.contactInformation.fullName` | Required |
| `shipperDetails.customerDetails.email` | `customerDetails.shipperDetails.contactInformation.email` | Optional |
| `shipperDetails.customerDetails.phone[0]` | `customerDetails.shipperDetails.contactInformation.phone` | First phone number only |

**Helper Functions**: `constructShipperDetails()`, `constructReceiverDetails()` (Lines 58-106)

#### 3.2 Pickup Configuration

**Code**: Lines 117-170

```javascript
pickup: {
    isRequested: jobOrderInfo.pickup.isRequested,  // Boolean
    location: "Reception Area",                     // Optional
    specialInstructions: [{
        value: "Call before arrival",              // Max 20 chars
        typeCode: "TBD"
    }]
}
```

**Pickup Date Transformation**:
```javascript
// Input: UTC ISO string from frontend
// Output: UAE timezone (UTC+4) formatted string

const date = new Date(jobOrderInfo.pickupDateAndTime);  // "2025-10-25T10:30:00.000Z"
const uaeTime = new Date(date.getTime() + (4 * 60 * 60 * 1000));
const formatted = `${year}-${month}-${day}T${hour}:${minute}:${second} GMT+04:00`;
// Result: "2025-10-25T14:30:00 GMT+04:00"
```

#### 3.3 Packages/Content

**Code**: Lines 192-248

```javascript
content: {
    packages: [
        {
            weight: 5.0,
            dimensions: {
                width: 20,
                height: 10,
                length: 30
            }
        }
    ],
    description: "Electronic Components",           // Max 70 chars
    isCustomsDeclarable: true,
    declaredValue: 500,
    declaredValueCurrency: "AED",
    unitOfMeasurement: "metric",
    incoterm: "DAP",                                 // or "DDP"

    exportDeclaration: { /* See customs section */ }
}
```

**Package Expansion Logic** (Lines 195-208):
```javascript
// If packaging has quantity > 1, expand into individual packages
jobOrderInfo.containerInfo.packaging.forEach(pkg => {
    for (let i = 0; i < pkg.quantity; i++) {
        packages.push({
            weight: pkg.weight,
            dimensions: pkg.dimensions
        });
    }
});

// Example:
// Input:  [{ weight: 5, quantity: 3, dimensions: {...} }]
// Output: [{ weight: 5, dimensions: {...} }, { weight: 5, dimensions: {...} }, { weight: 5, dimensions: {...} }]
```

#### 3.4 Customs Declaration

**Code**: Lines 224-243

```javascript
exportDeclaration: {
    lineItems: [
        {
            number: 1,                              // Sequential numbering
            description: "Smartphone",
            price: 800,
            quantity: {
                value: 2,
                unitOfMeasurement: "PCS"
            },
            weight: {
                grossValue: 1.5,                     // kg
                netValue: 1.2                        // kg
            },
            manufacturerCountry: "AE"
        }
    ],
    invoice: {
        date: "2025-10-25",                         // YYYY-MM-DD
        number: "INV-2025-001234"
    },
    exportReason: "commercial",                     // Fixed
    exportReasonType: "commercial_purpose_or_sale"  // From dropdown
}
```

**Line Items Transformation**:
```javascript
lineItems: jobOrderInfo.containerInfo.exportDeclaration.lineItems.map((lineItem, index) => {
    const {_id, ...rest} = lineItem;  // Remove MongoDB _id
    return {
        number: index + 1,
        manufacturerCountry: jobOrderInfo.containerInfo.manufacturerCountry,
        ...rest
    }
})
```

#### 3.5 Value Added Services

**Code**: Lines 268-282

```javascript
// Determine service code based on Incoterm
if (content.incoterm === "DDP") {
    // Delivered Duty Paid - shipper pays all duties/taxes
    valueAddedServices.push({ serviceCode: "DD" });
} else {
    // All other incoterms (DAP, DDU, EXW, FCA, CPT, CIP, DAT, DPU, DAF)
    // Receiver pays duties and taxes
    valueAddedServices.push({ serviceCode: "DS" });
}
```

**Incoterm Mapping**:
| Incoterm | Service Code | Meaning |
|----------|--------------|---------|
| DDP | DD | Duties & Taxes Paid (by shipper) |
| DAP, DDU, EXW, FCA, CPT, CIP, DAT, DPU, DAF | DS | Duties & Taxes Unpaid (by receiver) |

#### 3.6 Document Images (Optional)

**Code**: Lines 250-257

```javascript
documentImages: [
    {
        typeCode: "INV",                           // Invoice
        imageFormat: "PDF",
        content: "JVBERi0xLjQKJeLjz9MK..."        // Base64 encoded PDF
    }
]
```

**Note**: If provided, DHL will use customer's invoice instead of generating one.

### Response Structure

```javascript
{
    shipmentTrackingNumber: "1234567890",
    dispatchConfirmationNumber: "PRG999999999999999",
    cancelPickupUrl: "https://api.dhl.com/cancel/...",
    trackingUrl: "https://www.dhl.com/en/express/tracking.html?AWB=1234567890",

    packages: [
        {
            referenceNumber: 1,
            trackingNumber: "JD012345678901234567",
            trackingUrl: "https://www.dhl.com/..."
        }
    ],

    documents: [
        {
            imageFormat: "PDF",
            content: "JVBERi0xLjQKJeLjz9MK...",    // Base64 encoded
            typeCode: "label"
        },
        {
            imageFormat: "PDF",
            content: "JVBERi0xLjQKJeLjz9MK...",
            typeCode: "invoice"
        },
        {
            imageFormat: "PDF",
            content: "JVBERi0xLjQKJeLjz9MK...",
            typeCode: "waybillDoc"
        }
    ],

    shipmentsMetaData: {
        productCode: "P",
        localProductCode: "P",
        productName: "EXPRESS WORLDWIDE"
    }
}
```

### Workflow Integration

**Location**: `JobOrderCRUD.service.js` (Lines 371-445)

```javascript
async function changeJobOrderStatus(id, statusData) {
    if (status === "REVIEW_COMPLETED") {
        // === NEW 2-STEP WORKFLOW ===

        // STEP 1: Create AWB using DHL API
        const awbRes = await handleAWBAutomationFlow.startAWBAutomationFlow(
            jobOrder,
            normalJobOrder,
            quotation,
            changeJobOrderStatus,
            true  // skipStatusUpdate = true (don't mark as AWB_GENERATED yet)
        );

        if (!awbRes.success) {
            // Handle AWB creation failure
            return responseModel(false, awbRes.message);
        }

        // STEP 2: Create Job Order in ERP with AWB details
        const erpRes = await createJobOrderInERPService.createJobOrderInERP(
            jobOrder,
            normalJobOrder,
            quotation,
            changeJobOrderStatus,
            awbRes.data.awbDetails  // Pass AWB data to ERP
        );

        if (!erpRes.success) {
            // AWB created but ERP sync failed
            // Manual intervention required
            return responseModel(false, erpRes.error);
        }

        // Both steps successful
        // Status automatically updated to AWB_GENERATED in CreateJobOrderInERP
        return responseModel(true, "Job order created successfully");
    }
}
```

**Status Flow**:
```
REVIEW_COMPLETED →
    makeBookingUsingDHL() →
        createJobOrderInERP() →
            AWB_GENERATED (automatic)
```

### Critical Notes

1. **DHL Account**: `454466892` is hardcoded (Line 113)
2. **Product Code**: `"P"` (EXPRESS WORLDWIDE) is fixed (Line 109)
3. **Test Server Toggle**: Determined by `isTestServer` parameter or `process.env.NODE_ENV === 'development'`
4. **Authorization**: Hardcoded Basic Auth credentials
5. **Timezone**: All dates converted from UTC to UAE (GMT+04:00)
6. **Phone Numbers**: Only first phone number used (Line 74, 99)
7. **MongoDB _id**: Stripped from export declaration line items (Line 228)

---

## Integration Point 4: Shipping Labels

### Location
Labels are embedded in the booking response (Integration Point 3)

### Purpose
Receive shipping labels (waybills) and invoices as base64-encoded PDFs for printing and email distribution.

### Configuration

**Request** (Lines 125-139 in `makeBookingUsingDHL`):
```javascript
outputImageProperties: {
    imageOptions: [
        {
            typeCode: "invoice",
            templateName: "COMMERCIAL_INVOICE_P_10",
            isRequested: true
        },
        {
            typeCode: "waybillDoc",
            templateName: "ECOM26_84_001",
            isRequested: true,
            hideAccountNumber: true
        }
    ]
}
```

### Document Types

| Type Code | Template Name | Purpose |
|-----------|---------------|---------|
| `invoice` | `COMMERCIAL_INVOICE_P_10` | Commercial invoice for customs |
| `waybillDoc` | `ECOM26_84_001` | Air waybill / shipping label |
| `label` | (auto-generated) | Package label |

### Response Structure

```javascript
documents: [
    {
        imageFormat: "PDF",
        content: "JVBERi0xLjQKJeLjz9MKMyAwIG9iago8P...",  // Base64
        typeCode: "label"
    },
    {
        imageFormat: "PDF",
        content: "JVBERi0xLjQKJeLjz9MK...",
        typeCode: "invoice"
    },
    {
        imageFormat: "PDF",
        content: "JVBERi0xLjQKJeLjz9MK...",
        typeCode: "waybillDoc"
    }
]
```

### Storage

**Location**: `HandleAWBAutomationFlow.service.js` (Lines 55-100)

```javascript
await changeJobOrderStatus(jobOrderId, {
    status: "AWB_GENERATED",
    updatedJobOrder: {
        ...normalJobOrderInfo,
        bookedShipmentDetails: {
            // === DHL API Response ===
            shipmentTrackingNumber: "1234567890",
            dispatchConfirmationNumber: "PRG999999999999999",
            trackingUrl: "https://www.dhl.com/...",
            cancelPickupUrl: "https://...",
            packages: [...],
            documents: [...]  // All PDF documents stored here
        },
        erpInfo: {
            ...normalJobOrderInfo.erpInfo,
            bookingReference: dhlShipmentDetails.dispatchConfirmationNumber
        }
    }
});
```

**MongoDB Schema** (`JobOrder.model.js`, Lines 163-178):
```javascript
bookedShipmentDetails: {
    type: mongoose.Schema.Types.Mixed,  // Flexible schema for carrier responses
    required: false
}
```

### Usage

#### Email Distribution
**Location**: `SendWayBillOverEmail.service.js`

```javascript
async function sendWayBillOverEmail(jobOrderInfo, fromInbox, quotationInfo) {
    // Extract documents from bookedShipmentDetails
    const documents = jobOrderInfo.bookedShipmentDetails.documents;

    // Filter for waybill
    const waybillDoc = documents.find(doc =>
        doc.typeCode === "waybillDoc" || doc.typeCode === "label"
    );

    // Decode base64 and attach to email
    const pdfBuffer = Buffer.from(waybillDoc.content, 'base64');

    // Send email with PDF attachment
    await sendEmail({
        to: quotationInfo.customerInfo.email,
        subject: `Shipment AWB - ${jobOrderInfo.bookedShipmentDetails.shipmentTrackingNumber}`,
        attachments: [{
            filename: `AWB_${jobOrderInfo.bookedShipmentDetails.shipmentTrackingNumber}.pdf`,
            content: pdfBuffer
        }]
    });
}
```

#### Manual Download (Unused)
**Function**: `getDhlShipmentImage(shipmentId, accountNumber, typeCode, pickupYearAndMonth)`

**Note**: This function exists but is not currently used in the workflow. Labels are retrieved from the initial booking response instead.

```javascript
async function getDhlShipmentImage(shipmentId, shipperAccountNumber = "454466892", typeCode = "waybill", pickupYearAndMonth) {
    const dhlURL = `https://api-mock.dhl.com/mydhlapi/shipments/${shipmentId}/get-image`;

    const response = await axios.get(dhlURL, {
        params: {
            shipperAccountNumber,
            typeCode,
            pickupYearAndMonth  // Format: "YYYY-MM"
        },
        headers: {
            'Authorization': 'Basic YXBVN3RQN3dENHhQOGs6VSExdE8kOXdSIzloRkAyYw=='
        }
    });

    return responseModel(true, 'Shipment image fetched successfully', response.data);
}
```

### Key Points

1. **No Separate API Call**: Labels are included in the booking response
2. **Base64 Encoding**: All documents are base64-encoded PDF strings
3. **Multiple Documents**: Typically 2-3 documents per shipment
4. **Stored in DB**: Full documents stored in `bookedShipmentDetails.documents[]`
5. **Email Automation**: Automatically sent after status change to `SENDING_AWB`

---

## Supporting Systems

### DHL Constants & Metadata

**Location**: `Backend-Quotation-Mgmt/src/logic/jobOrder/dhlConstants.js`

**Size**: 2,700+ lines

**Contents**:

#### 1. Countries Metadata (1,640 lines)
```javascript
DHL_COUNTRIES_META_DATA = [
    {
        countryName: "United Arab Emirates",
        countryCode: "AE",
        postalCode: "required",
        stateCode: "optional",
        domesticShippingAvailable: true,
        internationalShippingAvailable: true
    },
    // ... 238 more countries
]
```

**Usage**:
- Country dropdowns in frontend
- Validation of country codes
- Postal code format requirements

#### 2. Postal Code Formats (383 lines)
```javascript
DHL_POSTAL_CODE_FORMATS = {
    "US": {
        formats: ["99999", "99999-9999"],
        regex: "^\\d{5}(-\\d{4})?$"
    },
    "AE": {
        formats: ["999999"],
        regex: "^\\d{6}$"
    },
    // ... 150+ countries
}
```

**Usage**:
- Frontend validation of postal codes
- API endpoint: `/job-orders/dhl/validate-postal-code`

#### 3. Units of Measurement (593 lines)
```javascript
DHL_UNITS = [
    [
        { attribute: "unitOfMeasurement", value: "PCS" },
        { attribute: "description", value: "Pieces" }
    ],
    [
        { attribute: "unitOfMeasurement", value: "KG" },
        { attribute: "description", value: "Kilograms" }
    ],
    // ... 90+ units
]
```

**Usage**:
- Customs declaration dropdowns
- Export line item unit selection
- Transformed into key-value pairs by `JobOrderOptions.service.js`

### JobOrder Options Service

**Location**: `Backend-Quotation-Mgmt/src/logic/jobOrder/service/JobOrderOptions.service.js`

**Purpose**: Serve DHL metadata to frontend

**API Endpoints**:
```javascript
GET /job-orders/dhl-countries-metadata
GET /job-orders/dhl-postal-code-formats
GET /job-orders/dhl-units
```

**Functions**:

1. `getDHLCountriesMetaData()` → Returns full country list
2. `getDHLPostalCodeFormats()` → Returns postal code regex patterns
3. `getDHLUnits()` → Returns transformed units object

### ERP Synchronization

**Location**: `CreateJobOrderInERP.service.js` (Lines 15-192)

**Purpose**: Create job order entry in company's ERP system

**Trigger**: Automatically after AWB creation

**Key Points**:
1. **Carrier Agnostic**: Works with any carrier
2. **AWB Storage**: Stores AWB number in ERP record
3. **Status Determination**:
   - DHL with AWB: Status = `AWB_GENERATED`
   - Non-DHL (FEDEX/UPS): Status = `PARK_AND_CLOSE` (manual processing)
4. **Field Mapping**: JobOrder fields → ERP FormData
5. **Job Order Lookup**: Polls ERP API to find generated job ID

**FormData Fields** (Lines 449-671):
```javascript
{
    'JobOrder[quote_id]': quotationInfo.erpInfo.erpQuotationId,
    'JobOrder[customer_id]': quotationInfo.customerInfo.erpCompanyId,
    'JobOrder[shipper_id]': jobOrderData.shipperDetails.customerDetails.companyName,
    'JobOrder[consignee_id]': jobOrderData.receiverDetails.customerDetails.companyName,
    'JobOrder[origin]': getCountryValueByLabel(shipperCountry),
    'JobOrder[destination]': getCountryValueByLabel(receiverCountry),
    'JobOrder[weight]': totalGrossWeight,
    'JobOrder[volume]': totalChargeableWeight,
    'JobOrder[booking_reference]': awbDetails?.dispatchConfirmationNumber || '',
    'JobOrder[job_awb_no]': awbDetails?.shipmentTrackingNumber || 'DHL-AWB-CREATION-IP',
    'JobOrder[selling_price]': agreedPriceOfShipment,
    'JobOrder[transport_type]': "10" | "7" | "5",  // Based on domestic/international
    'JobOrder[job_order_status]': 'O',  // Open
    'JobOrderAgentDetails[0][shipper_id]': agentERPId,
    'JobOrderAgentDetails[0][awb_no]': awbNumber,
    'JobOrderAgentDetails[0][activity_type]': serviceTypeId,
    'JobOrderAgentDetails[0][planned_cost]': costPrice,
    'JobOrderAgentDetails[0][planned_selling_price]': sellingPrice,
    // ... more fields
}
```

**Non-DHL Handling** (Lines 110-118):
```javascript
if (String(normalJobOrderInfo.agentConfirmedByCustomer) !== "DHL") {
    status = JOB_ORDER_STATUS_MAP.PARK_AND_CLOSE;
    // No AWB details, manual processing required
} else if (awbDetails) {
    status = JOB_ORDER_STATUS_MAP.AWB_GENERATED;
} else {
    throw new Error('AWB details are required for DHL orders');
}
```

---

## Critical Observations

### 1. Tight Coupling

**Areas**:
- Hardcoded DHL credentials and account numbers
- DHL-specific validation rules scattered in frontend
- Direct API calls without abstraction layer
- DHL constants embedded in codebase

**Impact on FedEx Integration**:
- Cannot simply "add" FedEx; requires refactoring
- Risk of code duplication if following same pattern
- Difficult to maintain multiple carriers

### 2. Test vs Production Server

**Current Implementation**:
```javascript
const useTestServer = process.env.NODE_ENV === 'development' || isTestServer;
```

**Issues**:
- `isTestServer` parameter passed based on customer ID check
- Test server activated for "New Client (TESTING)" (ERP ID: 11057)
- No centralized configuration

**Recommendation**: Environment-based carrier configuration

### 3. Validation Strategy

**Current Approach**:
```javascript
if (isDHLAgent) {
    // Apply DHL rules
} else {
    // Relaxed rules
}
```

**Problem**: Non-DHL agents bypass all validations, but FedEx will need specific validations

**Solution Needed**: Carrier-specific validation config

### 4. Error Handling

**Strengths**:
- Comprehensive try-catch blocks
- Detailed logging with `[CreateJobOrderInERP]` prefixes
- Status tracking at each step

**Weaknesses**:
- No retry logic for API failures
- Limited error recovery (manual intervention required)
- DHL error messages not translated for user

### 5. Field Transformations

**Key Transformations**:

| Field | From | To | Code Location |
|-------|------|-----|---------------|
| Pickup Date | UTC ISO | UAE timezone string | `JobOrderRateFetching.service.js:142-157` |
| Packages | Quantity-based array | Individual packages | `JobOrderRateFetching.service.js:195-208` |
| Incoterm | DDP/DAP/etc | DD/DS service code | `JobOrderRateFetching.service.js:268-282` |
| Phone | Array of numbers | Single string | `JobOrderRateFetching.service.js:74, 99` |
| Export Line Items | MongoDB docs | Sequential numbered array | `JobOrderRateFetching.service.js:227-234` |

**FedEx Consideration**: Need to map these same transformations to FedEx API format

### 6. Status Management

**Status Flow**:
```
QUOTE_CREATED →
OPEN_FOR_REVIEW →
REVIEW_INITIATED (auto) →
REVIEW_COMPLETED (trigger: user clicks "Proceed") →
AWB_GENERATED (auto after DHL booking + ERP sync) →
SENDING_AWB (manual status change) →
AWB_SENT (auto after email sent)
```

**Code**: `JobOrderCRUD.service.js` (Lines 90-154)

**Non-DHL Status**:
```
REVIEW_COMPLETED → PARK_AND_CLOSE (no automation)
```

### 7. Data Model Flexibility

**Positive**:
- `bookedShipmentDetails` uses `mongoose.Schema.Types.Mixed`
- Can store any carrier response structure
- No schema constraints on carrier-specific data

**Negative**:
- No validation of stored data
- Difficult to query specific fields
- Potential for inconsistent data structures

---

## Next Steps

### Immediate Actions

1. **Fetch FedEx API Documentation**
   - Rate Shopping API
   - Ship API
   - Track API (optional)
   - Label/Document retrieval API

2. **Create Field Mapping Comparison**
   - DHL vs FedEx field-by-field comparison
   - Identify missing fields
   - Identify extra fields
   - Document transformation differences

3. **Analyze Validation Requirements**
   - FedEx character limits
   - FedEx date/time restrictions
   - FedEx customs requirements
   - FedEx package dimension rules

4. **Design Carrier Abstraction Layer**
   - Unified service interface
   - Carrier-specific implementations
   - Configuration management
   - Error handling strategy

### Architecture Recommendations

#### 1. Carrier Service Factory Pattern
```javascript
// carriers/CarrierFactory.js
class CarrierFactory {
    static getCarrier(carrierName) {
        switch(carrierName) {
            case 'DHL': return new DHLCarrier();
            case 'FEDEX': return new FedExCarrier();
            default: throw new Error('Unsupported carrier');
        }
    }
}

// carriers/CarrierInterface.js
class CarrierInterface {
    async getRates(request) { throw new Error('Not implemented'); }
    async bookShipment(request) { throw new Error('Not implemented'); }
    async getLabels(request) { throw new Error('Not implemented'); }
}
```

#### 2. Request/Response Transformers
```javascript
// transformers/DHLTransformer.js
class DHLTransformer {
    transformRateRequest(jobOrder) { /* ... */ }
    transformBookingRequest(jobOrder) { /* ... */ }
    parseBookingResponse(apiResponse) { /* ... */ }
}

// transformers/FedExTransformer.js
class FedExTransformer {
    transformRateRequest(jobOrder) { /* ... */ }
    transformBookingRequest(jobOrder) { /* ... */ }
    parseBookingResponse(apiResponse) { /* ... */ }
}
```

#### 3. Carrier Configuration
```javascript
// config/carriers.json
{
    "DHL": {
        "enabled": true,
        "credentials": {
            "username": "apU7tP7wD4xP8k",
            "password": "U!1tO$9wR#9hF@2c",
            "accountNumber": "454466892"
        },
        "endpoints": {
            "production": {
                "rates": "https://express.api.dhl.com/mydhlapi/rates",
                "shipments": "https://express.api.dhl.com/mydhlapi/shipments"
            },
            "test": {
                "rates": "https://express.api.dhl.com/mydhlapi/test/rates",
                "shipments": "https://express.api.dhl.com/mydhlapi/test/shipments"
            }
        },
        "validations": {
            "specialInstructions": { "maxLength": 20 },
            "addressLine": { "maxLength": 45 },
            "itemDescription": { "maxLength": 70 },
            "pickupDate": { "noWeekends": true }
        }
    },
    "FEDEX": {
        "enabled": true,
        "credentials": {
            "clientId": "${FEDEX_CLIENT_ID}",
            "clientSecret": "${FEDEX_CLIENT_SECRET}",
            "accountNumber": "TBD"
        },
        "endpoints": {
            "production": {
                "rates": "TBD",
                "shipments": "TBD"
            },
            "test": {
                "rates": "TBD",
                "shipments": "TBD"
            }
        },
        "validations": {
            "TBD": {}
        }
    }
}
```

#### 4. Frontend Validation Refactor
```javascript
// utils/carrierValidations.js
const CARRIER_VALIDATIONS = {
    DHL: {
        specialInstructions: { maxLength: 20 },
        addressLine: { maxLength: 45 },
        itemDescription: { maxLength: 70 },
        pickupDate: { noWeekends: true },
        dimensions: { required: true }
    },
    FEDEX: {
        // To be populated after API analysis
    }
};

function validateJobOrder(jobOrder, carrier) {
    const rules = CARRIER_VALIDATIONS[carrier];
    // Apply carrier-specific validations
}
```

### Risk Assessment

| Risk | Severity | Mitigation |
|------|----------|------------|
| Field mismatch between carriers | **High** | Detailed field mapping before coding |
| Different customs requirements | **High** | Compare customs APIs thoroughly |
| Timezone handling differences | **Medium** | Standardize timezone transformations |
| Authentication/credential errors | **Medium** | Test with sandbox environment first |
| Label format incompatibility | **Low** | Both use PDF format |
| Breaking existing DHL flow | **Medium** | Extensive testing of refactored code |

---

## Appendix: File Reference

### Backend Files (DHL-Related)

| File Path | Lines | Purpose |
|-----------|-------|---------|
| `logic/jobOrder/model/JobOrder.model.js` | 500+ | MongoDB schema |
| `logic/jobOrder/routes/JobOrder.routes.js` | 150+ | API routes |
| `logic/jobOrder/controller/JobOrder.controller.js` | 200+ | Request handlers |
| `logic/jobOrder/service/JobOrderCRUD.service.js` | 600+ | Core business logic |
| `logic/jobOrder/service/JobOrderRateFetching.service.js` | 346 | **DHL API client** |
| `logic/jobOrder/service/JobOrderOptions.service.js` | 56 | DHL metadata API |
| `logic/jobOrder/service/workflows/handleAWBAutomationFlow/HandleAWBAutomationFlow.service.js` | 145 | AWB workflow |
| `logic/jobOrder/service/workflows/createJobOrderInERP/CreateJobOrderInERP.service.js` | 732 | ERP sync |
| `logic/jobOrder/dhlConstants.js` | 2700+ | DHL static data |

### Frontend Files (DHL-Related)

| File Path | Lines | Purpose |
|-----------|-------|---------|
| `screens/dashboard/jobOrders/JobOrderScreen.jsx` | 800+ | Listing page |
| `screens/dashboard/jobOrders/JobOrderReviewModal.jsx` | 1010 | Review interface |
| `screens/dashboard/jobOrders/FetchDHLRatesModal.jsx` | 110 | Rate display |
| `api/jobOrderApis.js` | 151 | API wrappers |

---

## Document Version

- **Version**: 1.0
- **Date**: October 22, 2025
- **Author**: Claude Code Analysis
- **Next Update**: After FedEx API documentation review

---

**END OF DHL INTEGRATION ANALYSIS**
