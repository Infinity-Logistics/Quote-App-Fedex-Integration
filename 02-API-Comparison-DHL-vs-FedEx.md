# API Comparison: DHL Express vs FedEx

**Date**: October 22, 2025
**Purpose**: Detailed field-level comparison to guide FedEx integration

---

## Table of Contents

1. [Authentication Comparison](#authentication-comparison)
2. [Rate Fetching APIs](#rate-fetching-apis)
3. [Shipment Booking APIs](#shipment-booking-apis)
4. [Critical Differences](#critical-differences)
5. [Field Mapping Strategy](#field-mapping-strategy)
6. [Integration Approach](#integration-approach)

---

## Authentication Comparison

### DHL Express - Basic Auth

**Method**: HTTP Basic Authentication
**Headers**:
```javascript
{
    'Authorization': 'Basic YXBVN3RQN3dENHhQOGs6VSExdE8kOXdSIzloRkAyYw==',
    'Content-Type': 'application/json'
}
```

**Characteristics**:
- Static credentials (Base64 encoded `username:password`)
- No token expiration
- Same credentials for all requests
- Hardcoded in application

**Current Implementation**:
- Location: `JobOrderRateFetching.service.js`
- Credentials: `apU7tP7wD4xP8k:U!1tO$9wR#9hF@2c`
- Account Number: `454466892`

---

### FedEx - OAuth 2.0

**Method**: OAuth 2.0 Client Credentials Flow
**Endpoint**: `POST /oauth/token`

#### Authentication Request

**Base URLs**:
- **Sandbox**: `https://apis-sandbox.fedex.com`
- **Production**: `https://apis.fedex.com`

**Headers**:
```javascript
{
    'Content-Type': 'application/x-www-form-urlencoded'
}
```

**Request Body** (form-urlencoded):
```javascript
{
    grant_type: "client_credentials",
    client_id: process.env.FEDEX_CLIENT_ID,
    client_secret: process.env.FEDEX_CLIENT_SECRET
}
```

#### Authentication Response

```javascript
{
    access_token: "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    token_type: "bearer",
    expires_in: 3600,  // Token valid for 1 hour
    scope: "CXS"
}
```

**Characteristics**:
- Token-based authentication
- Tokens expire after 1 hour (3600 seconds)
- Requires token refresh mechanism
- Bearer token used in subsequent API calls

**Usage in Subsequent Requests**:
```javascript
{
    'Authorization': `Bearer ${access_token}`,
    'Content-Type': 'application/json',
    'x-customer-transaction-id': 'unique_transaction_id'  // Optional but recommended
}
```

### Implementation Requirements

**For FedEx Integration**:

1. **Token Management Service** (New)
   - Request token on application startup
   - Store token in memory with expiration timestamp
   - Auto-refresh token before expiration
   - Handle token refresh failures
   - Thread-safe token access

2. **Environment Variables** (Already in `.env`):
   ```
   FEDEX_CLIENT_ID=your_client_id
   FEDEX_CLIENT_SECRET=your_client_secret
   ```

3. **Error Handling**:
   - HTTP 401: Token expired → Refresh and retry
   - HTTP 403: Invalid credentials → Log error
   - HTTP 503: Service unavailable → Retry with backoff

---

## Rate Fetching APIs

### DHL Express - Rates API

#### Endpoint Structure

**Base URL**: `https://express.api.dhl.com/mydhlapi`
**Test URL**: `https://express.api.dhl.com/mydhlapi/test`

**Methods**:
1. **GET `/rates`** - Single piece shipment
2. **POST `/rates`** - Multi-piece shipment

---

#### DHL GET /rates (Single Piece)

**Query Parameters**:

| Parameter | Type | Required | Description | Example |
|-----------|------|----------|-------------|---------|
| `accountNumber` | string | ✅ | DHL account number | `454466892` |
| `originCountryCode` | string | ✅ | ISO 2-char country code | `AE` |
| `originCityName` | string | ✅ | Origin city | `Dubai` |
| `originPostalCode` | string | ❌ | Origin postal code | `12345` |
| `destinationCountryCode` | string | ✅ | ISO 2-char country code | `US` |
| `destinationCityName` | string | ✅ | Destination city | `New York` |
| `destinationPostalCode` | string | ❌ | Destination postal code | `10001` |
| `weight` | number | ✅ | Package weight (kg or lb) | `5.0` |
| `length` | number | ✅ | Package length (cm or in) | `30` |
| `width` | number | ✅ | Package width | `20` |
| `height` | number | ✅ | Package height | `10` |
| `plannedShippingDate` | string | ✅ | Shipping date (YYYY-MM-DD) | `2025-10-25` |
| `isCustomsDeclarable` | boolean | ✅ | Customs required | `true` |
| `unitOfMeasurement` | string | ✅ | `metric` or `imperial` | `metric` |

**Response Schema**: `supermodelIoLogisticsExpressRates`

---

#### DHL POST /rates (Multi-Piece)

**Request Body**:
```javascript
{
    customerDetails: {
        shipperDetails: {
            cityName: "Dubai",
            countryCode: "AE",
            postalCode: "12345"  // Optional
        },
        receiverDetails: {
            cityName: "New York",
            countryCode: "US",
            postalCode: "10001"  // Optional
        }
    },
    accounts: [{
        typeCode: "shipper",
        number: "454466892"
    }],
    productCode: "P",  // Optional filter
    packages: [
        {
            weight: 5.0,
            dimensions: {
                length: 30,
                width: 20,
                height: 10
            }
        },
        {
            weight: 3.5,
            dimensions: {
                length: 25,
                width: 15,
                height: 8
            }
        }
    ],
    plannedShippingDateAndTime: "2025-10-25T14:30:00 GMT+04:00",
    isCustomsDeclarable: true,
    unitOfMeasurement: "metric"
}
```

**Response Schema**: Same as GET (aggregated rates for all packages)

---

#### DHL Rate Response Structure

```javascript
{
    products: [
        {
            productName: "EXPRESS WORLDWIDE",
            productCode: "P",
            localProductCode: "P",
            localProductCountryCode: "AE",
            networkTypeCode: "TD",
            isCustomerAgreement: false,

            weight: {
                volumetric: 6.0,  // Calculated volumetric weight
                provided: 5.0,
                unitOfMeasurement: "metric"
            },

            totalPrice: [{
                price: 350.50,
                priceCurrency: "AED"
            }],

            totalPriceBreakdown: [
                {
                    currencyType: "BILLC",
                    priceCurrency: "AED",
                    priceBreakdown: [
                        {
                            typeCode: "STDIS",  // Standard discount
                            price: 300.00,
                            priceType: "TAX",
                            rate: 0,
                            basePrice: 300.00
                        },
                        {
                            typeCode: "FUEL",  // Fuel surcharge
                            price: 50.50
                        }
                    ]
                }
            ],

            deliveryCapabilities: {
                deliveryTypeCode: "QDDC",
                estimatedDeliveryDateAndTime: "2025-10-27T18:00:00",
                destinationServiceAreaCode: "NYC",
                destinationFacilityAreaCode: "JFK",
                deliveryAdditionalDays: 0,
                pickupAdditionalDays: 0,
                pickupDayOfWeek: 3,
                deliveryDayOfWeek: 5
            },

            items: [...]  // Package-level details
        }
    ],
    warnings: [...]  // Optional warnings
}
```

---

### FedEx - Rate API

#### Endpoint Structure

**Base URL**: `https://apis.fedex.com`
**Test URL**: `https://apis-sandbox.fedex.com`

**Method**: **POST `/rate/v1/rates/quotes`**

**Note**: FedEx uses POST for all rate requests (no GET option)

---

#### FedEx POST /rate/v1/rates/quotes

**Request Headers**:
```javascript
{
    'Authorization': 'Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...',
    'Content-Type': 'application/json',
    'x-customer-transaction-id': 'unique_transaction_id'  // Optional
}
```

**Request Body** (Full Schema):
```javascript
{
    accountNumber: {
        value: "XXXXXXXXX"  // FedEx account number
    },

    requestedShipment: {
        shipper: {
            address: {
                postalCode: "12345",
                city: "Dubai",
                stateOrProvinceCode: "",  // Optional
                countryCode: "AE",
                residential: false,
                streetLines: [
                    "Building 123, Street 45",
                    "Near City Center"  // Optional, max 2 lines
                ]
            },
            contact: {
                personName: "John Doe",
                phoneNumber: "+971501234567",
                companyName: "ABC Trading LLC",
                emailAddress: "john@abc.com"  // Optional
            }
        },

        recipient: {
            address: {
                postalCode: "10001",
                city: "New York",
                stateOrProvinceCode: "NY",  // Required for US/CA
                countryCode: "US",
                residential: false,
                streetLines: [
                    "456 Madison Avenue",
                    "Suite 200"
                ]
            },
            contact: {
                personName: "Jane Smith",
                phoneNumber: "+12125551234",
                companyName: "XYZ Corporation",
                emailAddress: "jane@xyz.com"
            }
        },

        shipDateStamp: "2025-10-25",  // YYYY-MM-DD

        pickupType: "USE_SCHEDULED_PICKUP",  // or "CONTACT_FEDEX_TO_SCHEDULE"

        serviceType: "INTERNATIONAL_PRIORITY",  // Optional filter

        packagingType: "YOUR_PACKAGING",  // or "FEDEX_BOX", "FEDEX_ENVELOPE", etc.

        rateRequestType: ["ACCOUNT", "LIST"],  // Array of rate types

        requestedPackageLineItems: [
            {
                weight: {
                    value: 5.0,
                    units: "KG"  // or "LB"
                },
                dimensions: {
                    length: 30,
                    width: 20,
                    height: 10,
                    units: "CM"  // or "IN"
                },
                groupPackageCount: 1
            },
            {
                weight: {
                    value: 3.5,
                    units: "KG"
                },
                dimensions: {
                    length: 25,
                    width: 15,
                    height: 8,
                    units: "CM"
                },
                groupPackageCount: 1
            }
        ],

        totalPackageCount: 2,

        totalWeight: 8.5,  // Optional, sum of all packages

        customsClearanceDetail: {
            dutiesPayment: {
                paymentType: "SENDER"  // or "RECIPIENT"
            },
            commodities: [
                {
                    description: "Electronic Components",
                    quantity: 2,
                    quantityUnits: "PCS",
                    weight: {
                        value: 5.0,
                        units: "KG"
                    },
                    customsValue: {
                        amount: 500,
                        currency: "USD"
                    },
                    countryOfManufacture: "AE"
                }
            ]
        }
    }
}
```

**Required Fields** (Minimum):
- `accountNumber`
- `requestedShipment.shipper.address.{city, countryCode, postalCode}`
- `requestedShipment.recipient.address.{city, countryCode, postalCode}`
- `requestedShipment.shipDateStamp`
- `requestedShipment.pickupType`
- `requestedShipment.requestedPackageLineItems[].weight`

---

#### FedEx Rate Response Structure

```javascript
{
    transactionId: "624deea6-b709-470c-8c39-4b5511281492",
    customerTransactionId: "unique_transaction_id",

    output: {
        rateReplyDetails: [
            {
                serviceType: "INTERNATIONAL_PRIORITY",
                serviceName: "FedEx International Priority",

                packagingType: "YOUR_PACKAGING",

                ratedShipmentDetails: [
                    {
                        rateType: "ACCOUNT",

                        totalNetCharge: 350.50,
                        totalNetChargeWithDutiesAndTaxes: 385.75,
                        totalBaseCharge: 300.00,
                        totalNetFedExCharge: 350.50,
                        totalDutiesAndTaxes: 35.25,
                        totalDiscounts: 0,

                        currency: "USD",

                        shipmentRateDetail: {
                            rateZone: "E",
                            dimDivisor: 139,
                            fuelSurchargePercent: 16.75,
                            totalSurcharges: 50.50,
                            totalFreightDiscount: 0,
                            surcharges: [
                                {
                                    type: "FUEL",
                                    description: "Fuel Surcharge",
                                    amount: 50.50
                                }
                            ],
                            pricingCode: "ACTUAL",
                            currencyExchangeRate: {
                                rate: 3.67,
                                fromCurrency: "AED",
                                intoCurrency: "USD"
                            }
                        },

                        ratedPackages: [
                            {
                                groupNumber: 0,
                                effectiveNetDiscount: 0,
                                packageRateDetail: {
                                    rateType: "PAYOR_ACCOUNT_PACKAGE",
                                    ratedWeightMethod: "ACTUAL",
                                    baseCharge: 300.00,
                                    netFreight: 300.00,
                                    totalSurcharges: 50.50,
                                    netFedExCharge: 350.50,
                                    totalTaxes: 0,
                                    netCharge: 350.50,
                                    totalRebates: 0,
                                    billingWeight: {
                                        units: "KG",
                                        value: 6.0  // Volumetric weight if higher
                                    },
                                    totalFreightDiscounts: 0,
                                    surcharges: [...]
                                }
                            }
                        ]
                    },
                    {
                        rateType: "LIST",
                        // ... similar structure
                    }
                ],

                operationalDetail: {
                    originLocationIds: ["DXBA"],
                    commitDays: ["2"],
                    serviceCode: "70",
                    airportId: "DXB",
                    scac: "FDXE",
                    originServiceAreas: ["A1"],
                    deliveryDay: "FRI",
                    deliveryDate: "2025-10-27",
                    deliveryDateType: "ACTUAL",
                    deliveryTime: "18:00:00",
                    ineligibleForMoneyBackGuarantee: false,
                    maximumTransitTime: "TWO_DAYS",
                    transitTime: "TWO_DAYS",
                    packagingCode: "01"
                }
            },
            {
                serviceType: "INTERNATIONAL_ECONOMY",
                // ... another service option
            }
        ],

        alerts: [
            {
                code: "RATE.ALERT.CODE",
                message: "Rate details message",
                alertType: "WARNING"
            }
        ]
    }
}
```

---

### Rate API Comparison Summary

| Feature | DHL Express | FedEx |
|---------|-------------|-------|
| **HTTP Method** | GET (single) / POST (multi) | POST only |
| **Authentication** | Basic Auth | Bearer Token (OAuth) |
| **Account Param** | Query/Body field | Body object with `value` property |
| **Address Format** | Separate city/postalCode fields | Combined `address` object |
| **Street Address** | Not required for rating | Not required for rating |
| **State/Province** | Not required | Required for US/CA/MX |
| **Dimensions Required** | Yes (always) | No (optional for rating) |
| **Weight Units** | String `"metric"/"imperial"` | Object with `units: "KG"/"LB"` |
| **Dimension Units** | Same as weight | Separate `units: "CM"/"IN"` |
| **Date Format** | `YYYY-MM-DD` (GET) or ISO with TZ (POST) | `YYYY-MM-DD` only |
| **Multiple Packages** | Array of `packages` | Array of `requestedPackageLineItems` |
| **Service Filter** | `productCode` (optional) | `serviceType` (optional) |
| **Response Services** | Array of `products` | Array of `rateReplyDetails` |
| **Price Location** | `totalPrice[0].price` | `ratedShipmentDetails[0].totalNetCharge` |
| **Currency** | `priceCurrency` at price level | `currency` at rate detail level |
| **Delivery Date** | `deliveryCapabilities.estimatedDeliveryDateAndTime` | `operationalDetail.deliveryDate` + `deliveryTime` |
| **Volumetric Weight** | `weight.volumetric` | `ratedPackages[].packageRateDetail.billingWeight` |
| **Customs** | Simple boolean flag | Detailed `customsClearanceDetail` |

---

## Shipment Booking APIs

### DHL Express - Shipments API

#### Endpoint

**Method**: **POST `/shipments`**
**Base URL**: `https://express.api.dhl.com/mydhlapi`
**Test URL**: `https://express.api.dhl.com/mydhlapi/test`

---

#### DHL POST /shipments Request

```javascript
{
    // === PRODUCT & ACCOUNT ===
    productCode: "P",  // Required: Product type

    accounts: [{
        typeCode: "shipper",
        number: "454466892"
    }],

    // === PLANNED SHIPPING ===
    plannedShippingDateAndTime: "2025-10-25T14:30:00 GMT+04:00",  // Required

    // === PICKUP ===
    pickup: {
        isRequested: true,
        location: "Reception Area",  // Optional
        specialInstructions: [{
            value: "Call before arrival",  // Max 20 chars
            typeCode: "TBD"
        }]
    },

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

    // === CUSTOMER REFERENCES ===
    customerReferences: [{
        value: "JO_670000000000000000000001",
        typeCode: "CU"  // Customer reference
    }],

    // === PARTIES ===
    customerDetails: {
        shipperDetails: {
            postalAddress: {
                postalCode: "12345",  // Required
                cityName: "Dubai",  // Required, max 45 chars
                countryCode: "AE",  // Required, ISO 2-char
                addressLine1: "Building 123, Street 45",  // Required, max 45 chars
                addressLine2: "Near City Center",  // Optional, max 45 chars
                addressLine3: "Business Bay Area"  // Optional, max 45 chars
            },
            contactInformation: {
                companyName: "ABC Trading LLC",  // Required
                fullName: "John Doe",  // Required
                email: "john@abc.com",  // Optional
                phone: "+971501234567"  // Required
            }
        },
        receiverDetails: {
            postalAddress: {
                postalCode: "10001",
                cityName: "New York",
                countryCode: "US",
                addressLine1: "456 Madison Avenue",
                addressLine2: "Suite 200",
                addressLine3: ""
            },
            contactInformation: {
                companyName: "XYZ Corporation",
                fullName: "Jane Smith",
                email: "jane@xyz.com",
                phone: "+12125551234"
            }
        }
    },

    // === CONTENT ===
    content: {
        packages: [
            {
                weight: 5.0,  // Required
                dimensions: {
                    width: 20,  // Required
                    height: 10,  // Required
                    length: 30  // Required
                }
            },
            {
                weight: 3.5,
                dimensions: {
                    width: 15,
                    height: 8,
                    length: 25
                }
            }
        ],

        description: "Electronic Components",  // Required, max 70 chars

        isCustomsDeclarable: true,  // Required
        declaredValue: 500,  // Required if customs declarable
        declaredValueCurrency: "AED",  // Required

        unitOfMeasurement: "metric",  // Required

        incoterm: "DAP",  // Required: DAP, DDP, etc.

        // === CUSTOMS DECLARATION ===
        exportDeclaration: {
            lineItems: [
                {
                    number: 1,  // Sequential number
                    description: "Smartphone",  // Required
                    price: 800,  // Required
                    quantity: {
                        value: 2,  // Required
                        unitOfMeasurement: "PCS"  // Required
                    },
                    weight: {
                        grossValue: 1.5,  // Required
                        netValue: 1.2  // Required
                    },
                    manufacturerCountry: "AE"  // Required, ISO 2-char
                }
            ],
            invoice: {
                date: "2025-10-25",  // Required, YYYY-MM-DD
                number: "INV-2025-001234"  // Required
            },
            exportReason: "commercial",  // Fixed value
            exportReasonType: "commercial_purpose_or_sale"  // Required from predefined list
        }
    },

    // === VALUE ADDED SERVICES ===
    valueAddedServices: [
        {
            serviceCode: "DS"  // DS = Duties & Taxes Unpaid, DD = Paid
        }
    ],

    // === DOCUMENT IMAGES (optional) ===
    documentImages: [
        {
            typeCode: "INV",  // Invoice
            imageFormat: "PDF",
            content: "JVBERi0xLjQKJeLjz9MK..."  // Base64 encoded
        }
    ]
}
```

**Required Fields**:
- `productCode`
- `accounts`
- `plannedShippingDateAndTime`
- `customerDetails.{shipperDetails, receiverDetails}`
- `content.packages`
- `content.description`
- `content.isCustomsDeclarable`
- If customs: `exportDeclaration.{lineItems, invoice, exportReasonType}`

---

#### DHL POST /shipments Response

```javascript
{
    shipmentTrackingNumber: "1234567890",  // Main AWB number
    dispatchConfirmationNumber: "PRG999999999999999",

    trackingUrl: "https://www.dhl.com/en/express/tracking.html?AWB=1234567890",
    cancelPickupUrl: "https://api.dhl.com/cancel/...",

    packages: [
        {
            referenceNumber: 1,
            trackingNumber: "JD012345678901234567",
            trackingUrl: "https://www.dhl.com/..."
        },
        {
            referenceNumber: 2,
            trackingNumber: "JD012345678901234568",
            trackingUrl: "https://www.dhl.com/..."
        }
    ],

    documents: [
        {
            imageFormat: "PDF",
            content: "JVBERi0xLjQKJeLjz9MK...",  // Base64
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
        productName: "EXPRESS WORLDWIDE",

        origin: {
            countryCode: "AE",
            postalCode: "12345",
            cityName: "Dubai"
        },
        destination: {
            countryCode: "US",
            postalCode: "10001",
            cityName: "New York"
        }
    },

    warnings: [...]  // Optional warnings
}
```

---

### FedEx - Ship API

#### Endpoint

**Method**: **POST `/ship/v1/shipments`**
**Base URL**: `https://apis.fedex.com`
**Test URL**: `https://apis-sandbox.fedex.com`

---

#### FedEx POST /ship/v1/shipments Request

```javascript
{
    // === ACCOUNT ===
    accountNumber: {
        value: "XXXXXXXXX"  // Required
    },

    // === LABEL OPTIONS ===
    labelResponseOptions: "URL_ONLY",  // Required: "URL_ONLY", "LABEL", "URL_AND_LABEL"

    // === REQUESTED SHIPMENT ===
    requestedShipment: {

        // === SHIPPER ===
        shipper: {
            address: {
                streetLines: [
                    "Building 123, Street 45",  // Max 35 chars per line
                    "Near City Center"  // Max 2 lines
                ],
                city: "Dubai",  // Required, max 50 chars
                stateOrProvinceCode: "",  // Required for US/CA/MX, 2 chars
                postalCode: "12345",  // Required (format varies)
                countryCode: "AE",  // Required, ISO 2-char
                residential: false
            },
            contact: {
                personName: "John Doe",  // Required, max 70 chars
                phoneNumber: "+971501234567",  // Required
                phoneExtension: "",  // Optional
                companyName: "ABC Trading LLC",  // Optional but recommended
                emailAddress: "john@abc.com"  // Optional
            }
        },

        // === RECIPIENTS (Array) ===
        recipients: [
            {
                address: {
                    streetLines: [
                        "456 Madison Avenue",
                        "Suite 200"
                    ],
                    city: "New York",
                    stateOrProvinceCode: "NY",  // Required for US
                    postalCode: "10001",
                    countryCode: "US",
                    residential: false
                },
                contact: {
                    personName: "Jane Smith",
                    phoneNumber: "+12125551234",
                    companyName: "XYZ Corporation",
                    emailAddress: "jane@xyz.com"
                }
            }
        ],

        // === SHIPPING DATE ===
        shipDatestamp: "2025-10-25",  // Required, YYYY-MM-DD

        // === SERVICE TYPE ===
        serviceType: "INTERNATIONAL_PRIORITY",  // Required
        // Options: INTERNATIONAL_PRIORITY, INTERNATIONAL_ECONOMY,
        //          FEDEX_GROUND, FEDEX_EXPRESS_SAVER, etc.

        // === PACKAGING ===
        packagingType: "YOUR_PACKAGING",  // Required
        // Options: YOUR_PACKAGING, FEDEX_BOX, FEDEX_ENVELOPE, FEDEX_PAK

        // === PICKUP ===
        pickupType: "USE_SCHEDULED_PICKUP",  // Required
        // Options: USE_SCHEDULED_PICKUP, CONTACT_FEDEX_TO_SCHEDULE,
        //          DROPOFF_AT_FEDEX_LOCATION

        // === PACKAGES ===
        requestedPackageLineItems: [
            {
                sequenceNumber: 1,  // Required, starts from 1

                weight: {
                    value: 5.0,  // Required
                    units: "KG"  // Required: KG or LB
                },

                dimensions: {
                    length: 30,  // Required
                    width: 20,  // Required
                    height: 10,  // Required
                    units: "CM"  // Required: CM or IN
                },

                groupPackageCount: 1,  // Required for multi-piece

                itemDescription: "Electronic Components",  // Max 50 chars

                customerReferences: [
                    {
                        customerReferenceType: "CUSTOMER_REFERENCE",
                        value: "JO_670000000000000000000001"
                    }
                ]
            },
            {
                sequenceNumber: 2,
                weight: {
                    value: 3.5,
                    units: "KG"
                },
                dimensions: {
                    length: 25,
                    width: 15,
                    height: 8,
                    units: "CM"
                },
                groupPackageCount: 1
            }
        ],

        // === TOTAL WEIGHT (optional but recommended) ===
        totalWeight: 8.5,

        // === LABEL SPECIFICATION ===
        labelSpecification: {
            imageType: "PDF",  // Required: PDF, PNG, ZPLII
            labelStockType: "PAPER_85X11_TOP_HALF_LABEL",  // Required
            labelFormatType: "COMMON2D",  // Optional
            labelOrder: "SHIPPING_LABEL_FIRST"  // Optional
        },

        // === SHIPPING CHARGES PAYMENT ===
        shippingChargesPayment: {
            paymentType: "SENDER",  // Required: SENDER, RECIPIENT, THIRD_PARTY
            payor: {
                responsibleParty: {
                    accountNumber: {
                        value: "XXXXXXXXX"
                    },
                    address: {
                        countryCode: "AE"
                    }
                }
            }
        },

        // === CUSTOMS CLEARANCE ===
        customsClearanceDetail: {
            dutiesPayment: {
                paymentType: "SENDER",  // or "RECIPIENT"
                payor: {
                    responsibleParty: {
                        accountNumber: {
                            value: "XXXXXXXXX"
                        },
                        address: {
                            countryCode: "AE"
                        }
                    }
                }
            },

            documentContent: "DOCUMENTS_ONLY",  // or "NON_DOCUMENTS"

            customsValue: {
                amount: 500,  // Total declared value
                currency: "USD"
            },

            commodities: [
                {
                    description: "Smartphone",  // Max 450 chars
                    quantity: 2,
                    quantityUnits: "PCS",
                    weight: {
                        value: 1.5,
                        units: "KG"
                    },
                    customsValue: {
                        amount: 800,
                        currency: "USD"
                    },
                    unitPrice: {
                        amount: 400,
                        currency: "USD"
                    },
                    countryOfManufacture: "AE",
                    harmonizedCode: "8517120000"  // Optional HS code
                }
            ],

            commercialInvoice: {
                shipmentPurpose: "SOLD",  // Required
                // Options: SOLD, NOT_SOLD, GIFT, REPAIR, SAMPLE

                customerReferences: [{
                    customerReferenceType: "INVOICE_NUMBER",
                    value: "INV-2025-001234"
                }]
            }
        },

        // === SPECIAL SERVICES ===
        shipmentSpecialServices: {
            specialServiceTypes: [
                "DANGEROUS_GOODS",  // If applicable
                "DRY_ICE",  // If applicable
                "SIGNATURE_OPTION"  // If required
            ],

            signatureOptionType: "ADULT"  // Optional: ADULT, DIRECT, INDIRECT, NO_SIGNATURE_REQUIRED
        },

        // === EMAIL NOTIFICATIONS (optional) ===
        emailNotificationDetail: {
            recipients: [
                {
                    emailAddress: "john@abc.com",
                    notificationEventType: ["ON_SHIPMENT", "ON_DELIVERY"],
                    emailNotificationRecipientType: "SHIPPER",
                    notificationFormatType: "HTML",
                    locale: "en_US"
                }
            ]
        }
    }
}
```

**Required Fields**:
- `accountNumber`
- `labelResponseOptions`
- `requestedShipment.shipper.{address, contact}`
- `requestedShipment.recipients[].{address, contact}`
- `requestedShipment.shipDatestamp`
- `requestedShipment.serviceType`
- `requestedShipment.packagingType`
- `requestedShipment.pickupType`
- `requestedShipment.requestedPackageLineItems[].{weight, dimensions}`
- `requestedShipment.labelSpecification`
- `requestedShipment.shippingChargesPayment`
- If international: `customsClearanceDetail`

---

#### FedEx POST /ship/v1/shipments Response

```javascript
{
    transactionId: "624deea6-b709-470c-8c39-4b5511281492",
    customerTransactionId: "unique_transaction_id",

    output: {
        transactionShipments: [
            {
                serviceType: "INTERNATIONAL_PRIORITY",
                serviceName: "FedEx International Priority",

                shipDatestamp: "2025-10-25",

                // === MASTER TRACKING ===
                masterTrackingNumber: "794600000000",

                serviceCategory: "EXPRESS",

                // === SHIPMENT DOCUMENTS ===
                shipmentDocuments: [
                    {
                        contentKey: "AWB123456789",
                        copiesToPrint: 1,
                        contentType: "LABEL",
                        shippingDocumentDisposition: "RETURNED",
                        imageType: "PDF",
                        resolution: 203,

                        // Option 1: Base64 content (if labelResponseOptions = "LABEL")
                        encodedLabel: "JVBERi0xLjQKJeLjz9MK...",

                        // Option 2: URL (if labelResponseOptions = "URL_ONLY")
                        url: "https://www.fedex.com/shipping/awblabels/..."
                    },
                    {
                        contentKey: "COMMERCIAL_INVOICE",
                        contentType: "COMMERCIAL_INVOICE",
                        imageType: "PDF",
                        encodedLabel: "JVBERi0xLjQKJeLjz9MK..."
                    }
                ],

                // === PIECES (Packages) ===
                pieceResponses: [
                    {
                        masterTrackingNumber: "794600000000",
                        trackingNumber: "794600000001",  // Individual package tracking
                        customerReferences: [
                            {
                                customerReferenceType: "CUSTOMER_REFERENCE",
                                value: "JO_670000000000000000000001"
                            }
                        ],

                        packageDocuments: [
                            {
                                contentType: "LABEL",
                                imageType: "PDF",
                                encodedLabel: "JVBERi0xLjQKJeLjz9MK...",
                                url: "https://..."
                            }
                        ],

                        packageSequenceNumber: 1,
                        trackingIdType: "FEDEX"
                    },
                    {
                        trackingNumber: "794600000002",
                        packageSequenceNumber: 2,
                        // ...
                    }
                ],

                // === ALERTS/WARNINGS ===
                alerts: [
                    {
                        code: "SHIP.RECIPIENT.POSTALCODE.CHANGED",
                        message: "Recipient postal code was standardized",
                        alertType: "WARNING"
                    }
                ],

                // === COMPLETED SHIPMENT DETAILS ===
                completedShipmentDetail: {
                    completedPackageDetails: [
                        {
                            sequenceNumber: 1,

                            operationalDetail: {
                                ursaPrefixCode: "XJ",
                                ursaSuffixCode: "DXB",
                                originLocationId: "DXBA",
                                originLocationNumber: 0,
                                originServiceArea: "A1",
                                destinationLocationId: "EWRA",
                                destinationLocationNumber: 0,
                                destinationServiceArea: "C4",
                                deliveryDay: "FRI",
                                commitDate: "2025-10-27",
                                transitTime: "TWO_DAYS",
                                maximumTransitTime: "TWO_DAYS",
                                serviceCode: "70",
                                packagingCode: "01",
                                astraDescription: "INTL PRIORITY"
                            },

                            trackingIds: [
                                {
                                    trackingIdType: "FEDEX",
                                    formId: "0201",
                                    trackingNumber: "794600000001"
                                }
                            ],

                            packageRating: {
                                actualRateType: "PAYOR_ACCOUNT_PACKAGE",
                                effectiveNetDiscount: 0,
                                packageRateDetails: [
                                    {
                                        rateType: "PAYOR_ACCOUNT_PACKAGE",
                                        ratedWeightMethod: "ACTUAL",
                                        baseCharge: 300.00,
                                        netFreight: 300.00,
                                        totalSurcharges: 50.50,
                                        netFedExCharge: 350.50,
                                        totalTaxes: 0,
                                        netCharge: 350.50,
                                        totalRebates: 0,
                                        billingWeight: {
                                            units: "KG",
                                            value: 6.0
                                        },
                                        currency: "USD"
                                    }
                                ]
                            },

                            dryIceWeight: null,
                            signatureOption: "SERVICE_DEFAULT"
                        }
                    ],

                    shipmentRating: {
                        actualRateType: "PAYOR_ACCOUNT_SHIPMENT",
                        shipmentRateDetails: [
                            {
                                rateType: "PAYOR_ACCOUNT_SHIPMENT",
                                rateZone: "E",
                                pricingCode: "ACTUAL",
                                ratedWeightMethod: "ACTUAL",
                                currency: "USD",
                                totalBaseCharge: 300.00,
                                totalNetCharge: 350.50,
                                totalNetChargeWithDutiesAndTaxes: 385.75,
                                shipmentRateDetail: {
                                    rateZone: "E",
                                    dimDivisor: 139,
                                    fuelSurchargePercent: 16.75,
                                    totalSurcharges: 50.50,
                                    totalFreightDiscount: 0,
                                    surcharges: [
                                        {
                                            type: "FUEL",
                                            description: "Fuel Surcharge",
                                            amount: 50.50
                                        }
                                    ]
                                },
                                currency: "USD"
                            }
                        ]
                    }
                }
            }
        ]
    }
}
```

---

### Shipment API Comparison Summary

| Feature | DHL Express | FedEx |
|---------|-------------|-------|
| **HTTP Method** | POST only | POST only |
| **Authentication** | Basic Auth | Bearer Token (OAuth) |
| **Account Number** | In `accounts` array | Object with `value` property |
| **Product/Service** | `productCode` (e.g., "P") | `serviceType` (e.g., "INTERNATIONAL_PRIORITY") |
| **Date Format** | ISO with timezone (`GMT+04:00`) | YYYY-MM-DD only |
| **Address Max Length** | 45 chars per line (3 lines) | 35 chars per line (2 lines) |
| **City Name** | Max 45 chars, no commas | Max 50 chars |
| **State/Province** | Not used | Required for US/CA/MX |
| **Contact Structure** | Flat (fullName, email, phone) | Nested (personName, companyName) |
| **Phone Storage** | Single string | phoneNumber + phoneExtension |
| **Recipients** | Single `receiverDetails` | Array `recipients[]` |
| **Packages Array** | `content.packages[]` | `requestedPackageLineItems[]` |
| **Package Sequence** | Auto (not specified) | Manual (`sequenceNumber`) |
| **Weight Format** | Number + global `unitOfMeasurement` | Object `{value, units}` per package |
| **Dimensions Format** | Object without units | Object `{length, width, height, units}` |
| **Description Location** | `content.description` (global) | Per package `itemDescription` |
| **Customs** | `content.exportDeclaration` | `customsClearanceDetail` |
| **Customs Line Items** | `exportDeclaration.lineItems[]` | `customsClearanceDetail.commodities[]` |
| **Invoice Number** | `invoice.number` | `customerReferences[]` with type |
| **Export Reason** | `exportReasonType` (predefined) | `shipmentPurpose` (predefined) |
| **Incoterm** | `content.incoterm` + `valueAddedServices` | `dutiesPayment.paymentType` |
| **Label Request** | `outputImageProperties` | `labelResponseOptions` + `labelSpecification` |
| **Label in Response** | `documents[]` (always included) | Optional (URL or base64) |
| **Master Tracking** | `shipmentTrackingNumber` | `masterTrackingNumber` |
| **Package Tracking** | `packages[].trackingNumber` | `pieceResponses[].trackingNumber` |
| **Tracking URL** | Included in response | Not included (must construct) |
| **Document Types** | `label`, `invoice`, `waybillDoc` | `LABEL`, `COMMERCIAL_INVOICE` |
| **Special Instructions** | `pickup.specialInstructions[]` | No direct equivalent |
| **Customer Reference** | Top-level `customerReferences[]` | Per-package `customerReferences[]` |

---

## Critical Differences

### 1. Authentication Architecture

**Impact**: High
**Effort**: Medium

**DHL**:
- Static credentials, no expiration
- Simple implementation

**FedEx**:
- Dynamic tokens, 1-hour expiration
- Requires token management service
- Need retry logic for 401 errors

**Action Required**: Build token management layer

---

### 2. Address Format Constraints

**Impact**: High
**Effort**: Low

**DHL**:
```javascript
{
    cityName: "Dubai",  // Max 45 chars, no commas
    addressLine1: "...",  // Max 45 chars
    addressLine2: "...",  // Max 45 chars
    addressLine3: "..."   // Max 45 chars
}
```

**FedEx**:
```javascript
{
    city: "Dubai",  // Max 50 chars
    streetLines: [
        "...",  // Max 35 chars
        "..."   // Max 35 chars (only 2 lines)
    ],
    stateOrProvinceCode: "..."  // Required for US/CA/MX
}
```

**Action Required**:
- Frontend validation updates
- Field truncation logic
- State/province code lookup for US/CA/MX

---

### 3. Weight & Dimension Units

**Impact**: Medium
**Effort**: Low

**DHL**:
```javascript
{
    packages: [{
        weight: 5.0,
        dimensions: { length: 30, width: 20, height: 10 }
    }],
    unitOfMeasurement: "metric"  // Global for all packages
}
```

**FedEx**:
```javascript
{
    requestedPackageLineItems: [{
        weight: { value: 5.0, units: "KG" },
        dimensions: { length: 30, width: 20, height: 10, units: "CM" }
    }]
}
```

**Action Required**: Transform data structure

---

### 4. Customs Declaration Structure

**Impact**: High
**Effort**: High

**DHL** - Simpler structure:
```javascript
{
    exportDeclaration: {
        lineItems: [{
            number: 1,
            description: "Smartphone",
            price: 800,
            quantity: { value: 2, unitOfMeasurement: "PCS" },
            weight: { grossValue: 1.5, netValue: 1.2 },
            manufacturerCountry: "AE"
        }],
        invoice: { date: "2025-10-25", number: "INV-123" },
        exportReasonType: "commercial_purpose_or_sale"
    }
}
```

**FedEx** - More detailed:
```javascript
{
    customsClearanceDetail: {
        dutiesPayment: {
            paymentType: "SENDER",
            payor: {
                responsibleParty: {
                    accountNumber: { value: "XXX" },
                    address: { countryCode: "AE" }
                }
            }
        },
        customsValue: { amount: 500, currency: "USD" },
        commodities: [{
            description: "Smartphone",
            quantity: 2,
            quantityUnits: "PCS",
            weight: { value: 1.5, units: "KG" },
            customsValue: { amount: 800, currency: "USD" },
            unitPrice: { amount: 400, currency: "USD" },
            countryOfManufacture: "AE"
        }],
        commercialInvoice: {
            shipmentPurpose: "SOLD",
            customerReferences: [{
                customerReferenceType: "INVOICE_NUMBER",
                value: "INV-123"
            }]
        }
    }
}
```

**Key Differences**:
- FedEx requires separate `unitPrice` and `customsValue`
- FedEx needs `dutiesPayment` details with account info
- FedEx uses `shipmentPurpose` instead of `exportReasonType`
- No gross/net weight in FedEx

**Action Required**: Complex transformation logic

---

### 5. Label Retrieval Strategy

**Impact**: Medium
**Effort**: Low

**DHL**:
- Labels always included in booking response
- Base64-encoded PDF in `documents[]` array
- Multiple document types: label, invoice, waybillDoc

**FedEx**:
- Choose retrieval method via `labelResponseOptions`:
  - `"LABEL"`: Base64 in response
  - `"URL_ONLY"`: URL to download later
  - `"URL_AND_LABEL"`: Both
- Separate label spec configuration

**Action Required**: Decide on label retrieval strategy (recommend `"LABEL"` for consistency)

---

### 6. State/Province Requirement

**Impact**: Medium
**Effort**: Low

**DHL**: Not required

**FedEx**: **Required** for US, Canada, Mexico addresses

**Action Required**:
- Add state/province field to JobOrder model
- Frontend dropdown/autocomplete
- Validation rules

---

### 7. Date/Time Format

**Impact**: Low
**Effort**: Low

**DHL**: ISO 8601 with timezone
```javascript
"2025-10-25T14:30:00 GMT+04:00"
```

**FedEx**: Date only
```javascript
"2025-10-25"
```

**Action Required**: Strip time component for FedEx

---

### 8. Service Type Mapping

**Impact**: Medium
**Effort**: Medium

**DHL Products**:
- `P` = EXPRESS WORLDWIDE
- `N` = DOMESTIC EXPRESS
- `W` = ECONOMY SELECT
- etc.

**FedEx Services**:
- `INTERNATIONAL_PRIORITY` = Premium express
- `INTERNATIONAL_ECONOMY` = Economy
- `FEDEX_GROUND` = Domestic ground
- etc.

**Action Required**:
- Create service mapping table
- Allow configuration in admin panel
- Default to priority service

---

## Field Mapping Strategy

### JobOrder Model → API Request Mapping

#### Shipper/Receiver Transformation

```javascript
// Our JobOrder Model
{
    shipperDetails: {
        customerDetails: {
            companyName: "ABC Trading LLC",
            fullName: "John Doe",
            email: "john@abc.com",
            phone: ["+971501234567", "+971501234568"]
        },
        locationDetails: {
            addressLine1: "Building 123, Street 45",
            addressLine2: "Near City Center",
            addressLine3: "Business Bay Area",
            cityName: "Dubai",
            countryName: "United Arab Emirates",
            countryCode: "AE",
            postalCode: "12345",
            stateOrProvinceCode: ""  // NEW FIELD FOR FEDEX
        }
    }
}

// Transform to DHL
{
    customerDetails: {
        shipperDetails: {
            postalAddress: {
                postalCode: model.locationDetails.postalCode,
                cityName: model.locationDetails.cityName,
                countryCode: model.locationDetails.countryCode,
                addressLine1: model.locationDetails.addressLine1.substring(0, 45),
                addressLine2: model.locationDetails.addressLine2.substring(0, 45),
                addressLine3: model.locationDetails.addressLine3.substring(0, 45)
            },
            contactInformation: {
                companyName: model.customerDetails.companyName,
                fullName: model.customerDetails.fullName,
                email: model.customerDetails.email,
                phone: model.customerDetails.phone[0]
            }
        }
    }
}

// Transform to FedEx
{
    shipper: {
        address: {
            streetLines: [
                model.locationDetails.addressLine1.substring(0, 35),
                (model.locationDetails.addressLine2 + " " + model.locationDetails.addressLine3).substring(0, 35).trim()
            ].filter(Boolean),
            city: model.locationDetails.cityName.substring(0, 50),
            stateOrProvinceCode: model.locationDetails.stateOrProvinceCode || "",
            postalCode: model.locationDetails.postalCode,
            countryCode: model.locationDetails.countryCode,
            residential: false
        },
        contact: {
            personName: model.customerDetails.fullName.substring(0, 70),
            phoneNumber: model.customerDetails.phone[0],
            companyName: model.customerDetails.companyName,
            emailAddress: model.customerDetails.email
        }
    }
}
```

---

#### Package Transformation

```javascript
// Our JobOrder Model
{
    containerInfo: {
        packaging: [
            {
                weight: 5.0,
                quantity: 2,
                dimensions: {
                    length: 30,
                    width: 20,
                    height: 10
                }
            }
        ]
    }
}

// Transform to DHL (expand by quantity)
{
    content: {
        packages: [
            { weight: 5.0, dimensions: { length: 30, width: 20, height: 10 } },
            { weight: 5.0, dimensions: { length: 30, width: 20, height: 10 } }
        ],
        unitOfMeasurement: "metric"
    }
}

// Transform to FedEx (expand by quantity)
{
    requestedPackageLineItems: [
        {
            sequenceNumber: 1,
            weight: { value: 5.0, units: "KG" },
            dimensions: { length: 30, width: 20, height: 10, units: "CM" },
            groupPackageCount: 1
        },
        {
            sequenceNumber: 2,
            weight: { value: 5.0, units: "KG" },
            dimensions: { length: 30, width: 20, height: 10, units: "CM" },
            groupPackageCount: 1
        }
    ],
    totalPackageCount: 2
}
```

---

#### Customs Transformation

```javascript
// Our JobOrder Model
{
    containerInfo: {
        exportDeclaration: {
            lineItems: [
                {
                    description: "Smartphone",
                    price: 800,
                    quantity: { value: 2, unitOfMeasurement: "PCS" },
                    weight: { grossValue: 1.5, netValue: 1.2 }
                }
            ],
            invoice: {
                date: "2025-10-25",
                number: "INV-123"
            },
            exportReasonType: "commercial_purpose_or_sale"
        },
        declaredValue: 1600,
        declaredValueCurrency: "AED",
        manufacturerCountry: "AE"
    }
}

// Transform to DHL (mostly direct mapping)
{
    content: {
        exportDeclaration: {
            lineItems: model.exportDeclaration.lineItems.map((item, idx) => ({
                number: idx + 1,
                description: item.description.substring(0, 70),
                price: item.price,
                quantity: item.quantity,
                weight: item.weight,
                manufacturerCountry: model.manufacturerCountry
            })),
            invoice: {
                date: model.exportDeclaration.invoice.date,
                number: model.exportDeclaration.invoice.number
            },
            exportReason: "commercial",
            exportReasonType: model.exportDeclaration.exportReasonType
        },
        declaredValue: model.declaredValue,
        declaredValueCurrency: model.declaredValueCurrency
    }
}

// Transform to FedEx (complex restructuring)
{
    customsClearanceDetail: {
        dutiesPayment: {
            paymentType: determinePaymentType(model.incoterm),  // DDP → "SENDER", DAP → "RECIPIENT"
            payor: {
                responsibleParty: {
                    accountNumber: { value: FEDEX_ACCOUNT_NUMBER },
                    address: { countryCode: model.shipperDetails.locationDetails.countryCode }
                }
            }
        },
        customsValue: {
            amount: model.declaredValue,
            currency: model.declaredValueCurrency
        },
        commodities: model.exportDeclaration.lineItems.map(item => ({
            description: item.description.substring(0, 450),
            quantity: item.quantity.value,
            quantityUnits: item.quantity.unitOfMeasurement,
            weight: {
                value: item.weight.netValue || item.weight.grossValue,
                units: "KG"
            },
            customsValue: {
                amount: item.price * item.quantity.value,
                currency: model.declaredValueCurrency
            },
            unitPrice: {
                amount: item.price,
                currency: model.declaredValueCurrency
            },
            countryOfManufacture: model.manufacturerCountry
        })),
        commercialInvoice: {
            shipmentPurpose: mapExportReasonToShipmentPurpose(model.exportDeclaration.exportReasonType),
            customerReferences: [{
                customerReferenceType: "INVOICE_NUMBER",
                value: model.exportDeclaration.invoice.number
            }]
        }
    }
}

// Helper function
function mapExportReasonToShipmentPurpose(exportReasonType) {
    const mapping = {
        "commercial_purpose_or_sale": "SOLD",
        "gift": "GIFT",
        "sample": "SAMPLE",
        "repair": "REPAIR",
        "return": "RETURN"
    };
    return mapping[exportReasonType] || "SOLD";
}
```

---

## Integration Approach

### Phase 1: Infrastructure Setup

**Tasks**:
1. ✅ Create FedEx OAuth token management service
2. ✅ Add FedEx credentials to environment variables
3. ✅ Implement token refresh mechanism
4. ✅ Create carrier configuration management

**Files to Create**:
- `carriers/FedExAuthService.js`
- `carriers/CarrierFactory.js`
- `config/carriers.config.js`

---

### Phase 2: Transformer Layer

**Tasks**:
1. ✅ Create carrier-agnostic transformer interface
2. ✅ Implement DHL transformer (refactor existing)
3. ✅ Implement FedEx transformer
4. ✅ Add unit tests

**Files to Create**:
- `transformers/CarrierTransformer.interface.js`
- `transformers/DHLTransformer.js`
- `transformers/FedExTransformer.js`

---

### Phase 3: API Client Layer

**Tasks**:
1. ✅ Create carrier API interface
2. ✅ Refactor existing DHL API client
3. ✅ Implement FedEx API client
4. ✅ Add error handling and retries

**Files to Create**:
- `carriers/CarrierAPI.interface.js`
- `carriers/DHLAPI.js` (refactor existing)
- `carriers/FedExAPI.js`

---

### Phase 4: Frontend Updates

**Tasks**:
1. ✅ Update JobOrder model schema (add stateOrProvinceCode)
2. ✅ Add state/province dropdown component
3. ✅ Update validation rules (carrier-specific)
4. ✅ Update FetchRatesModal to support multiple carriers

**Files to Update**:
- `JobOrder.model.js`
- `JobOrderReviewModal.jsx`
- `FetchDHLRatesModal.jsx` → Rename to `FetchRatesModal.jsx`

---

### Phase 5: Service Layer Refactoring

**Tasks**:
1. ✅ Update JobOrderRateFetching.service.js to use CarrierFactory
2. ✅ Update JobOrderCRUD.service.js status change workflow
3. ✅ Update HandleAWBAutomationFlow.service.js
4. ✅ Add carrier selection logic

**Pattern**:
```javascript
const carrier = CarrierFactory.getCarrier(jobOrder.agentConfirmedByCustomer);
const rates = await carrier.getRates(jobOrder);
const booking = await carrier.bookShipment(jobOrder);
```

---

### Phase 6: Testing & Validation

**Tasks**:
1. ✅ Test FedEx authentication
2. ✅ Test rate fetching for FedEx
3. ✅ Test shipment booking for FedEx
4. ✅ Compare DHL vs FedEx responses
5. ✅ End-to-end testing
6. ✅ Production smoke tests

---

## Next Document: Field Validation Requirements

The next analysis document should detail:
1. FedEx-specific validation rules (character limits, format requirements)
2. State/province code validation
3. Postal code format validation
4. HS code requirements (if any)
5. Service type availability matrix

---

**END OF API COMPARISON DOCUMENT**
