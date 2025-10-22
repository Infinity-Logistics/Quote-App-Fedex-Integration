# FedEx Integration Implementation Roadmap

**Date**: October 22, 2025
**Estimated Duration**: 3-4 weeks
**Priority**: High

---

## Overview

This document provides a step-by-step implementation plan for integrating FedEx API alongside the existing DHL integration. The approach follows a **carrier abstraction pattern** to enable multi-carrier support.

---

## Architecture Design

### Carrier Abstraction Layer

```
┌─────────────────────────────────────────────────────────────┐
│                     JobOrder Business Logic                  │
│              (JobOrderCRUD, AWBAutomation, etc.)             │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                      CarrierFactory                          │
│         getCarrier(carrierName) → CarrierInterface           │
└───────────────────────────┬─────────────────────────────────┘
                            │
            ┌───────────────┼───────────────┐
            ▼               ▼               ▼
    ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
    │  DHLCarrier  │ │ FedExCarrier │ │  UPSCarrier  │
    │              │ │              │ │   (Future)   │
    └──────┬───────┘ └──────┬───────┘ └──────────────┘
           │                │
           ▼                ▼
    ┌──────────────┐ ┌──────────────┐
    │     DHLAPI   │ │   FedExAPI   │
    └──────┬───────┘ └──────┬───────┘
           │                │
           ▼                ▼
    ┌──────────────┐ ┌──────────────┐
    │  DHL Express │ │  FedEx APIs  │
    │     APIs     │ │              │
    └──────────────┘ └──────────────┘
```

---

## Implementation Phases

### Phase 1: Foundation (Week 1, Days 1-2)

#### 1.1 Create Carrier Interface

**File**: `Backend-Quotation-Mgmt/src/carriers/CarrierInterface.js`

```javascript
/**
 * Abstract interface for carrier implementations
 * All carrier-specific implementations must extend this class
 */
class CarrierInterface {
    constructor(config) {
        if (this.constructor === CarrierInterface) {
            throw new Error("Cannot instantiate abstract class CarrierInterface");
        }
        this.config = config;
    }

    /**
     * Authenticate with carrier API
     * @returns {Promise<Object>} Authentication result
     */
    async authenticate() {
        throw new Error("Method 'authenticate()' must be implemented");
    }

    /**
     * Fetch shipping rates
     * @param {Object} rateRequest - Standardized rate request
     * @returns {Promise<Object>} Standardized rate response
     */
    async getRates(rateRequest) {
        throw new Error("Method 'getRates()' must be implemented");
    }

    /**
     * Book a shipment
     * @param {Object} shipmentRequest - Standardized shipment request
     * @returns {Promise<Object>} Standardized shipment response
     */
    async bookShipment(shipmentRequest) {
        throw new Error("Method 'bookShipment()' must be implemented");
    }

    /**
     * Get shipment tracking details
     * @param {string} trackingNumber - Tracking number
     * @returns {Promise<Object>} Tracking details
     */
    async getTracking(trackingNumber) {
        throw new Error("Method 'getTracking()' must be implemented");
    }

    /**
     * Validate postal code format
     * @param {string} postalCode - Postal code to validate
     * @param {string} countryCode - ISO 2-char country code
     * @returns {Promise<Object>} Validation result
     */
    async validatePostalCode(postalCode, countryCode) {
        throw new Error("Method 'validatePostalCode()' must be implemented");
    }

    /**
     * Get carrier-specific metadata
     * @returns {Object} Carrier metadata
     */
    getMetadata() {
        return {
            name: this.config.name,
            supportedServices: this.config.services || [],
            validationRules: this.config.validations || {}
        };
    }
}

module.exports = CarrierInterface;
```

---

#### 1.2 Create Carrier Configuration

**File**: `Backend-Quotation-Mgmt/src/config/carriers.config.js`

