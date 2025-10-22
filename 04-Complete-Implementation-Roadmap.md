# Complete FedEx Integration Implementation Roadmap

**Date**: October 22, 2025
**Objective**: Integrate FedEx API alongside existing DHL integration with carrier-specific UX and unified backend architecture

---

## Table of Contents

1. [Implementation Principles](#implementation-principles)
2. [Phase 1: Backend Infrastructure](#phase-1-backend-infrastructure)
3. [Phase 2: Data Model Updates](#phase-2-data-model-updates)
4. [Phase 3: Transformer Layer](#phase-3-transformer-layer)
5. [Phase 4: Carrier API Clients](#phase-4-carrier-api-clients)
6. [Phase 5: Frontend Configuration](#phase-5-frontend-configuration)
7. [Phase 6: Frontend UI Updates](#phase-6-frontend-ui-updates)
8. [Phase 7: Service Layer Integration](#phase-7-service-layer-integration)
9. [Phase 8: Testing & Validation](#phase-8-testing--validation)
10. [Phase 9: Production Deployment](#phase-9-production-deployment)
11. [Verification Checklist](#verification-checklist)

---

## Implementation Principles

### **Core Rules**:
1. **No Breaking Changes**: Existing DHL functionality must remain 100% operational
2. **Backward Compatibility**: Existing JobOrder records must work without migration
3. **No Assumptions**: Every field mapping verified against API documentation
4. **Carrier Agnostic**: Business logic never directly references DHL or FedEx APIs
5. **Progressive Enhancement**: Features added incrementally, tested at each phase

### **Success Criteria**:
- ✅ DHL workflow operates exactly as before
- ✅ FedEx workflow operates with same functionality as DHL
- ✅ Users see carrier-appropriate terminology
- ✅ All existing JobOrders continue to work
- ✅ No data migration required
- ✅ ERP integration works for both carriers

---

## Phase 1: Backend Infrastructure

### **Objective**: Create foundational carrier abstraction layer

### **1.1 Create Carrier Interface**

**File**: `Backend-Quotation-Mgmt/src/carriers/CarrierInterface.js`

**Purpose**: Abstract base class defining standard carrier operations

**Implementation**:
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
     * @returns {Promise<Object>} { success: boolean, message: string, data?: any }
     */
    async authenticate() {
        throw new Error("Method 'authenticate()' must be implemented");
    }

    /**
     * Fetch shipping rates
     * @param {Object} jobOrderData - Complete JobOrder object from database
     * @returns {Promise<Object>} Standardized response: { success, message, data: { products: [...] } }
     */
    async getRates(jobOrderData) {
        throw new Error("Method 'getRates()' must be implemented");
    }

    /**
     * Book a shipment and generate AWB
     * @param {Object} jobOrderData - Complete JobOrder object from database
     * @returns {Promise<Object>} Standardized response: { success, message, data: { shipmentTrackingNumber, documents, ... } }
     */
    async bookShipment(jobOrderData) {
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

**Verification**:
- [ ] File created at correct path
- [ ] All required methods defined
- [ ] JSDoc comments complete
- [ ] Cannot be instantiated directly (throws error)

---

### **1.2 Create Carrier Configuration**

**File**: `Backend-Quotation-Mgmt/src/config/carriers.config.js`

**Purpose**: Centralized configuration for all carriers (DHL, FedEx, future carriers)

**Implementation**:
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

        // DHL-specific validation rules (used by frontend)
        validations: {
            specialInstructions: {
                maxLength: 20,
                message: "Special instructions must be under 20 characters (DHL requirement)"
            },
            addressLine: {
                maxLength: 45,
                message: "Address line must be under 45 characters (DHL limit)"
            },
            cityName: {
                maxLength: 45,
                noCommas: true,
                message: "City name must be under 45 characters and cannot contain commas (DHL requirement)"
            },
            itemDescription: {
                maxLength: 70,
                message: "Item description must be under 70 characters (DHL limit)"
            },
            pickupDate: {
                noWeekends: true,
                message: "DHL does not offer automated pickup on weekends"
            },
            dimensions: {
                required: true,
                message: "Package dimensions are required for DHL rating"
            },
            stateRequired: []  // DHL doesn't require state/province
        },

        // DHL product/service codes
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

        // FedEx-specific validation rules
        validations: {
            specialInstructions: {
                maxLength: null,  // No limit
                message: null
            },
            addressLine: {
                maxLength: 35,
                message: "Address line must be under 35 characters (FedEx limit)"
            },
            cityName: {
                maxLength: 50,
                noCommas: false,
                message: "City name must be under 50 characters"
            },
            personName: {
                maxLength: 70,
                message: "Person name must be under 70 characters (FedEx limit)"
            },
            itemDescription: {
                maxLength: 50,
                message: "Item description must be under 50 characters (FedEx limit)"
            },
            pickupDate: {
                noWeekends: false,
                message: null
            },
            dimensions: {
                required: false,
                message: null
            },
            stateRequired: ["US", "CA", "MX"]  // State/province required for these countries
        },

        // FedEx service types
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
            },
            "FEDEX_EXPRESS_SAVER": {
                code: "FEDEX_EXPRESS_SAVER",
                name: "FedEx Express Saver",
                type: "express"
            }
        },

        // Token management configuration
        tokenManagement: {
            expirationBuffer: 300  // Refresh token 5 minutes (300 seconds) before expiration
        }
    }
};

/**
 * Get carrier configuration by name
 * @param {string} carrierName - Carrier name (case insensitive)
 * @returns {Object} Carrier configuration
 * @throws {Error} If carrier not found or disabled
 */
function getCarrierConfig(carrierName) {
    const normalizedName = carrierName.toUpperCase();
    const config = carriers[normalizedName];

    if (!config) {
        throw new Error(`Carrier configuration not found for: ${carrierName}`);
    }

    if (!config.enabled) {
        throw new Error(`Carrier is disabled: ${carrierName}`);
    }

    return config;
}

/**
 * Get current environment (production or test)
 * @returns {string} 'production' or 'test'
 */
function getEnvironment() {
    return process.env.NODE_ENV === 'production' ? 'production' : 'test';
}

/**
 * Get all enabled carriers
 * @returns {string[]} Array of enabled carrier names
 */
function getEnabledCarriers() {
    return Object.keys(carriers).filter(name => carriers[name].enabled);
}

module.exports = {
    carriers,
    getCarrierConfig,
    getEnvironment,
    getEnabledCarriers
};
```

**Environment Variables Required** (`.env` file):
```
# DHL Credentials (existing)
DHL_USERNAME=apU7tP7wD4xP8k
DHL_PASSWORD=U!1tO$9wR#9hF@2c
DHL_ACCOUNT_NUMBER=454466892

# FedEx Credentials (new - MUST be provided)
FEDEX_CLIENT_ID=<your_fedex_client_id>
FEDEX_CLIENT_SECRET=<your_fedex_client_secret>
FEDEX_ACCOUNT_NUMBER=<your_fedex_account_number>

# Environment
NODE_ENV=development  # or 'production'
```

**Verification**:
- [ ] File created with all carrier configurations
- [ ] DHL config matches existing implementation
- [ ] FedEx config complete with all required fields
- [ ] Validation rules documented for both carriers
- [ ] Environment variables added to `.env`
- [ ] Helper functions (getCarrierConfig, getEnvironment) working
- [ ] Error handling for missing/disabled carriers

---

### **1.3 Create FedEx Authentication Service**

**File**: `Backend-Quotation-Mgmt/src/carriers/FedExAuthService.js`

**Purpose**: Manage OAuth 2.0 token lifecycle for FedEx API

**Implementation**:
```javascript
const axios = require('axios');
const { getCarrierConfig, getEnvironment } = require('../config/carriers.config');

class FedExAuthService {
    constructor() {
        this.config = getCarrierConfig('FEDEX');
        this.env = getEnvironment();
        this.token = null;
        this.tokenExpiration = null;
    }

    /**
     * Get valid access token (fetch new if expired or missing)
     * @returns {Promise<string>} Valid access token
     */
    async getAccessToken() {
        if (this.isTokenValid()) {
            console.log('[FedExAuth] Using cached token');
            return this.token;
        }

        console.log('[FedExAuth] Token expired or missing, fetching new token');
        return await this.fetchNewToken();
    }

    /**
     * Check if current token is valid
     * @returns {boolean} True if token exists and not expired
     */
    isTokenValid() {
        if (!this.token || !this.tokenExpiration) {
            return false;
        }

        const now = Date.now();
        const buffer = this.config.tokenManagement.expirationBuffer * 1000;  // Convert to milliseconds
        const expiresAt = this.tokenExpiration - buffer;

        return now < expiresAt;
    }

    /**
     * Fetch new OAuth token from FedEx
     * @returns {Promise<string>} New access token
     * @throws {Error} If authentication fails
     */
    async fetchNewToken() {
        try {
            const endpoint = this.config.endpoints[this.env];
            const url = `${endpoint.base}${endpoint.oauth}`;

            // Validate credentials
            if (!this.config.credentials.clientId || !this.config.credentials.clientSecret) {
                throw new Error('FedEx credentials not configured. Please set FEDEX_CLIENT_ID and FEDEX_CLIENT_SECRET in environment variables.');
            }

            // Prepare form data
            const params = new URLSearchParams();
            params.append('grant_type', 'client_credentials');
            params.append('client_id', this.config.credentials.clientId);
            params.append('client_secret', this.config.credentials.clientSecret);

            console.log(`[FedExAuth] Requesting token from ${url}`);

            const response = await axios.post(url, params.toString(), {
                headers: {
                    'Content-Type': 'application/x-www-form-urlencoded'
                },
                timeout: 10000  // 10 second timeout
            });

            // Validate response
            if (!response.data || !response.data.access_token) {
                throw new Error('Invalid token response from FedEx (missing access_token)');
            }

            // Store token and expiration
            this.token = response.data.access_token;
            this.tokenExpiration = Date.now() + (response.data.expires_in * 1000);

            console.log(`[FedExAuth] Token acquired successfully, expires in ${response.data.expires_in} seconds`);
            console.log(`[FedExAuth] Token will be refreshed at ${new Date(this.tokenExpiration - (this.config.tokenManagement.expirationBuffer * 1000)).toISOString()}`);

            return this.token;

        } catch (error) {
            console.error('[FedExAuth] Failed to fetch token:', error.response?.data || error.message);

            // Handle specific error cases
            if (error.response?.status === 401) {
                throw new Error('FedEx authentication failed: Invalid credentials (401). Please verify FEDEX_CLIENT_ID and FEDEX_CLIENT_SECRET.');
            }

            if (error.response?.status === 403) {
                throw new Error('FedEx authentication failed: Access forbidden (403). Please check account permissions.');
            }

            if (error.code === 'ECONNABORTED') {
                throw new Error('FedEx authentication failed: Request timeout. Please check network connectivity.');
            }

            throw new Error(`FedEx authentication failed: ${error.message}`);
        }
    }

    /**
     * Clear cached token (force refresh on next request)
     * Used when receiving 401 errors from API calls
     */
    clearToken() {
        console.log('[FedExAuth] Clearing cached token');
        this.token = null;
        this.tokenExpiration = null;
    }

    /**
     * Get token info for debugging
     * @returns {Object} Token status information
     */
    getTokenInfo() {
        return {
            hasToken: !!this.token,
            isValid: this.isTokenValid(),
            expiresAt: this.tokenExpiration ? new Date(this.tokenExpiration).toISOString() : null,
            willRefreshAt: this.tokenExpiration
                ? new Date(this.tokenExpiration - (this.config.tokenManagement.expirationBuffer * 1000)).toISOString()
                : null
        };
    }
}

// Singleton instance (one token manager for entire application)
const fedexAuthService = new FedExAuthService();

module.exports = fedexAuthService;
```

**Verification**:
- [ ] File created with complete implementation
- [ ] Token fetching works with test credentials
- [ ] Token expiration calculated correctly
- [ ] Token refresh logic works (5 min buffer)
- [ ] Error handling for invalid credentials
- [ ] Error handling for network issues
- [ ] Singleton pattern (one instance only)
- [ ] Logging shows token lifecycle

**Test**:
```javascript
// Test file: test-fedex-auth.js
const fedexAuthService = require('./src/carriers/FedExAuthService');

async function testAuth() {
    try {
        console.log('Initial token info:', fedexAuthService.getTokenInfo());

        const token1 = await fedexAuthService.getAccessToken();
        console.log('Token acquired:', token1.substring(0, 20) + '...');
        console.log('Token info after fetch:', fedexAuthService.getTokenInfo());

        const token2 = await fedexAuthService.getAccessToken();
        console.log('Should use cached token:', token1 === token2);

        fedexAuthService.clearToken();
        const token3 = await fedexAuthService.getAccessToken();
        console.log('After clear, new token fetched:', token1 !== token3);

    } catch (error) {
        console.error('Auth test failed:', error.message);
    }
}

testAuth();
```

---

### **1.4 Create Carrier Factory**

**File**: `Backend-Quotation-Mgmt/src/carriers/CarrierFactory.js`

**Purpose**: Factory pattern to get carrier instances (singleton per carrier)

**Implementation**:
```javascript
const { getCarrierConfig } = require('../config/carriers.config');
const DHLCarrier = require('./DHLCarrier');
const FedExCarrier = require('./FedExCarrier');

class CarrierFactory {
    // Static map to cache carrier instances
    static carriers = new Map();

    /**
     * Get carrier instance (singleton per carrier)
     * @param {string} carrierName - Carrier name (DHL, FEDEX, etc.)
     * @returns {CarrierInterface} Carrier instance
     * @throws {Error} If carrier not supported
     */
    static getCarrier(carrierName) {
        const normalizedName = carrierName.toUpperCase();

        // Return cached instance if exists
        if (this.carriers.has(normalizedName)) {
            console.log(`[CarrierFactory] Returning cached ${normalizedName} instance`);
            return this.carriers.get(normalizedName);
        }

        console.log(`[CarrierFactory] Creating new ${normalizedName} instance`);

        // Get configuration (validates carrier exists and is enabled)
        const config = getCarrierConfig(normalizedName);
        let carrier;

        // Create carrier instance based on name
        switch (normalizedName) {
            case 'DHL':
                carrier = new DHLCarrier(config);
                break;

            case 'FEDEX':
                carrier = new FedExCarrier(config);
                break;

            default:
                throw new Error(`Unsupported carrier: ${carrierName}. Supported carriers: DHL, FEDEX`);
        }

        // Cache and return
        this.carriers.set(normalizedName, carrier);
        console.log(`[CarrierFactory] ${normalizedName} instance created and cached`);

        return carrier;
    }

    /**
     * Clear all cached carrier instances
     * Useful for testing or forcing re-initialization
     */
    static clearCache() {
        console.log('[CarrierFactory] Clearing all cached carrier instances');
        this.carriers.clear();
    }

    /**
     * Get all enabled carriers
     * @returns {string[]} Array of enabled carrier names
     */
    static getEnabledCarriers() {
        const { getEnabledCarriers } = require('../config/carriers.config');
        return getEnabledCarriers();
    }

    /**
     * Check if carrier is supported and enabled
     * @param {string} carrierName - Carrier name to check
     * @returns {boolean} True if supported and enabled
     */
    static isCarrierSupported(carrierName) {
        try {
            const normalizedName = carrierName.toUpperCase();
            getCarrierConfig(normalizedName);
            return true;
        } catch (error) {
            return false;
        }
    }
}

module.exports = CarrierFactory;
```

**Verification**:
- [ ] File created with complete factory implementation
- [ ] Singleton pattern works (same instance returned)
- [ ] Supports DHL and FedEx
- [ ] Cache clearing works
- [ ] Error handling for unsupported carriers
- [ ] Helper methods (isCarrierSupported, getEnabledCarriers) work

**Test**:
```javascript
// Test file: test-carrier-factory.js
const CarrierFactory = require('./src/carriers/CarrierFactory');

console.log('Enabled carriers:', CarrierFactory.getEnabledCarriers());

const dhl1 = CarrierFactory.getCarrier('DHL');
const dhl2 = CarrierFactory.getCarrier('dhl');  // Case insensitive
console.log('Same DHL instance:', dhl1 === dhl2);

const fedex1 = CarrierFactory.getCarrier('FEDEX');
const fedex2 = CarrierFactory.getCarrier('fedex');
console.log('Same FedEx instance:', fedex1 === fedex2);

console.log('Is DHL supported:', CarrierFactory.isCarrierSupported('DHL'));
console.log('Is UPS supported:', CarrierFactory.isCarrierSupported('UPS'));

CarrierFactory.clearCache();
const dhl3 = CarrierFactory.getCarrier('DHL');
console.log('After clear, new instance:', dhl1 !== dhl3);
```

---

### **Phase 1 Completion Checklist**:
- [ ] `CarrierInterface.js` created and verified
- [ ] `carriers.config.js` created with DHL and FedEx configs
- [ ] Environment variables added to `.env`
- [ ] `FedExAuthService.js` created and token fetching works
- [ ] `CarrierFactory.js` created and tested
- [ ] All test scripts pass
- [ ] No errors in console logs

---

## Phase 2: Data Model Updates

### **Objective**: Add new fields to JobOrder model for carrier-agnostic storage

### **2.1 Update JobOrder Model Schema**

**File**: `Backend-Quotation-Mgmt/src/logic/jobOrder/model/JobOrder.model.js`

**Changes Required**:

#### **Change 1: Add State/Province Code to Location Details**

**Location**: In both `shipperDetails.locationDetails` and `receiverDetails.locationDetails` schemas

**Add After** `postalCode` field:
```javascript
stateOrProvinceCode: {
    type: String,
    required: false,
    trim: true,
    maxlength: 3,
    default: ""
}
```

**Purpose**: Required by FedEx for US, Canada, Mexico addresses

#### **Change 2: Add Carrier-Agnostic Customs Fields**

**Location**: In `containerInfo` schema

**Add After** `incoterm` field:
```javascript
// Carrier-agnostic field derived from incoterm
// "SENDER" = shipper pays duties (DDP)
// "RECIPIENT" = receiver pays duties (DAP, DDU, etc.)
// "THIRD_PARTY" = third party pays (future use)
dutiesPaymentType: {
    type: String,
    required: false,
    enum: ["SENDER", "RECIPIENT", "THIRD_PARTY"],
    default: function() {
        // Auto-calculate from incoterm if not provided
        return this.incoterm === "DDP" ? "SENDER" : "RECIPIENT";
    }
}
```

**Add Within** `exportDeclaration` schema (after `exportReasonType`):
```javascript
// Carrier-agnostic field for shipment purpose
// Maps to FedEx shipmentPurpose, derived from exportReasonType for DHL
shipmentPurpose: {
    type: String,
    required: false,
    enum: ["SOLD", "GIFT", "SAMPLE", "REPAIR", "RETURN", "PERSONAL_EFFECTS", "NOT_SOLD"],
    default: function() {
        // Auto-calculate from exportReasonType if not provided
        const mapping = {
            "commercial_purpose_or_sale": "SOLD",
            "gift": "GIFT",
            "sample": "SAMPLE",
            "repair": "REPAIR",
            "return": "RETURN"
        };
        return mapping[this.exportReasonType] || "SOLD";
    }
}
```

**Complete Updated Schema**:
```javascript
// Shipper Details Schema
const shipperDetailsSchema = new mongoose.Schema({
    customerDetails: {
        companyName: { type: String, required: false },
        fullName: { type: String, required: false },
        email: { type: String, required: false },
        phone: { type: [String], required: false }
    },
    locationDetails: {
        addressLine1: { type: String, required: false },
        addressLine2: { type: String, required: false },
        addressLine3: { type: String, required: false },
        cityName: { type: String, required: false },
        countryName: { type: String, required: false },
        countryCode: { type: String, required: false },
        postalCode: { type: String, required: false },
        stateOrProvinceCode: { type: String, required: false, trim: true, maxlength: 3, default: "" }  // NEW
    }
}, { _id: false });

// Receiver Details Schema (same structure)
const receiverDetailsSchema = new mongoose.Schema({
    customerDetails: {
        companyName: { type: String, required: false },
        fullName: { type: String, required: false },
        email: { type: String, required: false },
        phone: { type: [String], required: false }
    },
    locationDetails: {
        addressLine1: { type: String, required: false },
        addressLine2: { type: String, required: false },
        addressLine3: { type: String, required: false },
        cityName: { type: String, required: false },
        countryName: { type: String, required: false },
        countryCode: { type: String, required: false },
        postalCode: { type: String, required: false },
        stateOrProvinceCode: { type: String, required: false, trim: true, maxlength: 3, default: "" }  // NEW
    }
}, { _id: false });

// Container Info Schema (relevant parts)
const containerInfoSchema = new mongoose.Schema({
    // ... existing fields ...
    incoterm: { type: String, required: false },
    dutiesPaymentType: {  // NEW
        type: String,
        required: false,
        enum: ["SENDER", "RECIPIENT", "THIRD_PARTY"],
        default: function() {
            return this.incoterm === "DDP" ? "SENDER" : "RECIPIENT";
        }
    },
    // ... other fields ...
    exportDeclaration: {
        lineItems: [lineItemSchema],
        invoice: {
            date: { type: String, required: false },
            number: { type: String, required: false }
        },
        exportReasonType: { type: String, required: false },
        shipmentPurpose: {  // NEW
            type: String,
            required: false,
            enum: ["SOLD", "GIFT", "SAMPLE", "REPAIR", "RETURN", "PERSONAL_EFFECTS", "NOT_SOLD"],
            default: function() {
                const mapping = {
                    "commercial_purpose_or_sale": "SOLD",
                    "gift": "GIFT",
                    "sample": "SAMPLE",
                    "repair": "REPAIR",
                    "return": "RETURN"
                };
                return mapping[this.exportReasonType] || "SOLD";
            }
        }
    }
    // ... other fields ...
}, { _id: false });
```

### **2.2 Add Pre-Save Hook for Auto-Calculation**

**Location**: After schema definitions, before `module.exports`

**Add**:
```javascript
/**
 * Pre-save hook to auto-calculate carrier-agnostic fields
 * Ensures backward compatibility with existing records
 */
JobOrderSchema.pre('save', function(next) {
    try {
        // Auto-calculate dutiesPaymentType from incoterm if not set
        if (this.containerInfo && this.containerInfo.incoterm && !this.containerInfo.dutiesPaymentType) {
            this.containerInfo.dutiesPaymentType =
                this.containerInfo.incoterm === 'DDP' ? 'SENDER' : 'RECIPIENT';
            console.log(`[JobOrder] Auto-calculated dutiesPaymentType: ${this.containerInfo.dutiesPaymentType} from incoterm: ${this.containerInfo.incoterm}`);
        }

        // Auto-calculate shipmentPurpose from exportReasonType if not set
        if (this.containerInfo?.exportDeclaration?.exportReasonType &&
            !this.containerInfo.exportDeclaration.shipmentPurpose) {

            const mapping = {
                "commercial_purpose_or_sale": "SOLD",
                "gift": "GIFT",
                "sample": "SAMPLE",
                "repair": "REPAIR",
                "return": "RETURN"
            };

            this.containerInfo.exportDeclaration.shipmentPurpose =
                mapping[this.containerInfo.exportDeclaration.exportReasonType] || "SOLD";

            console.log(`[JobOrder] Auto-calculated shipmentPurpose: ${this.containerInfo.exportDeclaration.shipmentPurpose} from exportReasonType: ${this.containerInfo.exportDeclaration.exportReasonType}`);
        }

        next();
    } catch (error) {
        console.error('[JobOrder] Error in pre-save hook:', error);
        next(error);
    }
});
```

### **2.3 Add Helper Methods to Schema**

**Location**: Before `module.exports`

**Add**:
```javascript
/**
 * Instance method to get carrier-agnostic duties payment info
 * @returns {string} "SENDER", "RECIPIENT", or "THIRD_PARTY"
 */
JobOrderSchema.methods.getDutiesPaymentType = function() {
    if (this.containerInfo?.dutiesPaymentType) {
        return this.containerInfo.dutiesPaymentType;
    }

    // Fallback to incoterm
    if (this.containerInfo?.incoterm) {
        return this.containerInfo.incoterm === 'DDP' ? 'SENDER' : 'RECIPIENT';
    }

    return 'RECIPIENT';  // Default
};

/**
 * Instance method to get carrier-agnostic shipment purpose
 * @returns {string} Shipment purpose code
 */
JobOrderSchema.methods.getShipmentPurpose = function() {
    if (this.containerInfo?.exportDeclaration?.shipmentPurpose) {
        return this.containerInfo.exportDeclaration.shipmentPurpose;
    }

    // Fallback to exportReasonType
    if (this.containerInfo?.exportDeclaration?.exportReasonType) {
        const mapping = {
            "commercial_purpose_or_sale": "SOLD",
            "gift": "GIFT",
            "sample": "SAMPLE",
            "repair": "REPAIR",
            "return": "RETURN"
        };
        return mapping[this.containerInfo.exportDeclaration.exportReasonType] || "SOLD";
    }

    return "SOLD";  // Default
};

/**
 * Instance method to check if state/province is required
 * @param {string} countryCode - ISO 2-char country code
 * @param {string} carrierName - Carrier name (DHL, FEDEX)
 * @returns {boolean} True if required
 */
JobOrderSchema.methods.isStateRequired = function(countryCode, carrierName) {
    const { getCarrierConfig } = require('../../config/carriers.config');

    try {
        const config = getCarrierConfig(carrierName);
        const stateRequiredCountries = config.validations?.stateRequired || [];
        return stateRequiredCountries.includes(countryCode);
    } catch (error) {
        return false;
    }
};
```

**Verification**:
- [ ] Schema updated with all new fields
- [ ] Default values configured correctly
- [ ] Pre-save hook implemented
- [ ] Helper methods added
- [ ] Model still loads without errors
- [ ] Existing JobOrders load correctly (backward compatible)

**Test**:
```javascript
// Test file: test-joborder-model.js
const JobOrder = require('./src/logic/jobOrder/model/JobOrder.model');

// Test 1: Existing record compatibility
const existingRecord = {
    containerInfo: {
        incoterm: "DAP",
        exportDeclaration: {
            exportReasonType: "commercial_purpose_or_sale"
        }
    }
};

const jobOrder = new JobOrder(existingRecord);
console.log('Duties payment type:', jobOrder.getDutiesPaymentType());  // Should be "RECIPIENT"
console.log('Shipment purpose:', jobOrder.getShipmentPurpose());  // Should be "SOLD"

// Test 2: Pre-save hook
jobOrder.save().then(() => {
    console.log('After save - dutiesPaymentType:', jobOrder.containerInfo.dutiesPaymentType);
    console.log('After save - shipmentPurpose:', jobOrder.containerInfo.exportDeclaration.shipmentPurpose);
});
```

---

### **Phase 2 Completion Checklist**:
- [ ] JobOrder model updated with new fields
- [ ] Pre-save hook implemented and tested
- [ ] Helper methods added and tested
- [ ] Existing records load without errors
- [ ] Auto-calculation works for new records
- [ ] No breaking changes to existing code

---

## Phase 3: Transformer Layer

### **Objective**: Create data transformation logic to map JobOrder model to carrier-specific API formats

### **3.1 Create Transformer Interface**

**File**: `Backend-Quotation-Mgmt/src/transformers/CarrierTransformer.interface.js`

**Implementation**:
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

    // === RATE TRANSFORMATION ===

    /**
     * Transform JobOrder data to carrier rate request format
     * @param {Object} jobOrderData - Complete JobOrder object
     * @returns {Object} Carrier-specific rate request
     */
    transformRateRequest(jobOrderData) {
        throw new Error("Method 'transformRateRequest()' must be implemented");
    }

    /**
     * Transform carrier rate response to standardized format
     * @param {Object} carrierResponse - Raw carrier API response
     * @returns {Object} Standardized rate response
     */
    transformRateResponse(carrierResponse) {
        throw new Error("Method 'transformRateResponse()' must be implemented");
    }

    // === SHIPMENT TRANSFORMATION ===

    /**
     * Transform JobOrder data to carrier shipment request format
     * @param {Object} jobOrderData - Complete JobOrder object
     * @returns {Object} Carrier-specific shipment request
     */
    transformShipmentRequest(jobOrderData) {
        throw new Error("Method 'transformShipmentRequest()' must be implemented");
    }

    /**
     * Transform carrier shipment response to standardized format
     * @param {Object} carrierResponse - Raw carrier API response
     * @returns {Object} Standardized shipment response
     */
    transformShipmentResponse(carrierResponse) {
        throw new Error("Method 'transformShipmentResponse()' must be implemented");
    }

    // === HELPER METHODS ===

    /**
     * Check if JobOrder has single package
     * @param {Object} jobOrderData - JobOrder object
     * @returns {boolean} True if single package
     */
    isSinglePackage(jobOrderData) {
        const packaging = jobOrderData.containerInfo?.packaging || [];
        return packaging.length === 1 && packaging[0].quantity === 1;
    }

    /**
     * Get total package count
     * @param {Object} packaging - Packaging array from JobOrder
     * @returns {number} Total package count
     */
    getTotalPackageCount(packaging) {
        return packaging.reduce((sum, pkg) => sum + (pkg.quantity || 1), 0);
    }

    /**
     * Format date to YYYY-MM-DD
     * @param {string|Date} dateInput - Date to format
     * @returns {string} Formatted date
     */
    formatDate(dateInput) {
        const date = dateInput instanceof Date ? dateInput : new Date(dateInput);
        const year = date.getFullYear();
        const month = String(date.getMonth() + 1).padStart(2, '0');
        const day = String(date.getDate()).padStart(2, '0');
        return `${year}-${month}-${day}`;
    }
}

module.exports = CarrierTransformer;
```

**Verification**:
- [ ] Interface created with all required methods
- [ ] Helper methods implemented
- [ ] Cannot be instantiated directly

---

### **3.2 Create DHL Transformer** (Refactor existing logic)

**File**: `Backend-Quotation-Mgmt/src/transformers/DHLTransformer.js`

**Implementation**: See separate detailed implementation in next section due to length

**Key Methods**:
1. `transformRateRequest()` - Maps to DHL GET/POST /rates format
2. `transformRateResponse()` - Standardizes DHL rate response
3. `transformShipmentRequest()` - Maps to DHL POST /shipments format
4. `transformShipmentResponse()` - Standardizes DHL shipment response
5. Helper methods for address, packages, customs

**Verification**:
- [ ] All existing DHL API calls work through transformer
- [ ] Rate fetching produces same results as before
- [ ] Shipment booking produces same results as before
- [ ] No regression in DHL functionality

---

### **3.3 Create FedEx Transformer**

**File**: `Backend-Quotation-Mgmt/src/transformers/FedExTransformer.js`

**Implementation**: Full transformer with all field mappings

(See detailed implementation below)

**Verification**:
- [ ] All JobOrder fields mapped to FedEx API format
- [ ] Customs transformation handles all fields
- [ ] Address transformation handles street line limits
- [ ] Package transformation expands quantities correctly
- [ ] State/province included for US/CA/MX
- [ ] Documents extracted from response correctly

---

### **Phase 3 Implementation Details**

Due to length, detailed transformer implementations are provided in separate sections below.

---

## Phase 4: Carrier API Clients

### **Objective**: Implement carrier-specific API clients that use transformers

### **4.1 Refactor DHL Carrier**

**File**: `Backend-Quotation-Mgmt/src/carriers/DHLCarrier.js`

**Purpose**: Refactor existing DHL API calls into CarrierInterface implementation

**Implementation**:
```javascript
const CarrierInterface = require('./CarrierInterface');
const axios = require('axios');
const { getEnvironment } = require('../config/carriers.config');
const DHLTransformer = require('../transformers/DHLTransformer');
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
     * Get Basic Auth header
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
            console.error('[DHLCarrier] Rate fetch failed:', error);
            return responseModel(false, 'Failed to fetch DHL rates', error.response?.data || error.message);
        }
    }

    async getSinglePackageRate(jobOrderData, endpoint) {
        const url = `${endpoint.base}${endpoint.rates}`;
        const params = this.transformer.transformRateRequestSingle(jobOrderData);

        console.log(`[DHLCarrier] Fetching single package rate from ${url}`);

        const response = await axios.get(url, {
            params,
            headers: {
                'Authorization': this.getAuthHeader(),
                'Content-Type': 'application/json'
            },
            timeout: 30000
        });

        return responseModel(true, 'Rates fetched successfully',
            this.transformer.transformRateResponse(response.data));
    }

    async getMultiPackageRate(jobOrderData, endpoint) {
        const url = `${endpoint.base}${endpoint.rates}`;
        const body = this.transformer.transformRateRequestMulti(jobOrderData);

        console.log(`[DHLCarrier] Fetching multi-package rate from ${url}`);

        const response = await axios.post(url, body, {
            headers: {
                'Authorization': this.getAuthHeader(),
                'Content-Type': 'application/json'
            },
            timeout: 30000
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

            console.log(`[DHLCarrier] Creating shipment for JobOrder: ${jobOrderData._id}`);
            console.log(`[DHLCarrier] Request body:`, JSON.stringify(body, null, 2));

            const response = await axios.post(url, body, {
                headers: {
                    'Authorization': this.getAuthHeader(),
                    'Content-Type': 'application/json'
                },
                timeout: 60000  // 60 second timeout for shipment creation
            });

            console.log(`[DHLCarrier] Shipment created successfully: ${response.data.shipmentTrackingNumber}`);

            return responseModel(true, 'Shipment created successfully',
                this.transformer.transformShipmentResponse(response.data));
        } catch (error) {
            console.error('[DHLCarrier] Shipment booking failed:', error.response?.data || error.message);
            return responseModel(false, 'Failed to create DHL shipment', error.response?.data || error.message);
        }
    }

    /**
     * Validate postal code (DHL has predefined formats)
     */
    async validatePostalCode(postalCode, countryCode) {
        // TODO: Implement using DHL_POSTAL_CODE_FORMATS from dhlConstants
        return responseModel(true, 'Postal code validation not yet implemented for DHL');
    }

    /**
     * Get tracking info
     */
    async getTracking(trackingNumber) {
        // TODO: Implement if needed
        return responseModel(false, 'Tracking not yet implemented for DHL');
    }
}

module.exports = DHLCarrier;
```

**Verification**:
- [ ] DHL carrier extends CarrierInterface
- [ ] Uses DHLTransformer for all transformations
- [ ] Rate fetching works (GET and POST methods)
- [ ] Shipment booking works
- [ ] Error handling comprehensive
- [ ] Logging shows request/response details
- [ ] No regression from existing functionality

---

### **4.2 Implement FedEx Carrier**

**File**: `Backend-Quotation-Mgmt/src/carriers/FedExCarrier.js`

**Purpose**: Complete FedEx API integration using FedExTransformer

**Implementation**:
```javascript
const CarrierInterface = require('./CarrierInterface');
const axios = require('axios');
const { getEnvironment } = require('../config/carriers.config');
const fedexAuthService = require('./FedExAuthService');
const FedExTransformer = require('../transformers/FedExTransformer');
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

            console.log(`[FedExCarrier] Fetching rates for JobOrder: ${jobOrderData._id}`);
            console.log(`[FedExCarrier] Request body:`, JSON.stringify(body, null, 2));

            const response = await axios.post(url, body, {
                headers: {
                    'Authorization': await this.getAuthHeader(),
                    'Content-Type': 'application/json',
                    'x-customer-transaction-id': `RATE_${jobOrderData._id}_${Date.now()}`
                },
                timeout: 30000
            });

            console.log(`[FedExCarrier] Rates fetched successfully`);

            return responseModel(true, 'Rates fetched successfully',
                this.transformer.transformRateResponse(response.data));
        } catch (error) {
            console.error('[FedExCarrier] Rate fetch failed:', error.response?.data || error.message);

            // Handle token expiration
            if (error.response?.status === 401) {
                console.warn('[FedExCarrier] Token expired (401), clearing cache and retrying');
                fedexAuthService.clearToken();

                // Retry once with new token
                try {
                    const url = `${this.config.endpoints[this.env].base}${this.config.endpoints[this.env].rates}`;
                    const body = this.transformer.transformRateRequest(jobOrderData);

                    const retryResponse = await axios.post(url, body, {
                        headers: {
                            'Authorization': await this.getAuthHeader(),
                            'Content-Type': 'application/json',
                            'x-customer-transaction-id': `RATE_RETRY_${jobOrderData._id}_${Date.now()}`
                        },
                        timeout: 30000
                    });

                    console.log('[FedExCarrier] Retry successful after token refresh');
                    return responseModel(true, 'Rates fetched successfully',
                        this.transformer.transformRateResponse(retryResponse.data));
                } catch (retryError) {
                    console.error('[FedExCarrier] Retry failed:', retryError);
                    return responseModel(false, 'Failed to fetch FedEx rates after retry', retryError.response?.data || retryError.message);
                }
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

            console.log(`[FedExCarrier] Creating shipment for JobOrder: ${jobOrderData._id}`);
            console.log(`[FedExCarrier] Request body:`, JSON.stringify(body, null, 2));

            const response = await axios.post(url, body, {
                headers: {
                    'Authorization': await this.getAuthHeader(),
                    'Content-Type': 'application/json',
                    'x-customer-transaction-id': `SHIP_${jobOrderData._id}_${Date.now()}`
                },
                timeout: 60000  // 60 second timeout for shipment creation
            });

            const trackingNumber = response.data.output?.transactionShipments?.[0]?.masterTrackingNumber;
            console.log(`[FedExCarrier] Shipment created successfully: ${trackingNumber}`);

            return responseModel(true, 'Shipment created successfully',
                this.transformer.transformShipmentResponse(response.data));
        } catch (error) {
            console.error('[FedExCarrier] Shipment booking failed:', error.response?.data || error.message);

            // Handle token expiration
            if (error.response?.status === 401) {
                console.warn('[FedExCarrier] Token expired (401), clearing cache and retrying');
                fedexAuthService.clearToken();

                // Retry once with new token
                try {
                    const url = `${this.config.endpoints[this.env].base}${this.config.endpoints[this.env].shipments}`;
                    const body = this.transformer.transformShipmentRequest(jobOrderData);

                    const retryResponse = await axios.post(url, body, {
                        headers: {
                            'Authorization': await this.getAuthHeader(),
                            'Content-Type': 'application/json',
                            'x-customer-transaction-id': `SHIP_RETRY_${jobOrderData._id}_${Date.now()}`
                        },
                        timeout: 60000
                    });

                    console.log('[FedExCarrier] Retry successful after token refresh');
                    return responseModel(true, 'Shipment created successfully',
                        this.transformer.transformShipmentResponse(retryResponse.data));
                } catch (retryError) {
                    console.error('[FedExCarrier] Retry failed:', retryError);
                    return responseModel(false, 'Failed to create FedEx shipment after retry', retryError.response?.data || retryError.message);
                }
            }

            return responseModel(false, 'Failed to create FedEx shipment', error.response?.data || error.message);
        }
    }

    /**
     * Validate postal code
     */
    async validatePostalCode(postalCode, countryCode) {
        // TODO: Implement using FedEx Address Validation API if needed
        return responseModel(true, 'Postal code validation not yet implemented for FedEx');
    }

    /**
     * Get tracking info
     */
    async getTracking(trackingNumber) {
        // TODO: Implement if needed
        return responseModel(false, 'Tracking not yet implemented for FedEx');
    }
}

module.exports = FedExCarrier;
```

**Verification**:
- [ ] FedEx carrier extends CarrierInterface
- [ ] Uses FedExTransformer for all transformations
- [ ] OAuth authentication working
- [ ] Token expiration handled with retry logic
- [ ] Rate fetching works
- [ ] Shipment booking works
- [ ] Error handling comprehensive
- [ ] Logging shows request/response details

---

### **Phase 4 Completion Checklist**:
- [ ] DHLCarrier refactored and tested
- [ ] FedExCarrier implemented and tested
- [ ] Both carriers work through CarrierFactory
- [ ] Rate fetching works for both carriers
- [ ] Shipment booking works for both carriers
- [ ] Token management working for FedEx
- [ ] Retry logic tested for 401 errors
- [ ] No regression in DHL functionality

---

## Phase 5: Frontend Configuration

### **Objective**: Create carrier-specific UI configurations for dynamic form rendering

### **5.1 Create Carrier UI Configuration**

**File**: `Frontend-Quotation-Mgmt-App/src/config/carrierUIConfig.js`

**Implementation**:
```javascript
/**
 * Carrier-specific UI configuration
 * Defines labels, options, and validation rules for each carrier
 */

// Incoterm options for DHL
export const INCOTERM_OPTIONS_DHL = [
    { value: "CIP", label: "CIP - Carriage and Insurance Paid to", dutiesPaymentType: "RECIPIENT" },
    { value: "CPT", label: "CPT - Carriage Paid To", dutiesPaymentType: "RECIPIENT" },
    { value: "DAF", label: "DAF - Delivered at Frontier", dutiesPaymentType: "RECIPIENT" },
    { value: "DAP", label: "DAP - Delivered at Place", dutiesPaymentType: "RECIPIENT" },
    { value: "DAT", label: "DAT - Delivered at Terminal", dutiesPaymentType: "RECIPIENT" },
    { value: "DDP", label: "DDP - Delivered Duty Paid", dutiesPaymentType: "SENDER" },
    { value: "DDU", label: "DDU - Delivered Duty Unpaid", dutiesPaymentType: "RECIPIENT" },
    { value: "DPU", label: "DPU - Delivered at Place Unloaded", dutiesPaymentType: "RECIPIENT" },
    { value: "EXW", label: "EXW - Ex Works", dutiesPaymentType: "RECIPIENT" },
    { value: "FCA", label: "FCA - Free Carrier", dutiesPaymentType: "RECIPIENT" }
];

// Export reason options for DHL
export const EXPORT_REASON_OPTIONS_DHL = [
    { value: "commercial_purpose_or_sale", label: "Commercial Purpose or Sale", shipmentPurpose: "SOLD" },
    { value: "gift", label: "Gift", shipmentPurpose: "GIFT" },
    { value: "sample", label: "Sample", shipmentPurpose: "SAMPLE" },
    { value: "repair", label: "Repair or Return", shipmentPurpose: "REPAIR" },
    { value: "return", label: "Return", shipmentPurpose: "RETURN" }
];

// Duties payment options for FedEx
export const DUTIES_PAYMENT_OPTIONS_FEDEX = [
    {
        value: "SENDER",
        label: "I will pay (Shipper)",
        description: "You pay all customs duties and taxes",
        icon: "💳",
        incoterm: "DDP"  // Equivalent incoterm for backward compatibility
    },
    {
        value: "RECIPIENT",
        label: "Recipient will pay",
        description: "Receiver pays all customs duties and taxes",
        icon: "📦",
        incoterm: "DAP"  // Equivalent incoterm for backward compatibility
    }
];

// Shipment purpose options for FedEx
export const SHIPMENT_PURPOSE_OPTIONS_FEDEX = [
    { value: "SOLD", label: "Sold (Commercial Sale)", exportReasonType: "commercial_purpose_or_sale" },
    { value: "GIFT", label: "Gift", exportReasonType: "gift" },
    { value: "SAMPLE", label: "Sample", exportReasonType: "sample" },
    { value: "REPAIR", label: "Repair", exportReasonType: "repair" },
    { value: "RETURN", label: "Return", exportReasonType: "return" },
    { value: "PERSONAL_EFFECTS", label: "Personal Effects", exportReasonType: "personal_effects" },
    { value: "NOT_SOLD", label: "Not Sold", exportReasonType: "not_sold" }
];

/**
 * Complete carrier UI configuration
 */
export const CARRIER_UI_CONFIG = {
    DHL: {
        name: "DHL Express",

        // Customs configuration
        customs: {
            // Incoterm field (DHL-specific)
            incotermField: {
                show: true,
                label: "Incoterm",
                helpText: "International Commercial Terms define shipping responsibilities and who pays duties/taxes",
                options: INCOTERM_OPTIONS_DHL,
                required: true
            },

            // Duties payment field (hidden for DHL, derived from incoterm)
            dutiesPaymentField: {
                show: false
            },

            // Export reason field (DHL-specific)
            exportReasonField: {
                show: true,
                label: "Export Reason",
                helpText: "Select the reason for this international shipment",
                options: EXPORT_REASON_OPTIONS_DHL,
                required: true
            },

            // Shipment purpose field (hidden for DHL, derived from export reason)
            shipmentPurposeField: {
                show: false
            }
        },

        // Validation rules
        validations: {
            specialInstructions: {
                maxLength: 20,
                message: "Special instructions must be under 20 characters (DHL requirement)"
            },
            addressLine: {
                maxLength: 45,
                lines: 3,
                message: "Address line must be under 45 characters (DHL limit)"
            },
            cityName: {
                maxLength: 45,
                noCommas: true,
                message: "City name must be under 45 characters and cannot contain commas (DHL requirement)"
            },
            itemDescription: {
                maxLength: 70,
                message: "Item description must be under 70 characters (DHL limit)"
            },
            pickupDate: {
                noWeekends: true,
                message: "DHL does not offer automated pickup on weekends. Please select a weekday."
            },
            dimensions: {
                required: true,
                message: "Package dimensions (length, width, height) are required for DHL rate calculation"
            },
            stateRequired: false  // DHL doesn't require state/province
        }
    },

    FEDEX: {
        name: "FedEx",

        // Customs configuration
        customs: {
            // Incoterm field (hidden for FedEx)
            incotermField: {
                show: false
            },

            // Duties payment field (FedEx-specific, user-friendly)
            dutiesPaymentField: {
                show: true,
                label: "Who Pays Duties & Taxes?",
                helpText: "Select who is responsible for paying customs duties and import taxes",
                options: DUTIES_PAYMENT_OPTIONS_FEDEX,
                required: true,
                displayAs: "radio"  // Show as radio buttons for better UX
            },

            // Export reason field (hidden for FedEx)
            exportReasonField: {
                show: false
            },

            // Shipment purpose field (FedEx-specific)
            shipmentPurposeField: {
                show: true,
                label: "Shipment Purpose",
                helpText: "What is the purpose of this international shipment?",
                options: SHIPMENT_PURPOSE_OPTIONS_FEDEX,
                required: true
            }
        },

        // Validation rules
        validations: {
            specialInstructions: {
                maxLength: null,  // No limit for FedEx
                message: null
            },
            addressLine: {
                maxLength: 35,
                lines: 2,
                message: "Address line must be under 35 characters (FedEx limit)"
            },
            cityName: {
                maxLength: 50,
                noCommas: false,
                message: "City name must be under 50 characters"
            },
            personName: {
                maxLength: 70,
                message: "Person name must be under 70 characters (FedEx limit)"
            },
            itemDescription: {
                maxLength: 50,
                message: "Item description must be under 50 characters (FedEx limit)"
            },
            pickupDate: {
                noWeekends: false,
                message: null
            },
            dimensions: {
                required: false,
                message: null
            },
            stateRequired: true,  // Show state field for US/CA/MX
            stateRequiredCountries: ["US", "CA", "MX"]
        }
    }
};

/**
 * Get carrier UI configuration
 * @param {string} carrier - Carrier name (DHL, FEDEX)
 * @returns {Object} Carrier UI configuration
 */
export function getCarrierUIConfig(carrier) {
    const normalizedCarrier = carrier?.toUpperCase();
    return CARRIER_UI_CONFIG[normalizedCarrier] || CARRIER_UI_CONFIG.DHL;
}

/**
 * Get validation rules for carrier
 * @param {string} carrier - Carrier name
 * @returns {Object} Validation rules
 */
export function getCarrierValidations(carrier) {
    const config = getCarrierUIConfig(carrier);
    return config.validations;
}

/**
 * Check if state/province required for country and carrier
 * @param {string} countryCode - ISO 2-char country code
 * @param {string} carrier - Carrier name
 * @returns {boolean} True if required
 */
export function isStateRequired(countryCode, carrier) {
    const config = getCarrierUIConfig(carrier);
    if (!config.validations.stateRequired) {
        return false;
    }
    return config.validations.stateRequiredCountries?.includes(countryCode) || false;
}
```

**Verification**:
- [ ] File created with complete configurations
- [ ] All options include backward compatibility mappings
- [ ] Helper functions work correctly
- [ ] DHL config matches existing functionality
- [ ] FedEx config complete

---

### **5.2 Create Validation Utility**

**File**: `Frontend-Quotation-Mgmt-App/src/utils/carrierValidation.js`

**Purpose**: Centralized validation logic using carrier configurations

**Implementation**:
```javascript
import { getCarrierUIConfig } from '../config/carrierUIConfig';

/**
 * Validate job order details based on carrier requirements
 * @param {Object} jobOrderDetails - Job order details to validate
 * @returns {Object} { isValid: boolean, errors: Object, errorsList: Array }
 */
export function validateJobOrderByCarrier(jobOrderDetails) {
    const carrier = jobOrderDetails?.agentConfirmedByCustomer;
    const config = getCarrierUIConfig(carrier);
    const rules = config.validations;

    const errors = {};
    const errorsList = [];

    // Helper to add error
    const addError = (field, message) => {
        errors[field] = message;
        errorsList.push(message);
    };

    // Validate special instructions
    const specialInstructions = jobOrderDetails?.pickup?.specialInstructionsForPickup;
    if (rules.specialInstructions.maxLength && specialInstructions) {
        if (specialInstructions.length > rules.specialInstructions.maxLength) {
            addError('pickup.specialInstructions', rules.specialInstructions.message);
        }
    }

    // Validate shipper address lines
    ['addressLine1', 'addressLine2', 'addressLine3'].forEach((field, index) => {
        const value = jobOrderDetails?.shipperDetails?.locationDetails?.[field];
        if (value && rules.addressLine.maxLength) {
            if (value.length > rules.addressLine.maxLength) {
                addError(`shipper.${field}`, `Shipper ${field}: ${rules.addressLine.message}`);
            }
        }
    });

    // Validate receiver address lines
    ['addressLine1', 'addressLine2', 'addressLine3'].forEach((field, index) => {
        const value = jobOrderDetails?.receiverDetails?.locationDetails?.[field];
        if (value && rules.addressLine.maxLength) {
            if (value.length > rules.addressLine.maxLength) {
                addError(`receiver.${field}`, `Receiver ${field}: ${rules.addressLine.message}`);
            }
        }
    });

    // Validate city names
    const shipperCity = jobOrderDetails?.shipperDetails?.locationDetails?.cityName;
    const receiverCity = jobOrderDetails?.receiverDetails?.locationDetails?.cityName;

    if (shipperCity) {
        if (rules.cityName.maxLength && shipperCity.length > rules.cityName.maxLength) {
            addError('shipper.cityName', `Shipper city: ${rules.cityName.message}`);
        }
        if (rules.cityName.noCommas && shipperCity.includes(',')) {
            addError('shipper.cityName', 'Shipper city name cannot contain commas (DHL requirement)');
        }
    }

    if (receiverCity) {
        if (rules.cityName.maxLength && receiverCity.length > rules.cityName.maxLength) {
            addError('receiver.cityName', `Receiver city: ${rules.cityName.message}`);
        }
        if (rules.cityName.noCommas && receiverCity.includes(',')) {
            addError('receiver.cityName', 'Receiver city name cannot contain commas (DHL requirement)');
        }
    }

    // Validate item description
    const itemDescription = jobOrderDetails?.containerInfo?.itemDescription;
    if (itemDescription && rules.itemDescription.maxLength) {
        if (itemDescription.length > rules.itemDescription.maxLength) {
            addError('itemDescription', rules.itemDescription.message);
        }
    }

    // Validate pickup date (weekends)
    if (rules.pickupDate.noWeekends && jobOrderDetails?.pickupDateAndTime) {
        const pickupDate = new Date(jobOrderDetails.pickupDateAndTime);
        const dayOfWeek = pickupDate.getDay();
        if (dayOfWeek === 0 || dayOfWeek === 6) {  // Sunday = 0, Saturday = 6
            addError('pickupDateAndTime', rules.pickupDate.message);
        }
    }

    // Validate dimensions (if required)
    if (rules.dimensions.required) {
        const packaging = jobOrderDetails?.containerInfo?.packaging || [];
        packaging.forEach((pkg, index) => {
            if (!pkg.dimensions?.width || pkg.dimensions.width <= 0) {
                addError(`packaging.${index}.width`, `Package ${index + 1}: Width is required (${carrier} requirement)`);
            }
            if (!pkg.dimensions?.height || pkg.dimensions.height <= 0) {
                addError(`packaging.${index}.height`, `Package ${index + 1}: Height is required (${carrier} requirement)`);
            }
            if (!pkg.dimensions?.length || pkg.dimensions.length <= 0) {
                addError(`packaging.${index}.length`, `Package ${index + 1}: Length is required (${carrier} requirement)`);
            }
        });
    }

    // Validate state/province for FedEx US/CA/MX
    if (rules.stateRequired && rules.stateRequiredCountries) {
        const receiverCountry = jobOrderDetails?.receiverDetails?.locationDetails?.countryCode;
        const receiverState = jobOrderDetails?.receiverDetails?.locationDetails?.stateOrProvinceCode;

        if (rules.stateRequiredCountries.includes(receiverCountry) && !receiverState) {
            addError('receiver.stateOrProvinceCode', `State/Province is required for ${receiverCountry} addresses when shipping with ${carrier}`);
        }

        const shipperCountry = jobOrderDetails?.shipperDetails?.locationDetails?.countryCode;
        const shipperState = jobOrderDetails?.shipperDetails?.locationDetails?.stateOrProvinceCode;

        if (rules.stateRequiredCountries.includes(shipperCountry) && !shipperState) {
            addError('shipper.stateOrProvinceCode', `State/Province is required for ${shipperCountry} addresses when shipping with ${carrier}`);
        }
    }

    // Validate customs fields
    if (jobOrderDetails?.containerInfo?.isCustomsDeclarable) {
        const exportDeclaration = jobOrderDetails?.containerInfo?.exportDeclaration;

        if (!exportDeclaration?.lineItems || exportDeclaration.lineItems.length === 0) {
            addError('exportDeclaration', 'Export declaration line items are required for international shipments');
        }

        if (!exportDeclaration?.invoice?.number) {
            addError('invoice.number', 'Invoice number is required for customs clearance');
        }

        if (!exportDeclaration?.invoice?.date) {
            addError('invoice.date', 'Invoice date is required for customs clearance');
        }

        // Validate carrier-specific purpose fields
        if (carrier === 'DHL' && !exportDeclaration?.exportReasonType) {
            addError('exportReasonType', 'Export reason is required for DHL international shipments');
        }

        if (carrier === 'FEDEX' && !exportDeclaration?.shipmentPurpose) {
            addError('shipmentPurpose', 'Shipment purpose is required for FedEx international shipments');
        }
    }

    return {
        isValid: errorsList.length === 0,
        errors,
        errorsList
    };
}

/**
 * Get character count info for field
 * @param {string} value - Field value
 * @param {number} maxLength - Max length allowed
 * @returns {Object} { count: number, max: number, isValid: boolean }
 */
export function getCharacterCountInfo(value, maxLength) {
    const count = value?.length || 0;
    return {
        count,
        max: maxLength,
        isValid: !maxLength || count <= maxLength,
        remaining: maxLength ? maxLength - count : null
    };
}

/**
 * Check if weekend
 * @param {Date} date - Date to check
 * @returns {boolean} True if weekend
 */
export function isWeekend(date) {
    const dayOfWeek = date.getDay();
    return dayOfWeek === 0 || dayOfWeek === 6;
}
```

**Verification**:
- [ ] Validation utility created
- [ ] All carrier-specific rules implemented
- [ ] Helper functions work correctly
- [ ] Returns structured error object

---

### **Phase 5 Completion Checklist**:
- [ ] Carrier UI config created
- [ ] Validation utility created
- [ ] All options include backward compatibility
- [ ] DHL config matches existing behavior
- [ ] FedEx config complete with all fields

---

## Phase 6: Frontend UI Updates

### **Objective**: Update frontend components to use carrier-specific configurations

### **6.1 Update JobOrderContainerDetails Component**

**File**: `Frontend-Quotation-Mgmt-App/src/screens/dashboard/jobOrders/components/JobOrderContainerDetails.jsx`

**Changes Required**:

#### **Import carrier configuration at top**:
```javascript
import { getCarrierUIConfig, DUTIES_PAYMENT_OPTIONS_FEDEX, SHIPMENT_PURPOSE_OPTIONS_FEDEX } from '../../../../config/carrierUIConfig';
```

#### **Get carrier config in component**:
```javascript
const JobOrderContainerDetails = ({ jobOrderDetails, setJobOrderDetails, viewOnly, validationErrors }) => {
    // Get carrier-specific configuration
    const carrier = jobOrderDetails?.agentConfirmedByCustomer;
    const carrierConfig = getCarrierUIConfig(carrier);
    const customsConfig = carrierConfig.customs;

    // ... rest of component
```

#### **Replace Incoterm Section** (around line 784):

**Original**:
```javascript
{/* Incoterm */}
<div>
    <label>Incoterm</label>
    <select
        value={jobOrderDetails?.containerInfo?.incoterm || ''}
        onChange={(e) => handleContainerInfoChange("info", null, "incoterm", e.target.value)}
        style={inputStyle}
        disabled={viewOnly}
    >
        <option value="">Select Incoterm</option>
        {INCOTERM_OPTIONS.map(option => (
            <option key={option.value} value={option.value}>{option.label}</option>
        ))}
    </select>
</div>
```

**Replace With**:
```javascript
{/* Carrier-specific customs fields */}
{customsConfig.incotermField.show && (
    <div>
        <label style={{ display: "block", marginBottom: "5px", fontWeight: "bold" }}>
            {customsConfig.incotermField.label} {customsConfig.incotermField.required && '*'}
        </label>
        {customsConfig.incotermField.helpText && (
            <div style={{ fontSize: "0.85rem", color: "#666", marginBottom: "8px" }}>
                {customsConfig.incotermField.helpText}
            </div>
        )}
        <select
            value={jobOrderDetails?.containerInfo?.incoterm || ''}
            onChange={(e) => {
                const selectedOption = customsConfig.incotermField.options.find(opt => opt.value === e.target.value);

                setJobOrderDetails(prev => {
                    const newState = JSON.parse(JSON.stringify(prev));
                    newState.containerInfo.incoterm = e.target.value;

                    // Auto-calculate dutiesPaymentType from selected incoterm
                    if (selectedOption) {
                        newState.containerInfo.dutiesPaymentType = selectedOption.dutiesPaymentType;
                    }

                    return newState;
                });
            }}
            style={getInputStyle('incoterm')}
            disabled={viewOnly}
        >
            <option value="">Select {customsConfig.incotermField.label}</option>
            {customsConfig.incotermField.options.map(option => (
                <option key={option.value} value={option.value}>{option.label}</option>
            ))}
        </select>
        {validationErrors['incoterm'] && (
            <div style={{ color: '#f44336', fontSize: '0.75rem', marginTop: '4px' }}>
                {validationErrors['incoterm']}
            </div>
        )}
    </div>
)}

{customsConfig.dutiesPaymentField.show && (
    <div>
        <label style={{ display: "block", marginBottom: "5px", fontWeight: "bold" }}>
            {customsConfig.dutiesPaymentField.label} {customsConfig.dutiesPaymentField.required && '*'}
        </label>
        {customsConfig.dutiesPaymentField.helpText && (
            <div style={{ fontSize: "0.85rem", color: "#666", marginBottom: "8px" }}>
                {customsConfig.dutiesPaymentField.helpText}
            </div>
        )}

        {customsConfig.dutiesPaymentField.displayAs === 'radio' ? (
            // Radio button display for FedEx
            <div style={{ display: 'flex', flexDirection: 'column', gap: '12px' }}>
                {customsConfig.dutiesPaymentField.options.map(option => (
                    <label
                        key={option.value}
                        style={{
                            display: 'flex',
                            alignItems: 'flex-start',
                            padding: '12px',
                            border: '1px solid #ddd',
                            borderRadius: '6px',
                            cursor: viewOnly ? 'not-allowed' : 'pointer',
                            backgroundColor: jobOrderDetails?.containerInfo?.dutiesPaymentType === option.value ? '#e3f2fd' : '#fff',
                            transition: 'all 0.2s'
                        }}
                    >
                        <input
                            type="radio"
                            value={option.value}
                            checked={jobOrderDetails?.containerInfo?.dutiesPaymentType === option.value}
                            onChange={(e) => {
                                setJobOrderDetails(prev => {
                                    const newState = JSON.parse(JSON.stringify(prev));
                                    newState.containerInfo.dutiesPaymentType = e.target.value;

                                    // Set equivalent incoterm for backward compatibility
                                    if (option.incoterm) {
                                        newState.containerInfo.incoterm = option.incoterm;
                                    }

                                    return newState;
                                });
                            }}
                            disabled={viewOnly}
                            style={{ marginRight: '12px', marginTop: '4px' }}
                        />
                        <div style={{ flex: 1 }}>
                            <div style={{ fontWeight: 'bold', marginBottom: '4px' }}>
                                {option.icon && <span style={{ marginRight: '8px' }}>{option.icon}</span>}
                                {option.label}
                            </div>
                            <div style={{ fontSize: '0.85rem', color: '#666' }}>
                                {option.description}
                            </div>
                        </div>
                    </label>
                ))}
            </div>
        ) : (
            // Dropdown display
            <select
                value={jobOrderDetails?.containerInfo?.dutiesPaymentType || ''}
                onChange={(e) => {
                    const selectedOption = customsConfig.dutiesPaymentField.options.find(opt => opt.value === e.target.value);

                    setJobOrderDetails(prev => {
                        const newState = JSON.parse(JSON.stringify(prev));
                        newState.containerInfo.dutiesPaymentType = e.target.value;

                        // Set equivalent incoterm for backward compatibility
                        if (selectedOption?.incoterm) {
                            newState.containerInfo.incoterm = selectedOption.incoterm;
                        }

                        return newState;
                    });
                }}
                style={getInputStyle('dutiesPaymentType')}
                disabled={viewOnly}
            >
                <option value="">Select payment type</option>
                {customsConfig.dutiesPaymentField.options.map(option => (
                    <option key={option.value} value={option.value}>{option.label}</option>
                ))}
            </select>
        )}

        {validationErrors['dutiesPaymentType'] && (
            <div style={{ color: '#f44336', fontSize: '0.75rem', marginTop: '4px' }}>
                {validationErrors['dutiesPaymentType']}
            </div>
        )}
    </div>
)}
```

#### **Add State/Province Field** (for addresses):

**Add to Shipper/Receiver LocationDetails sections** (after postal code field):

```javascript
{/* State/Province Code - Show for FedEx US/CA/MX */}
{carrier === 'FEDEX' &&
 ['US', 'CA', 'MX'].includes(jobOrderDetails?.shipperDetails?.locationDetails?.countryCode) && (
    <div>
        <label style={{ display: "block", marginBottom: "5px", fontWeight: "bold" }}>
            State/Province Code *
        </label>
        <input
            type="text"
            value={jobOrderDetails?.shipperDetails?.locationDetails?.stateOrProvinceCode || ''}
            onChange={(e) => handleShipperDetailsChange("locationDetails", "stateOrProvinceCode", e.target.value.toUpperCase())}
            style={getInputStyle('shipper.stateOrProvinceCode')}
            disabled={viewOnly}
            maxLength={3}
            placeholder={jobOrderDetails?.shipperDetails?.locationDetails?.countryCode === 'US' ? 'e.g., NY' : 'e.g., ON'}
        />
        {validationErrors['shipper.stateOrProvinceCode'] && (
            <div style={{ color: '#f44336', fontSize: '0.75rem', marginTop: '4px' }}>
                {validationErrors['shipper.stateOrProvinceCode']}
            </div>
        )}
        <div style={{ fontSize: '0.75rem', color: '#666', marginTop: '4px' }}>
            Required for FedEx shipments to/from US, Canada, Mexico
        </div>
    </div>
)}
```

#### **Update Export Reason / Shipment Purpose Section**:

**Find the export reason dropdown** (search for "exportReasonType") and **replace with**:

```javascript
{/* Carrier-specific export reason / shipment purpose */}
{customsConfig.exportReasonField.show && (
    <div>
        <label style={{ display: "block", marginBottom: "5px", fontWeight: "bold" }}>
            {customsConfig.exportReasonField.label} {customsConfig.exportReasonField.required && '*'}
        </label>
        {customsConfig.exportReasonField.helpText && (
            <div style={{ fontSize: "0.85rem", color: "#666", marginBottom: "8px" }}>
                {customsConfig.exportReasonField.helpText}
            </div>
        )}
        <select
            value={jobOrderDetails?.containerInfo?.exportDeclaration?.exportReasonType || ''}
            onChange={(e) => {
                const selectedOption = customsConfig.exportReasonField.options.find(opt => opt.value === e.target.value);

                setJobOrderDetails(prev => {
                    const newState = JSON.parse(JSON.stringify(prev));
                    if (!newState.containerInfo.exportDeclaration) {
                        newState.containerInfo.exportDeclaration = { lineItems: [], invoice: {} };
                    }
                    newState.containerInfo.exportDeclaration.exportReasonType = e.target.value;

                    // Auto-calculate shipmentPurpose
                    if (selectedOption) {
                        newState.containerInfo.exportDeclaration.shipmentPurpose = selectedOption.shipmentPurpose;
                    }

                    return newState;
                });
            }}
            style={getInputStyle('exportReasonType')}
            disabled={viewOnly}
        >
            <option value="">Select {customsConfig.exportReasonField.label}</option>
            {customsConfig.exportReasonField.options.map(option => (
                <option key={option.value} value={option.value}>{option.label}</option>
            ))}
        </select>
        {validationErrors['exportReasonType'] && (
            <div style={{ color: '#f44336', fontSize: '0.75rem', marginTop: '4px' }}>
                {validationErrors['exportReasonType']}
            </div>
        )}
    </div>
)}

{customsConfig.shipmentPurposeField.show && (
    <div>
        <label style={{ display: "block", marginBottom: "5px", fontWeight: "bold" }}>
            {customsConfig.shipmentPurposeField.label} {customsConfig.shipmentPurposeField.required && '*'}
        </label>
        {customsConfig.shipmentPurposeField.helpText && (
            <div style={{ fontSize: "0.85rem", color: "#666", marginBottom: "8px" }}>
                {customsConfig.shipmentPurposeField.helpText}
            </div>
        )}
        <select
            value={jobOrderDetails?.containerInfo?.exportDeclaration?.shipmentPurpose || ''}
            onChange={(e) => {
                const selectedOption = customsConfig.shipmentPurposeField.options.find(opt => opt.value === e.target.value);

                setJobOrderDetails(prev => {
                    const newState = JSON.parse(JSON.stringify(prev));
                    if (!newState.containerInfo.exportDeclaration) {
                        newState.containerInfo.exportDeclaration = { lineItems: [], invoice: {} };
                    }
                    newState.containerInfo.exportDeclaration.shipmentPurpose = e.target.value;

                    // Set equivalent exportReasonType for backward compatibility
                    if (selectedOption?.exportReasonType) {
                        newState.containerInfo.exportDeclaration.exportReasonType = selectedOption.exportReasonType;
                    }

                    return newState;
                });
            }}
            style={getInputStyle('shipmentPurpose')}
            disabled={viewOnly}
        >
            <option value="">Select {customsConfig.shipmentPurposeField.label}</option>
            {customsConfig.shipmentPurposeField.options.map(option => (
                <option key={option.value} value={option.value}>{option.label}</option>
            ))}
        </select>
        {validationErrors['shipmentPurpose'] && (
            <div style={{ color: '#f44336', fontSize: '0.75rem', marginTop: '4px' }}>
                {validationErrors['shipmentPurpose']}
            </div>
        )}
    </div>
)}
```

**Verification**:
- [ ] Imports added correctly
- [ ] Carrier config retrieved in component
- [ ] Incoterm field shown only for DHL
- [ ] Duties payment field shown only for FedEx
- [ ] Export reason shown only for DHL
- [ ] Shipment purpose shown only for FedEx
- [ ] State/province shown for FedEx US/CA/MX only
- [ ] Auto-calculation works (incoterm ↔ dutiesPaymentType)
- [ ] Backward compatibility maintained

---

### **6.2 Update JobOrderReviewModal Validation**

**File**: `Frontend-Quotation-Mgmt-App/src/screens/dashboard/jobOrders/JobOrderReviewModal.jsx`

**Changes Required**:

#### **Import validation utility**:
```javascript
import { validateJobOrderByCarrier } from '../../../utils/carrierValidation';
```

#### **Replace existing validateFields function** (around line 67-377):

**Find**:
```javascript
const validateFields = (details) => {
    // ... existing validation logic
};
```

**Replace With**:
```javascript
const validateFields = (details) => {
    // Use centralized carrier-aware validation
    return validateJobOrderByCarrier(details);
};
```

**Remove all DHL-specific validation logic** since it's now in the utility.

**Verification**:
- [ ] Validation import added
- [ ] validateFields replaced with utility call
- [ ] All DHL-specific checks removed
- [ ] Validation works for both carriers
- [ ] Error messages carrier-specific

---

### **6.3 Update FetchDHLRatesModal** → **Rename to FetchRatesModal**

**File**: `Frontend-Quotation-Mgmt-App/src/screens/dashboard/jobOrders/FetchDHLRatesModal.jsx`

**Rename to**: `FetchRatesModal.jsx`

**Changes Required**:

#### **Update component name and props**:
```javascript
const FetchRatesModal = ({ agentConfirmedByCustomer, rates, isFetchingRates, onProceed, onClose, isProcessing }) => {
    const isDHL = agentConfirmedByCustomer === "DHL";
    const isFEDEX = agentConfirmedByCustomer === "FEDEX";

    // ... rest of component
```

#### **Update conditional rendering**:
```javascript
{isFetchingRates ? (
    <div>Loading rates from {agentConfirmedByCustomer}...</div>
) : (
    <>
        {(isDHL || isFEDEX) && rates ? (
            <ShipmentOptions fetchRates={rates} carrier={agentConfirmedByCustomer} />
        ) : (
            <div>
                Rates can only be fetched from {agentConfirmedByCustomer}.
                You can proceed to create an entry in the ERP.
            </div>
        )}
    </>
)}
```

#### **Update all imports in other files**:

**In JobOrderReviewModal.jsx**:
```javascript
import FetchRatesModal from './FetchRatesModal.jsx';  // Updated import
```

**Verification**:
- [ ] File renamed to FetchRatesModal.jsx
- [ ] Component supports both DHL and FedEx
- [ ] All imports updated
- [ ] No hardcoded "DHL" references

---

### **Phase 6 Completion Checklist**:
- [ ] JobOrderContainerDetails updated with carrier-specific UI
- [ ] Incoterm field shown only for DHL
- [ ] Duties payment shown only for FedEx (as radio buttons)
- [ ] Export reason shown only for DHL
- [ ] Shipment purpose shown only for FedEx
- [ ] State/province shown for FedEx US/CA/MX
- [ ] Auto-calculation works both ways
- [ ] Validation uses carrier utility
- [ ] FetchDHLRatesModal renamed to FetchRatesModal
- [ ] All carrier-specific hardcoding removed
- [ ] UI shows appropriate terminology per carrier

---

## Phase 7: Service Layer Integration

### **Objective**: Update existing services to use CarrierFactory instead of direct DHL calls

### **7.1 Update JobOrderRateFetching Service**

**File**: `Backend-Quotation-Mgmt/src/logic/jobOrder/service/JobOrderRateFetching.service.js`

**Changes Required**:

#### **Add imports at top**:
```javascript
const CarrierFactory = require('../../../carriers/CarrierFactory');
```

#### **Add new carrier-agnostic functions**:
```javascript
/**
 * Get rates from specified carrier
 * @param {Object} jobOrderData - Complete JobOrder object
 * @param {string} carrierName - Carrier name (DHL, FEDEX)
 * @returns {Promise<Object>} Rate response
 */
async function getCarrierRates(jobOrderData, carrierName) {
    try {
        console.log(`[JobOrderRateFetching] Fetching rates from ${carrierName} for JobOrder: ${jobOrderData._id}`);

        const carrier = CarrierFactory.getCarrier(carrierName);
        const result = await carrier.getRates(jobOrderData);

        if (result.success) {
            console.log(`[JobOrderRateFetching] Successfully fetched ${result.data?.products?.length || 0} rate options from ${carrierName}`);
        } else {
            console.error(`[JobOrderRateFetching] Failed to fetch rates from ${carrierName}:`, result.message);
        }

        return result;
    } catch (error) {
        console.error(`[JobOrderRateFetching] Error fetching rates from ${carrierName}:`, error);
        return responseModel(false, `Failed to fetch rates from ${carrierName}`, error.message);
    }
}

/**
 * Book shipment with specified carrier
 * @param {Object} jobOrderData - Complete JobOrder object
 * @param {string} carrierName - Carrier name (DHL, FEDEX)
 * @returns {Promise<Object>} Booking response
 */
async function makeBookingUsingCarrier(jobOrderData, carrierName) {
    try {
        console.log(`[JobOrderRateFetching] Booking shipment with ${carrierName} for JobOrder: ${jobOrderData._id}`);

        const carrier = CarrierFactory.getCarrier(carrierName);
        const result = await carrier.bookShipment(jobOrderData);

        if (result.success) {
            console.log(`[JobOrderRateFetching] Successfully booked shipment with ${carrierName}. Tracking: ${result.data?.shipmentTrackingNumber}`);
        } else {
            console.error(`[JobOrderRateFetching] Failed to book with ${carrierName}:`, result.message);
        }

        return result;
    } catch (error) {
        console.error(`[JobOrderRateFetching] Error booking with ${carrierName}:`, error);
        return responseModel(false, `Failed to book shipment with ${carrierName}`, error.message);
    }
}
```

#### **Keep existing DHL functions for backward compatibility**:
```javascript
/**
 * @deprecated Use getCarrierRates() instead
 * Kept for backward compatibility
 */
async function getDhlRatesFromAPI(params, isTestServer = false) {
    console.warn('[JobOrderRateFetching] getDhlRatesFromAPI is deprecated. Use getCarrierRates() instead.');

    // Convert params to JobOrder-like object for new function
    const jobOrderData = {
        _id: 'legacy-call',
        agentConfirmedByCustomer: 'DHL',
        shipperDetails: {
            locationDetails: {
                countryCode: params.originCountryCode,
                cityName: params.originCityName,
                postalCode: params.originPostalCode
            }
        },
        receiverDetails: {
            locationDetails: {
                countryCode: params.destinationCountryCode,
                cityName: params.destinationCityName,
                postalCode: params.destinationPostalCode
            }
        },
        containerInfo: {
            packaging: [{
                weight: params.weight,
                dimensions: {
                    length: params.length,
                    width: params.width,
                    height: params.height
                },
                quantity: 1
            }],
            isCustomsDeclarable: params.isCustomsDeclarable
        },
        pickupDateAndTime: params.plannedShippingDate
    };

    return await getCarrierRates(jobOrderData, 'DHL');
}

/**
 * @deprecated Use getCarrierRates() instead
 * Kept for backward compatibility
 */
async function getDhlRatesForMultiplePackages(bodyPayload, isTestServer = false) {
    console.warn('[JobOrderRateFetching] getDhlRatesForMultiplePackages is deprecated. Use getCarrierRates() instead.');

    // For new implementation, bodyPayload should be full JobOrder object
    return await getCarrierRates(bodyPayload, 'DHL');
}

/**
 * @deprecated Use makeBookingUsingCarrier() instead
 * Kept for backward compatibility
 */
async function makeBookingUsingDHL(jobOrderInfo, isTestServer = false) {
    console.warn('[JobOrderRateFetching] makeBookingUsingDHL is deprecated. Use makeBookingUsingCarrier() instead.');
    return await makeBookingUsingCarrier(jobOrderInfo, 'DHL');
}
```

#### **Update module exports**:
```javascript
module.exports = {
    // New carrier-agnostic exports (RECOMMENDED)
    getCarrierRates,
    makeBookingUsingCarrier,

    // Legacy exports (DEPRECATED but kept for backward compatibility)
    getDhlRatesFromAPI,
    getDhlRatesForMultiplePackages,
    makeBookingUsingDHL,
    getDhlShipmentImage
};
```

**Verification**:
- [ ] New carrier-agnostic functions added
- [ ] Legacy functions kept and marked deprecated
- [ ] Logging shows carrier name
- [ ] Exports include both new and old functions

---

### **7.2 Update JobOrderCRUD Service**

**File**: `Backend-Quotation-Mgmt/src/logic/jobOrder/service/JobOrderCRUD.service.js`

**Changes Required**:

#### **Find the REVIEW_COMPLETED status handler** (around line 371-445):

**Current**:
```javascript
if (status === "REVIEW_COMPLETED") {
    await makeBookingUsingDHL(...)
    await createJobOrderInERP(...)
}
```

**Replace With**:
```javascript
if (status === "REVIEW_COMPLETED") {
    const carrierName = jobOrder.agentConfirmedByCustomer;

    console.log(`[JobOrderCRUD] Starting AWB automation flow for ${carrierName}`);

    // Check if carrier supports automated booking
    const supportedCarriers = ['DHL', 'FEDEX'];
    if (!supportedCarriers.includes(carrierName)) {
        console.log(`[JobOrderCRUD] ${carrierName} does not support automated booking. Moving to PARK_AND_CLOSE.`);

        // For unsupported carriers, create ERP entry without AWB
        const erpRes = await createJobOrderInERPService.createJobOrderInERP(
            jobOrder,
            normalJobOrder,
            quotation,
            changeJobOrderStatus,
            null  // No AWB details
        );

        if (!erpRes.success) {
            return responseModel(false, erpRes.error || 'Failed to create job order in ERP');
        }

        return responseModel(true, 'Job order created in ERP for manual processing');
    }

    // For supported carriers (DHL, FEDEX), proceed with AWB automation
    const awbRes = await handleAWBAutomationFlow.startAWBAutomationFlow(
        jobOrder,
        normalJobOrder,
        quotation,
        changeJobOrderStatus,
        true,  // skipStatusUpdate
        carrierName  // Pass carrier name
    );

    if (!awbRes.success) {
        return responseModel(false, awbRes.message || 'Failed to generate AWB');
    }

    // Create ERP entry with AWB details
    const erpRes = await createJobOrderInERPService.createJobOrderInERP(
        jobOrder,
        normalJobOrder,
        quotation,
        changeJobOrderStatus,
        awbRes.data.awbDetails
    );

    if (!erpRes.success) {
        // AWB created but ERP sync failed - needs manual intervention
        console.error('[JobOrderCRUD] AWB created but ERP sync failed:', erpRes.error);
        return responseModel(false, 'AWB generated but failed to sync with ERP. Manual intervention required.');
    }

    return responseModel(true, 'Job order created successfully');
}
```

**Verification**:
- [ ] Carrier name extracted from jobOrder
- [ ] Supported carriers checked
- [ ] Unsupported carriers handled gracefully
- [ ] Carrier name passed to AWB automation
- [ ] Error handling comprehensive

---

### **7.3 Update HandleAWBAutomationFlow Service**

**File**: `Backend-Quotation-Mgmt/src/logic/jobOrder/service/workflows/syncJobOrdersWithERP/handleAWBAutomationFlow/HandleAWBAutomationFlow.service.js`

**Changes Required**:

#### **Update function signature**:
```javascript
async function startAWBAutomationFlow(
    jobOrder,
    normalJobOrderInfo,
    quotationInfo,
    changeJobOrderStatus,
    skipStatusUpdate = false,
    carrierName = null  // NEW PARAMETER
) {
```

#### **Determine carrier**:
```javascript
// Determine carrier (from parameter or jobOrder)
const carrier = carrierName || jobOrder.agentConfirmedByCustomer;

console.log(`[HandleAWBAutomationFlow] Starting AWB automation for ${carrier}`);
```

#### **Replace DHL-specific call**:

**Current**:
```javascript
const dhlShipmentDetails = await makeBookingUsingDHL(normalJobOrderInfo, isTestServer);
```

**Replace With**:
```javascript
const { makeBookingUsingCarrier } = require('../../JobOrderRateFetching.service');

const shipmentResult = await makeBookingUsingCarrier(normalJobOrderInfo, carrier);

if (!shipmentResult.success) {
    console.error(`[HandleAWBAutomationFlow] ${carrier} booking failed:`, shipmentResult.message);
    return responseModel(false, `Failed to book shipment with ${carrier}: ${shipmentResult.message}`, shipmentResult.data);
}

const shipmentDetails = shipmentResult.data;
console.log(`[HandleAWBAutomationFlow] ${carrier} booking successful. Tracking: ${shipmentDetails.shipmentTrackingNumber}`);
```

#### **Update response**:
```javascript
return responseModel(true, `AWB generated successfully via ${carrier}`, {
    awbDetails: shipmentDetails
});
```

**Verification**:
- [ ] Carrier parameter added
- [ ] Uses makeBookingUsingCarrier
- [ ] Works for both DHL and FedEx
- [ ] Error handling maintained
- [ ] Logging shows carrier name

---

### **7.4 Update JobOrder Controller**

**File**: `Backend-Quotation-Mgmt/src/logic/jobOrder/controller/JobOrder.controller.js`

**Changes Required**:

#### **Find rate fetching endpoints**:

**Current GET /dhl/rates**:
```javascript
async getDHLRates(req, res) {
    const result = await getDhlRatesFromAPI(req.query);
    // ...
}
```

**Add new carrier-agnostic endpoint**:
```javascript
/**
 * Get rates from specified carrier
 * POST /job-orders/rates/:carrier
 */
async getCarrierRates(req, res) {
    try {
        const { carrier } = req.params;
        const jobOrderData = req.body;

        console.log(`[JobOrderController] Fetching rates from ${carrier}`);

        const result = await jobOrderRateFetchingService.getCarrierRates(jobOrderData, carrier);

        if (result.success) {
            return res.status(200).json(result);
        } else {
            return res.status(400).json(result);
        }
    } catch (error) {
        console.error('[JobOrderController] Error fetching carrier rates:', error);
        return res.status(500).json(responseModel(false, 'Internal server error', error.message));
    }
}
```

**Keep existing DHL endpoint for backward compatibility**:
```javascript
/**
 * @deprecated Use getCarrierRates instead
 * GET /job-orders/dhl/rates
 */
async getDHLRates(req, res) {
    console.warn('[JobOrderController] /dhl/rates endpoint is deprecated. Use /rates/:carrier instead.');
    // ... existing implementation
}
```

**Verification**:
- [ ] New carrier-agnostic endpoint added
- [ ] Old DHL endpoints kept
- [ ] Error handling maintained

---

### **7.5 Update JobOrder Routes**

**File**: `Backend-Quotation-Mgmt/src/logic/jobOrder/routes/JobOrder.routes.js`

**Add new routes**:
```javascript
// New carrier-agnostic routes
router.post('/rates/:carrier', jobOrderController.getCarrierRates);

// Keep old DHL routes for backward compatibility
router.get('/dhl/rates', jobOrderController.getDHLRates);
router.post('/dhl/rates/multiple', jobOrderController.getDHLRatesMultiple);
```

**Verification**:
- [ ] New routes added
- [ ] Old routes maintained
- [ ] Route order correct (specific before general)

---

### **Phase 7 Completion Checklist**:
- [ ] JobOrderRateFetching service updated
- [ ] JobOrderCRUD service updated
- [ ] HandleAWBAutomationFlow service updated
- [ ] Controller updated with new endpoints
- [ ] Routes updated
- [ ] Backward compatibility maintained
- [ ] All existing DHL functionality works
- [ ] New carrier-agnostic functions tested
- [ ] Logging shows carrier information

---

## Phase 8: Testing & Validation

### **Objective**: Comprehensive testing to ensure both DHL and FedEx work correctly

### **8.1 Unit Tests**

**Create test files**:

#### **Test Carrier Factory**:
```javascript
// tests/carriers/CarrierFactory.test.js
const CarrierFactory = require('../../src/carriers/CarrierFactory');

describe('CarrierFactory', () => {
    test('should return DHL carrier instance', () => {
        const carrier = CarrierFactory.getCarrier('DHL');
        expect(carrier).toBeDefined();
        expect(carrier.constructor.name).toBe('DHLCarrier');
    });

    test('should return FedEx carrier instance', () => {
        const carrier = CarrierFactory.getCarrier('FEDEX');
        expect(carrier).toBeDefined();
        expect(carrier.constructor.name).toBe('FedExCarrier');
    });

    test('should return same instance (singleton)', () => {
        const carrier1 = CarrierFactory.getCarrier('DHL');
        const carrier2 = CarrierFactory.getCarrier('DHL');
        expect(carrier1).toBe(carrier2);
    });

    test('should throw error for unsupported carrier', () => {
        expect(() => CarrierFactory.getCarrier('UPS')).toThrow();
    });
});
```

#### **Test Transformers**:
```javascript
// tests/transformers/DHLTransformer.test.js
// tests/transformers/FedExTransformer.test.js
```

#### **Test Auth Service**:
```javascript
// tests/carriers/FedExAuthService.test.js
```

**Verification**:
- [ ] All unit tests pass
- [ ] Code coverage > 80%

---

### **8.2 Integration Tests (Sandbox)**

**Test scenarios for both carriers**:

#### **DHL Integration Tests**:
```javascript
// Integration test: DHL Rate Fetching
async function testDHLRates() {
    const jobOrder = {
        _id: 'test-123',
        agentConfirmedByCustomer: 'DHL',
        shipperDetails: { /* ... */ },
        receiverDetails: { /* ... */ },
        containerInfo: { /* ... */ },
        pickupDateAndTime: new Date().toISOString()
    };

    const carrier = CarrierFactory.getCarrier('DHL');
    const result = await carrier.getRates(jobOrder);

    console.log('DHL Rates Result:', result);
    expect(result.success).toBe(true);
    expect(result.data.products).toBeDefined();
}

// Integration test: DHL Shipment Booking
async function testDHLBooking() {
    // ... similar structure
}
```

#### **FedEx Integration Tests**:
```javascript
// Integration test: FedEx Authentication
async function testFedExAuth() {
    const fedexAuthService = require('../../src/carriers/FedExAuthService');
    const token = await fedexAuthService.getAccessToken();

    console.log('Token acquired:', token.substring(0, 20) + '...');
    expect(token).toBeDefined();
    expect(token.length).toBeGreaterThan(0);
}

// Integration test: FedEx Rate Fetching
async function testFedExRates() {
    const jobOrder = {
        _id: 'test-456',
        agentConfirmedByCustomer: 'FEDEX',
        shipperDetails: {
            locationDetails: {
                countryCode: 'AE',
                cityName: 'Dubai',
                postalCode: '12345',
                stateOrProvinceCode: '',
                addressLine1: 'Test Address'
            },
            customerDetails: {
                companyName: 'Test Company',
                fullName: 'Test User',
                phone: ['+971501234567']
            }
        },
        receiverDetails: {
            locationDetails: {
                countryCode: 'US',
                cityName: 'New York',
                postalCode: '10001',
                stateOrProvinceCode: 'NY',  // Required for US
                addressLine1: 'Test Address'
            },
            customerDetails: {
                companyName: 'Test Company US',
                fullName: 'Test User US',
                phone: ['+12125551234']
            }
        },
        containerInfo: {
            packaging: [{
                weight: 5.0,
                dimensions: { length: 30, width: 20, height: 10 },
                quantity: 1
            }],
            isCustomsDeclarable: true,
            itemDescription: 'Test Item',
            declaredValue: 100,
            declaredValueCurrency: 'USD',
            dutiesPaymentType: 'SENDER',
            exportDeclaration: {
                shipmentPurpose: 'SOLD',
                lineItems: [{
                    description: 'Test Item',
                    price: 100,
                    quantity: { value: 1, unitOfMeasurement: 'PCS' },
                    weight: { grossValue: 5.0, netValue: 4.5 }
                }],
                invoice: {
                    date: '2025-10-22',
                    number: 'INV-TEST-001'
                }
            }
        },
        pickupDateAndTime: new Date(Date.now() + 86400000).toISOString()  // Tomorrow
    };

    const carrier = CarrierFactory.getCarrier('FEDEX');
    const result = await carrier.getRates(jobOrder);

    console.log('FedEx Rates Result:', JSON.stringify(result, null, 2));
    expect(result.success).toBe(true);
    expect(result.data.products).toBeDefined();
}

// Integration test: FedEx Shipment Booking
async function testFedExBooking() {
    // ... same jobOrder structure as above

    const carrier = CarrierFactory.getCarrier('FEDEX');
    const result = await carrier.bookShipment(jobOrder);

    console.log('FedEx Booking Result:', JSON.stringify(result, null, 2));
    expect(result.success).toBe(true);
    expect(result.data.shipmentTrackingNumber).toBeDefined();
    expect(result.data.documents).toBeDefined();
}
```

**Test Checklist**:
- [ ] FedEx authentication works
- [ ] FedEx rate fetching returns results
- [ ] FedEx shipment booking creates AWB
- [ ] FedEx labels returned in response
- [ ] DHL rate fetching still works
- [ ] DHL shipment booking still works
- [ ] Token expiration/refresh tested
- [ ] Error handling tested (invalid credentials, network errors)

---

### **8.3 End-to-End Testing**

**Test complete workflows**:

#### **DHL E2E Test**:
1. Create JobOrder from quote
2. Open for review
3. Fill in all required details
4. Fetch rates (should show DHL rates)
5. Proceed to booking
6. Verify AWB generated
7. Verify ERP entry created
8. Verify email sent
9. Verify documents stored

#### **FedEx E2E Test**:
1. Create JobOrder from quote
2. Select "FEDEX" as agent
3. Fill in all required details (including state for US address)
4. Select "I will pay" for duties
5. Select "Sold" for shipment purpose
6. Fetch rates (should show FedEx rates)
7. Proceed to booking
8. Verify AWB generated
9. Verify ERP entry created
10. Verify email sent
11. Verify documents stored

**Verification Points**:
- [ ] DHL workflow unchanged (regression test)
- [ ] FedEx workflow works end-to-end
- [ ] UI shows carrier-appropriate terminology
- [ ] State/province required for FedEx US/CA/MX
- [ ] Duties payment shown as radio for FedEx
- [ ] All documents stored correctly
- [ ] ERP integration works for both carriers
- [ ] Email automation works for both carriers

---

### **8.4 Data Validation Testing**

**Test backward compatibility**:

#### **Test existing JobOrder records**:
```javascript
// Load existing DHL JobOrder (created before migration)
const existingJobOrder = await JobOrder.findById('existing-dhl-id');

console.log('Existing JobOrder fields:');
console.log('incoterm:', existingJobOrder.containerInfo?.incoterm);
console.log('exportReasonType:', existingJobOrder.containerInfo?.exportDeclaration?.exportReasonType);

// Test helper methods
console.log('getDutiesPaymentType():', existingJobOrder.getDutiesPaymentType());
console.log('getShipmentPurpose():', existingJobOrder.getShipmentPurpose());

// Verify auto-calculation on save
await existingJobOrder.save();
console.log('After save - dutiesPaymentType:', existingJobOrder.containerInfo?.dutiesPaymentType);
console.log('After save - shipmentPurpose:', existingJobOrder.containerInfo?.exportDeclaration?.shipmentPurpose);
```

**Expected Results**:
- [ ] Existing records load without errors
- [ ] Helper methods return correct values
- [ ] Pre-save hook calculates new fields
- [ ] No data loss
- [ ] No validation errors

---

### **Phase 8 Completion Checklist**:
- [ ] Unit tests written and passing
- [ ] FedEx authentication tested
- [ ] FedEx rate fetching tested
- [ ] FedEx shipment booking tested
- [ ] DHL regression tests passing
- [ ] End-to-end workflow tested for both carriers
- [ ] Backward compatibility verified
- [ ] Error scenarios tested
- [ ] Token refresh tested
- [ ] All test documentation updated

---

## Phase 9: Production Deployment

### **Objective**: Deploy to production with zero downtime

### **9.1 Pre-Deployment Checklist**

**Code Review**:
- [ ] All code reviewed and approved
- [ ] No console.log statements in production code
- [ ] All TODOs addressed or documented
- [ ] Code follows existing patterns
- [ ] Comments and documentation complete

**Configuration**:
- [ ] Production environment variables set
- [ ] FedEx production credentials configured
- [ ] DHL production credentials verified
- [ ] Database backup created
- [ ] Rollback plan documented

**Testing**:
- [ ] All tests passing in staging
- [ ] Performance testing completed
- [ ] Load testing completed (if applicable)
- [ ] Security scan passed

---

### **9.2 Deployment Steps**

#### **Step 1: Database Migration** (No-op migration)
```javascript
// migrations/add-carrier-agnostic-fields.js
// This is a no-op migration since fields have defaults
// Just verify schema changes applied

module.exports = {
    up: async (db) => {
        console.log('Adding carrier-agnostic fields...');
        console.log('Fields have defaults, no data migration needed');
        console.log('Pre-save hook will populate fields on next save');
        return Promise.resolve();
    },

    down: async (db) => {
        console.log('No rollback needed (fields are optional)');
        return Promise.resolve();
    }
};
```

#### **Step 2: Deploy Backend**
```bash
# 1. Pull latest code
git pull origin main

# 2. Install dependencies
npm install

# 3. Run migrations (if any)
npm run migrate

# 4. Restart server (with zero downtime)
pm2 reload backend-quotation-mgmt
```

#### **Step 3: Deploy Frontend**
```bash
# 1. Pull latest code
git pull origin main

# 2. Install dependencies
npm install

# 3. Build production bundle
npm run build

# 4. Deploy to server
# (specific steps depend on hosting setup)
```

#### **Step 4: Smoke Tests**
```bash
# Test DHL (existing functionality)
curl -X GET "https://api.example.com/job-orders/dhl/rates?..."

# Test FedEx (new functionality)
curl -X POST "https://api.example.com/job-orders/rates/FEDEX" -d '{...}'
```

---

### **9.3 Post-Deployment Verification**

**Immediate Checks** (within 1 hour):
- [ ] DHL rate fetching works
- [ ] DHL booking works
- [ ] FedEx authentication works
- [ ] FedEx rate fetching works
- [ ] FedEx booking works
- [ ] No errors in server logs
- [ ] No errors in client logs
- [ ] ERP integration works

**Monitoring** (first 24 hours):
- [ ] Monitor error rates
- [ ] Monitor API response times
- [ ] Monitor token refresh frequency
- [ ] Monitor success rates (DHL vs FedEx)
- [ ] Check user feedback

**First Week**:
- [ ] Track FedEx adoption rate
- [ ] Monitor any DHL regression issues
- [ ] Collect user feedback
- [ ] Performance metrics stable
- [ ] No critical bugs reported

---

### **9.4 Rollback Plan**

**If critical issues found**:

1. **Immediate Rollback** (Frontend only):
   ```bash
   # Revert frontend to previous version
   git revert HEAD
   npm run build
   # Deploy
   ```

2. **Full Rollback** (Backend + Frontend):
   ```bash
   # Backend
   git revert HEAD
   npm install
   pm2 reload backend-quotation-mgmt

   # Frontend
   git revert HEAD
   npm install
   npm run build
   # Deploy
   ```

3. **Database Rollback** (if needed):
   ```bash
   # Run down migration
   npm run migrate:down
   ```

---

### **Phase 9 Completion Checklist**:
- [ ] Code deployed to production
- [ ] Smoke tests passed
- [ ] Monitoring in place
- [ ] No critical errors
- [ ] DHL functionality verified
- [ ] FedEx functionality verified
- [ ] Team notified of deployment
- [ ] Documentation updated

---

## Verification Checklist

### **Final Verification - DHL (No Regression)**

**Frontend**:
- [ ] Incoterm dropdown shows for DHL
- [ ] Export reason dropdown shows for DHL
- [ ] Special instructions limited to 20 chars
- [ ] Address lines limited to 45 chars
- [ ] City name limited to 45 chars, no commas
- [ ] Item description limited to 70 chars
- [ ] Weekend pickups blocked
- [ ] Dimensions required
- [ ] State/province not shown for DHL

**Backend**:
- [ ] DHL rate fetching works
- [ ] DHL booking works
- [ ] AWB generation works
- [ ] Documents retrieved correctly
- [ ] ERP integration works
- [ ] Email automation works

---

### **Final Verification - FedEx (New Functionality)**

**Frontend**:
- [ ] "Who pays duties" radio buttons show for FedEx
- [ ] "Shipment purpose" dropdown shows for FedEx
- [ ] Incoterm hidden for FedEx
- [ ] Export reason hidden for FedEx
- [ ] State/province shows for US/CA/MX
- [ ] State/province required for US/CA/MX
- [ ] Address lines limited to 35 chars (2 lines)
- [ ] Item description limited to 50 chars
- [ ] No weekend restriction
- [ ] Dimensions optional

**Backend**:
- [ ] FedEx authentication works
- [ ] FedEx rate fetching works
- [ ] FedEx booking works
- [ ] AWB generation works
- [ ] Documents retrieved correctly
- [ ] ERP integration works
- [ ] Email automation works
- [ ] Token refresh works

---

### **Final Verification - Common**

**Data Model**:
- [ ] Existing JobOrders load correctly
- [ ] New fields auto-calculated
- [ ] Pre-save hook works
- [ ] Helper methods work
- [ ] No validation errors

**Architecture**:
- [ ] CarrierFactory working
- [ ] DHLCarrier working
- [ ] FedExCarrier working
- [ ] DHLTransformer working
- [ ] FedExTransformer working
- [ ] All services use CarrierFactory
- [ ] No hardcoded carrier logic in business layer

**Backward Compatibility**:
- [ ] Old API endpoints work
- [ ] Old DHL functions work
- [ ] Existing integrations work
- [ ] No breaking changes

---

## Success Criteria - Final Confirmation

✅ **DHL Functionality**: Operates exactly as before (zero regression)

✅ **FedEx Functionality**: Complete integration matching DHL capabilities

✅ **User Experience**: Carrier-appropriate terminology shown

✅ **Data Integrity**: All existing records work, new fields calculated

✅ **Architecture**: Clean carrier abstraction, easy to add new carriers

✅ **Testing**: All tests passing, no critical bugs

✅ **Documentation**: Complete and up-to-date

✅ **Production**: Deployed successfully with monitoring

---

## End of Implementation Roadmap

This comprehensive roadmap provides a complete, phase-by-phase implementation plan with:
- ✅ Exact accomplishments per phase
- ✅ No timeline assumptions
- ✅ Specific code implementations
- ✅ Verification checklists
- ✅ Zero breaking changes
- ✅ Full backward compatibility
- ✅ Carrier-specific UX
- ✅ Clean architecture

**Total Phases**: 9 phases (Infrastructure → Data Model → Transformers → Carriers → Frontend Config → Frontend UI → Services → Testing → Deployment)

**End State**: Both DHL and FedEx fully operational with best possible UX and maintainable, extensible architecture.