```javascript
const carriers = {
    DHL: {
        name: "DHL Express",
        enabled: true,
        credentials: {
            username: process.env.DHL_USERNAME || "apU7tP7wD4xP8k",
            password: process.env.DHL_PASSWORD || "U!1tO$9wR#9hF@2c",
            accountNumber: process.env.DHL_ACCOUNT_NUMBER || "454466892"
        },
        endpoints: {
            production: {
                base: "https://express.api.dhl.com/mydhlapi",
                rates: "/rates",
                shipments: "/shipments"
            },
            test: {
                base: "https://express.api.dhl.com/mydhlapi/test",
                rates: "/rates",
                shipments: "/shipments"
            }
        },
        validations: {
            specialInstructions: { maxLength: 20 },
            addressLine: { maxLength: 45 },
            cityName: { maxLength: 45, noCommas: true },
            itemDescription: { maxLength: 70 },
            pickupDate: { noWeekends: true },
            dimensions: { required: true }
        },
        services: {
            "P": { code: "P", name: "EXPRESS WORLDWIDE", type: "express" },
            "N": { code: "N", name: "DOMESTIC EXPRESS", type: "domestic" },
            "W": { code: "W", name: "ECONOMY SELECT", type: "economy" }
        }
    },

    FEDEX: {
        name: "FedEx",
        enabled: true,
        credentials: {
            clientId: process.env.FEDEX_CLIENT_ID,
            clientSecret: process.env.FEDEX_CLIENT_SECRET,
            accountNumber: process.env.FEDEX_ACCOUNT_NUMBER
        },
        endpoints: {
            production: {
                base: "https://apis.fedex.com",
                oauth: "/oauth/token",
                rates: "/rate/v1/rates/quotes",
                shipments: "/ship/v1/shipments"
            },
            test: {
                base: "https://apis-sandbox.fedex.com",
                oauth: "/oauth/token",
                rates: "/rate/v1/rates/quotes",
                shipments: "/ship/v1/shipments"
            }
        },
        validations: {
            addressLine: { maxLength: 35 },
            cityName: { maxLength: 50 },
            personName: { maxLength: 70 },
            itemDescription: { maxLength: 50 },
            stateRequired: ["US", "CA", "MX"],
            dimensions: { required: false }
        },
        services: {
            "INTERNATIONAL_PRIORITY": {
                code: "INTERNATIONAL_PRIORITY",
                name: "FedEx International Priority",
                type: "express"
            },
            "INTERNATIONAL_ECONOMY": {
                code: "INTERNATIONAL_ECONOMY",
                name: "FedEx International Economy",
                type: "economy"
            },
            "FEDEX_GROUND": {
                code: "FEDEX_GROUND",
                name: "FedEx Ground",
                type: "ground"
            }
        },
        tokenManagement: {
            expirationBuffer: 300  // Refresh token 5 minutes before expiration
        }
    }
};

function getCarrierConfig(carrierName) {
    const config = carriers[carrierName.toUpperCase()];
    if (!config) {
        throw new Error(`Carrier configuration not found for: ${carrierName}`);
    }
    if (!config.enabled) {
        throw new Error(`Carrier is disabled: ${carrierName}`);
    }
    return config;
}

function getEnvironment() {
    return process.env.NODE_ENV === 'production' ? 'production' : 'test';
}

module.exports = {
    carriers,
    getCarrierConfig,
    getEnvironment
};
```

---

#### 1.3 Create FedEx Authentication Service

**File**: `Backend-Quotation-Mgmt/src/carriers/FedExAuthService.js`

```javascript
const axios = require('axios');
const { getCarrierConfig, getEnvironment } = require('../config/carriers.config');
const logger = require('../utils/logger');

class FedExAuthService {
    constructor() {
        this.config = getCarrierConfig('FEDEX');
        this.env = getEnvironment();
        this.token = null;
        this.tokenExpiration = null;
    }

    /**
     * Get valid access token (fetch new if expired)
     */
    async getAccessToken() {
        if (this.isTokenValid()) {
            return this.token;
        }

        logger.info('[FedExAuth] Token expired or missing, fetching new token');
        return await this.fetchNewToken();
    }

    /**
     * Check if current token is valid
     */
    isTokenValid() {
        if (!this.token || !this.tokenExpiration) {
            return false;
        }

        const now = Date.now();
        const buffer = this.config.tokenManagement.expirationBuffer * 1000;
        return now < (this.tokenExpiration - buffer);
    }

    /**
     * Fetch new OAuth token from FedEx
     */
    async fetchNewToken() {
        try {
            const endpoint = this.config.endpoints[this.env];
            const url = `${endpoint.base}${endpoint.oauth}`;

            const params = new URLSearchParams();
            params.append('grant_type', 'client_credentials');
            params.append('client_id', this.config.credentials.clientId);
            params.append('client_secret', this.config.credentials.clientSecret);

            logger.info(`[FedExAuth] Requesting token from ${url}`);

            const response = await axios.post(url, params.toString(), {
                headers: {
                    'Content-Type': 'application/x-www-form-urlencoded'
                }
            });

            if (!response.data || !response.data.access_token) {
                throw new Error('Invalid token response from FedEx');
            }

            this.token = response.data.access_token;
            this.tokenExpiration = Date.now() + (response.data.expires_in * 1000);

            logger.info(`[FedExAuth] Token acquired successfully, expires in ${response.data.expires_in}s`);

            return this.token;
        } catch (error) {
            logger.error('[FedExAuth] Failed to fetch token:', error.response?.data || error.message);

            if (error.response?.status === 401) {
                throw new Error('FedEx authentication failed: Invalid credentials');
            }

            throw new Error(`FedEx authentication failed: ${error.message}`);
        }
    }

    /**
     * Clear cached token (force refresh on next request)
     */
    clearToken() {
        logger.info('[FedExAuth] Clearing cached token');
        this.token = null;
        this.tokenExpiration = null;
    }
}

// Singleton instance
const fedexAuthService = new FedExAuthService();

module.exports = fedexAuthService;
```

---

### Phase 2: Carrier Implementations (Week 1, Days 3-5)

#### 2.1 Create Carrier Factory

**File**: `Backend-Quotation-Mgmt/src/carriers/CarrierFactory.js`

```javascript
const { getCarrierConfig } = require('../config/carriers.config');
const DHLCarrier = require('./DHLCarrier');
const FedExCarrier = require('./FedExCarrier');

class CarrierFactory {
    static carriers = new Map();

    /**
     * Get carrier instance (singleton per carrier)
     */
    static getCarrier(carrierName) {
        const normalizedName = carrierName.toUpperCase();

        // Return cached instance if exists
        if (this.carriers.has(normalizedName)) {
            return this.carriers.get(normalizedName);
        }

        // Create new instance based on carrier name
        const config = getCarrierConfig(normalizedName);
        let carrier;

        switch (normalizedName) {
            case 'DHL':
                carrier = new DHLCarrier(config);
                break;
            case 'FEDEX':
                carrier = new FedExCarrier(config);
                break;
            default:
                throw new Error(`Unsupported carrier: ${carrierName}`);
        }

        // Cache and return
        this.carriers.set(normalizedName, carrier);
        return carrier;
    }

    /**
     * Clear all cached carrier instances
     */
    static clearCache() {
        this.carriers.clear();
    }

    /**
     * Get all enabled carriers
     */
    static getEnabledCarriers() {
        return ['DHL', 'FEDEX'];  // TODO: Get from config
    }
}

module.exports = CarrierFactory;
```

---

#### 2.2 Refactor DHL Carrier

**File**: `Backend-Quotation-Mgmt/src/carriers/DHLCarrier.js`

```javascript
const CarrierInterface = require('./CarrierInterface');
const axios = require('axios');
const { getEnvironment } = require('../config/carriers.config');
const DHLTransformer = require('../transformers/DHLTransformer');
const logger = require('../utils/logger');
const responseModel = require('../utils/responseModel');

class DHLCarrier extends CarrierInterface {
    constructor(config) {
        super(config);
        this.env = getEnvironment();
        this.transformer = new DHLTransformer();
    }

    /**
     * DHL uses Basic Auth (no separate authentication step)
     */
    async authenticate() {
        return responseModel(true, 'DHL uses Basic Auth (always authenticated)');
    }

    /**
     * Get authorization header
     */
    getAuthHeader() {
        const { username, password } = this.config.credentials;
        const credentials = Buffer.from(`${username}:${password}`).toString('base64');
        return `Basic ${credentials}`;
    }

    /**
     * Fetch shipping rates from DHL
     */
    async getRates(jobOrderData) {
        try {
            const endpoint = this.config.endpoints[this.env];
            const isSinglePackage = this.transformer.isSinglePackage(jobOrderData);

            if (isSinglePackage) {
                return await this.getSinglePackageRate(jobOrderData, endpoint);
            } else {
                return await this.getMultiPackageRate(jobOrderData, endpoint);
            }
        } catch (error) {
            logger.error('[DHLCarrier] Rate fetch failed:', error);
            return responseModel(false, 'Failed to fetch DHL rates', error.response?.data || error.message);
        }
    }

    async getSinglePackageRate(jobOrderData, endpoint) {
        const url = `${endpoint.base}${endpoint.rates}`;
        const params = this.transformer.transformRateRequestSingle(jobOrderData);

        const response = await axios.get(url, {
            params,
            headers: {
                'Authorization': this.getAuthHeader(),
                'Content-Type': 'application/json'
            }
        });

        return responseModel(true, 'Rates fetched successfully',
            this.transformer.transformRateResponse(response.data));
    }

    async getMultiPackageRate(jobOrderData, endpoint) {
        const url = `${endpoint.base}${endpoint.rates}`;
        const body = this.transformer.transformRateRequestMulti(jobOrderData);

        const response = await axios.post(url, body, {
            headers: {
                'Authorization': this.getAuthHeader(),
                'Content-Type': 'application/json'
            }
        });

        return responseModel(true, 'Rates fetched successfully',
            this.transformer.transformRateResponse(response.data));
    }

    /**
     * Book shipment with DHL
     */
    async bookShipment(jobOrderData) {
        try {
            const endpoint = this.config.endpoints[this.env];
            const url = `${endpoint.base}${endpoint.shipments}`;
            const body = this.transformer.transformShipmentRequest(jobOrderData);

            logger.info(`[DHLCarrier] Creating shipment for JobOrder: ${jobOrderData._id}`);

            const response = await axios.post(url, body, {
                headers: {
                    'Authorization': this.getAuthHeader(),
                    'Content-Type': 'application/json'
                }
            });

            logger.info(`[DHLCarrier] Shipment created successfully: ${response.data.shipmentTrackingNumber}`);

            return responseModel(true, 'Shipment created successfully',
                this.transformer.transformShipmentResponse(response.data));
        } catch (error) {
            logger.error('[DHLCarrier] Shipment booking failed:', error);
            return responseModel(false, 'Failed to create DHL shipment', error.response?.data || error.message);
        }
    }

    /**
     * Validate postal code (DHL has predefined formats)
     */
    async validatePostalCode(postalCode, countryCode) {
        // TODO: Implement using dhlConstants.DHL_POSTAL_CODE_FORMATS
        return responseModel(true, 'Postal code validation not implemented for DHL');
    }
}

module.exports = DHLCarrier;
```

---

#### 2.3 Implement FedEx Carrier

**File**: `Backend-Quotation-Mgmt/src/carriers/FedExCarrier.js`

```javascript
const CarrierInterface = require('./CarrierInterface');
const axios = require('axios');
const { getEnvironment } = require('../config/carriers.config');
const fedexAuthService = require('./FedExAuthService');
const FedExTransformer = require('../transformers/FedExTransformer');
const logger = require('../utils/logger');
const responseModel = require('../utils/responseModel');

class FedExCarrier extends CarrierInterface {
    constructor(config) {
        super(config);
        this.env = getEnvironment();
        this.transformer = new FedExTransformer(config);
    }

    /**
     * Authenticate with FedEx (get OAuth token)
     */
    async authenticate() {
        try {
            await fedexAuthService.getAccessToken();
            return responseModel(true, 'FedEx authentication successful');
        } catch (error) {
            return responseModel(false, 'FedEx authentication failed', error.message);
        }
    }

    /**
     * Get authorization header with Bearer token
     */
    async getAuthHeader() {
        const token = await fedexAuthService.getAccessToken();
        return `Bearer ${token}`;
    }

    /**
     * Fetch shipping rates from FedEx
     */
    async getRates(jobOrderData) {
        try {
            const endpoint = this.config.endpoints[this.env];
            const url = `${endpoint.base}${endpoint.rates}`;
            const body = this.transformer.transformRateRequest(jobOrderData);

            logger.info('[FedExCarrier] Fetching rates');

            const response = await axios.post(url, body, {
                headers: {
                    'Authorization': await this.getAuthHeader(),
                    'Content-Type': 'application/json',
                    'x-customer-transaction-id': `RATE_${jobOrderData._id}_${Date.now()}`
                }
            });

            logger.info('[FedExCarrier] Rates fetched successfully');

            return responseModel(true, 'Rates fetched successfully',
                this.transformer.transformRateResponse(response.data));
        } catch (error) {
            logger.error('[FedExCarrier] Rate fetch failed:', error);

            // Handle token expiration
            if (error.response?.status === 401) {
                logger.warn('[FedExCarrier] Token expired, clearing cache');
                fedexAuthService.clearToken();
            }

            return responseModel(false, 'Failed to fetch FedEx rates', error.response?.data || error.message);
        }
    }

    /**
     * Book shipment with FedEx
     */
    async bookShipment(jobOrderData) {
        try {
            const endpoint = this.config.endpoints[this.env];
            const url = `${endpoint.base}${endpoint.shipments}`;
            const body = this.transformer.transformShipmentRequest(jobOrderData);

            logger.info(`[FedExCarrier] Creating shipment for JobOrder: ${jobOrderData._id}`);

            const response = await axios.post(url, body, {
                headers: {
                    'Authorization': await this.getAuthHeader(),
                    'Content-Type': 'application/json',
                    'x-customer-transaction-id': `SHIP_${jobOrderData._id}_${Date.now()}`
                }
            });

            logger.info(`[FedExCarrier] Shipment created successfully: ${response.data.output.transactionShipments[0].masterTrackingNumber}`);

            return responseModel(true, 'Shipment created successfully',
                this.transformer.transformShipmentResponse(response.data));
        } catch (error) {
            logger.error('[FedExCarrier] Shipment booking failed:', error);

            // Handle token expiration
            if (error.response?.status === 401) {
                logger.warn('[FedExCarrier] Token expired, clearing cache');
                fedexAuthService.clearToken();
            }

            return responseModel(false, 'Failed to create FedEx shipment', error.response?.data || error.message);
        }
    }

    /**
     * Validate postal code (FedEx provides validation API)
     */
    async validatePostalCode(postalCode, countryCode) {
        // TODO: Implement using FedEx Address Validation API if needed
        return responseModel(true, 'Postal code validation not implemented for FedEx');
    }
}

module.exports = FedExCarrier;
```

---

### Phase 3: Data Transformers (Week 2, Days 1-3)

#### 3.1 Create Transformer Interface

**File**: `Backend-Quotation-Mgmt/src/transformers/CarrierTransformer.interface.js`

```javascript
/**
 * Abstract transformer interface
 * Handles conversion between JobOrder model and carrier-specific API formats
 */
class CarrierTransformer {
    constructor() {
        if (this.constructor === CarrierTransformer) {
            throw new Error("Cannot instantiate abstract class CarrierTransformer");
        }
    }

    // === RATE REQUESTS ===
    transformRateRequest(jobOrderData) {
        throw new Error("Method 'transformRateRequest()' must be implemented");
    }

    transformRateResponse(carrierResponse) {
        throw new Error("Method 'transformRateResponse()' must be implemented");
    }

    // === SHIPMENT REQUESTS ===
    transformShipmentRequest(jobOrderData) {
        throw new Error("Method 'transformShipmentRequest()' must be implemented");
    }

    transformShipmentResponse(carrierResponse) {
        throw new Error("Method 'transformShipmentResponse()' must be implemented");
    }

    // === HELPERS ===
    isSinglePackage(jobOrderData) {
        const packaging = jobOrderData.containerInfo?.packaging || [];
        return packaging.length === 1 && packaging[0].quantity === 1;
    }
}

module.exports = CarrierTransformer;
```

---

#### 3.2 FedEx Transformer (Detailed Implementation)

**File**: `Backend-Quotation-Mgmt/src/transformers/FedExTransformer.js`

```javascript
const CarrierTransformer = require('./CarrierTransformer.interface');

class FedExTransformer extends CarrierTransformer {
    constructor(config) {
        super();
        this.config = config;
    }

    // === RATE TRANSFORMATION ===

    transformRateRequest(jobOrderData) {
        return {
            accountNumber: {
                value: this.config.credentials.accountNumber
            },
            requestedShipment: {
                shipper: this.transformParty(jobOrderData.shipperDetails),
                recipient: this.transformParty(jobOrderData.receiverDetails),
                shipDateStamp: this.formatDate(jobOrderData.pickupDateAndTime),
                pickupType: "USE_SCHEDULED_PICKUP",
                rateRequestType: ["ACCOUNT", "LIST"],
                requestedPackageLineItems: this.transformPackages(jobOrderData.containerInfo.packaging),
                totalPackageCount: this.getTotalPackageCount(jobOrderData.containerInfo.packaging),
                customsClearanceDetail: jobOrderData.containerInfo.isCustomsDeclarable
                    ? this.transformCustoms(jobOrderData.containerInfo)
                    : undefined
            }
        };
    }

    transformRateResponse(carrierResponse) {
        const rateDetails = carrierResponse.output?.rateReplyDetails || [];

        return {
            products: rateDetails.map(rate => ({
                productName: rate.serviceName,
                productCode: rate.serviceType,
                totalPrice: [{
                    price: rate.ratedShipmentDetails?.[0]?.totalNetCharge || 0,
                    priceCurrency: rate.ratedShipmentDetails?.[0]?.currency || "USD"
                }],
                deliveryCapabilities: {
                    estimatedDeliveryDateAndTime: rate.operationalDetail?.deliveryDate
                        ? `${rate.operationalDetail.deliveryDate}T${rate.operationalDetail.deliveryTime || '23:59:00'}`
                        : null
                },
                // Store raw data for reference
                _raw: rate
            }))
        };
    }

    // === SHIPMENT TRANSFORMATION ===

    transformShipmentRequest(jobOrderData) {
        return {
            accountNumber: {
                value: this.config.credentials.accountNumber
            },
            labelResponseOptions: "LABEL",  // Get base64 labels in response
            requestedShipment: {
                shipper: this.transformParty(jobOrderData.shipperDetails),
                recipients: [this.transformParty(jobOrderData.receiverDetails)],
                shipDatestamp: this.formatDate(jobOrderData.pickupDateAndTime),
                serviceType: this.mapServiceType(jobOrderData.agentConfirmedByCustomer),
                packagingType: "YOUR_PACKAGING",
                pickupType: "USE_SCHEDULED_PICKUP",
                requestedPackageLineItems: this.transformPackages(jobOrderData.containerInfo.packaging, true),
                totalPackageCount: this.getTotalPackageCount(jobOrderData.containerInfo.packaging),
                labelSpecification: {
                    imageType: "PDF",
                    labelStockType: "PAPER_85X11_TOP_HALF_LABEL",
                    labelFormatType: "COMMON2D"
                },
                shippingChargesPayment: {
                    paymentType: "SENDER",
                    payor: {
                        responsibleParty: {
                            accountNumber: {
                                value: this.config.credentials.accountNumber
                            },
                            address: {
                                countryCode: jobOrderData.shipperDetails.locationDetails.countryCode
                            }
                        }
                    }
                },
                customsClearanceDetail: jobOrderData.containerInfo.isCustomsDeclarable
                    ? this.transformCustomsForShipment(jobOrderData)
                    : undefined
            }
        };
    }

    transformShipmentResponse(carrierResponse) {
        const shipment = carrierResponse.output?.transactionShipments?.[0];
        if (!shipment) {
            throw new Error('Invalid FedEx shipment response');
        }

        const masterTracking = shipment.masterTrackingNumber;
        const pieces = shipment.pieceResponses || [];
        const documents = this.extractDocuments(shipment);

        return {
            shipmentTrackingNumber: masterTracking,
            dispatchConfirmationNumber: carrierResponse.transactionId,
            trackingUrl: `https://www.fedex.com/fedextrack/?trknbr=${masterTracking}`,
            packages: pieces.map((piece, idx) => ({
                referenceNumber: idx + 1,
                trackingNumber: piece.trackingNumber
            })),
            documents: documents,
            _raw: shipment
        };
    }

    // === HELPER METHODS ===

    transformParty(partyDetails) {
        return {
            address: {
                streetLines: [
                    partyDetails.locationDetails.addressLine1?.substring(0, 35) || '',
                    (partyDetails.locationDetails.addressLine2 + ' ' + partyDetails.locationDetails.addressLine3)
                        .substring(0, 35).trim()
                ].filter(Boolean),
                city: partyDetails.locationDetails.cityName?.substring(0, 50) || '',
                stateOrProvinceCode: partyDetails.locationDetails.stateOrProvinceCode || '',
                postalCode: partyDetails.locationDetails.postalCode || '',
                countryCode: partyDetails.locationDetails.countryCode || '',
                residential: false
            },
            contact: {
                personName: partyDetails.customerDetails.fullName?.substring(0, 70) || '',
                phoneNumber: partyDetails.customerDetails.phone?.[0] || '',
                companyName: partyDetails.customerDetails.companyName || '',
                emailAddress: partyDetails.customerDetails.email || ''
            }
        };
    }

    transformPackages(packaging, includeSequence = false) {
        let sequenceNumber = 1;
        const packages = [];

        packaging.forEach(pkg => {
            for (let i = 0; i < pkg.quantity; i++) {
                const packageItem = {
                    weight: {
                        value: pkg.weight,
                        units: "KG"
                    },
                    dimensions: {
                        length: pkg.dimensions.length,
                        width: pkg.dimensions.width,
                        height: pkg.dimensions.height,
                        units: "CM"
                    },
                    groupPackageCount: 1
                };

                if (includeSequence) {
                    packageItem.sequenceNumber = sequenceNumber++;
                }

                packages.push(packageItem);
            }
        });

        return packages;
    }

    transformCustomsForShipment(jobOrderData) {
        const { containerInfo } = jobOrderData;

        return {
            dutiesPayment: {
                paymentType: containerInfo.incoterm === "DDP" ? "SENDER" : "RECIPIENT",
                payor: {
                    responsibleParty: {
                        accountNumber: {
                            value: this.config.credentials.accountNumber
                        },
                        address: {
                            countryCode: jobOrderData.shipperDetails.locationDetails.countryCode
                        }
                    }
                }
            },
            customsValue: {
                amount: containerInfo.declaredValue,
                currency: containerInfo.declaredValueCurrency
            },
            commodities: this.transformCommodities(containerInfo.exportDeclaration, containerInfo),
            commercialInvoice: {
                shipmentPurpose: this.mapExportReasonToShipmentPurpose(
                    containerInfo.exportDeclaration?.exportReasonType
                ),
                customerReferences: [{
                    customerReferenceType: "INVOICE_NUMBER",
                    value: containerInfo.exportDeclaration?.invoice?.number || ''
                }]
            }
        };
    }

    transformCommodities(exportDeclaration, containerInfo) {
        if (!exportDeclaration?.lineItems) {
            return [];
        }

        return exportDeclaration.lineItems.map(item => ({
            description: item.description?.substring(0, 450) || '',
            quantity: item.quantity?.value || 1,
            quantityUnits: item.quantity?.unitOfMeasurement || 'PCS',
            weight: {
                value: item.weight?.netValue || item.weight?.grossValue || 1,
                units: "KG"
            },
            customsValue: {
                amount: item.price * (item.quantity?.value || 1),
                currency: containerInfo.declaredValueCurrency
            },
            unitPrice: {
                amount: item.price,
                currency: containerInfo.declaredValueCurrency
            },
            countryOfManufacture: containerInfo.manufacturerCountry || 'AE'
        }));
    }

    extractDocuments(shipment) {
        const documents = [];

        // Extract shipment-level documents
        if (shipment.shipmentDocuments) {
            shipment.shipmentDocuments.forEach(doc => {
                if (doc.encodedLabel) {
                    documents.push({
                        imageFormat: "PDF",
                        content: doc.encodedLabel,
                        typeCode: this.mapDocumentType(doc.contentType)
                    });
                }
            });
        }

        // Extract package-level documents
        if (shipment.pieceResponses) {
            shipment.pieceResponses.forEach(piece => {
                if (piece.packageDocuments) {
                    piece.packageDocuments.forEach(doc => {
                        if (doc.encodedLabel) {
                            documents.push({
                                imageFormat: "PDF",
                                content: doc.encodedLabel,
                                typeCode: this.mapDocumentType(doc.contentType)
                            });
                        }
                    });
                }
            });
        }

        return documents;
    }

    // === UTILITY METHODS ===

    formatDate(dateString) {
        // Convert to YYYY-MM-DD
        const date = new Date(dateString);
        return date.toISOString().split('T')[0];
    }

    getTotalPackageCount(packaging) {
        return packaging.reduce((sum, pkg) => sum + pkg.quantity, 0);
    }

    mapServiceType(agentName) {
        // Default mapping
        return "INTERNATIONAL_PRIORITY";
    }

    mapExportReasonToShipmentPurpose(exportReasonType) {
        const mapping = {
            "commercial_purpose_or_sale": "SOLD",
            "gift": "GIFT",
            "sample": "SAMPLE",
            "repair": "REPAIR",
            "return": "RETURN"
        };
        return mapping[exportReasonType] || "SOLD";
    }

    mapDocumentType(fedexType) {
        const mapping = {
            "LABEL": "label",
            "COMMERCIAL_INVOICE": "invoice"
        };
        return mapping[fedexType] || fedexType.toLowerCase();
    }
}

module.exports = FedExTransformer;
```

---

### Phase 4: Integration with Existing Services (Week 2, Days 4-5 + Week 3, Days 1-2)

#### 4.1 Update JobOrderRateFetching.service.js

Replace direct DHL API calls with CarrierFactory:

```javascript
const CarrierFactory = require('../../carriers/CarrierFactory');

async function getDhlRatesFromAPI(params, isTestServer = false) {
    // DEPRECATED - Use getCarrierRates instead
    const carrier = CarrierFactory.getCarrier('DHL');
    return await carrier.getRates(params);
}

async function getCarrierRates(jobOrderData, carrierName) {
    const carrier = CarrierFactory.getCarrier(carrierName);
    return await carrier.getRates(jobOrderData);
}

async function makeBookingUsingCarrier(jobOrderData, carrierName) {
    const carrier = CarrierFactory.getCarrier(carrierName);
    return await carrier.bookShipment(jobOrderData);
}

module.exports = {
    // Legacy exports (for backward compatibility)
    getDhlRatesFromAPI,
    getDhlRatesForMultiplePackages,
    makeBookingUsingDHL,

    // New carrier-agnostic exports
    getCarrierRates,
    makeBookingUsingCarrier
};
```

#### 4.2 Update JobOrderCRUD.service.js

```javascript
const CarrierFactory = require('../../carriers/CarrierFactory');

async function changeJobOrderStatus(id, statusData, isInternalCall = false) {
    // ... existing code ...

    if (status === "REVIEW_COMPLETED") {
        const carrierName = jobOrder.agentConfirmedByCustomer;
        const carrier = CarrierFactory.getCarrier(carrierName);

        // Step 1: Create AWB
        const awbRes = await handleAWBAutomationFlow.startAWBAutomationFlow(
            jobOrder,
            normalJobOrder,
            quotation,
            changeJobOrderStatus,
            true,
            carrier  // Pass carrier instance
        );

        // ... rest of the flow
    }
}
```

---

### Phase 5: Frontend Updates (Week 3, Days 3-5)

#### 5.1 Update JobOrder Model

Add `stateOrProvinceCode` field:

```javascript
stateOrProvinceCode: {
    type: String,
    required: false,
    trim: true,
    maxlength: 3
}
```

#### 5.2 Update Frontend Validation (JobOrderReviewModal.jsx)

```javascript
const CARRIER_VALIDATIONS = {
    DHL: {
        specialInstructions: 20,
        addressLine: 45,
        cityName: 45,
        itemDescription: 70,
        pickupDate: { noWeekends: true },
        dimensions: { required: true }
    },
    FEDEX: {
        addressLine: 35,
        cityName: 50,
        itemDescription: 50,
        pickupDate: { noWeekends: false },
        dimensions: { required: false },
        stateRequired: ["US", "CA", "MX"]
    }
};

function validateFields(details) {
    const carrier = details.agentConfirmedByCustomer;
    const rules = CARRIER_VALIDATIONS[carrier];

    // Apply carrier-specific validations
    // ...
}
```

---

## Testing Strategy

### Unit Tests
- Transformer tests (field mapping)
- Carrier interface tests
- Auth service tests

### Integration Tests
- Rate fetching (sandbox)
- Shipment booking (sandbox)
- Label retrieval

### End-to-End Tests
- Complete workflow: Quote → Review → Book → ERP sync

---

**END OF IMPLEMENTATION ROADMAP**
