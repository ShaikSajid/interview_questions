# Question 9: Data Transformation & Validation for Integration

## 🎯 Question
**How would you design a robust data transformation and validation framework for Bayer's integration platform? Discuss schema validation, data mapping, ETL patterns, data quality checks, and handling different data formats (JSON, XML, CSV, EDI).**

---

## 📋 Answer Overview

Data transformation challenges in pharmaceutical integration:
- **Multiple formats**: SAP (XML), Salesforce (JSON), legacy systems (CSV, EDI)
- **Schema validation**: Ensure data meets requirements before processing
- **Data mapping**: Transform source schema to target schema
- **Data quality**: Detect and handle invalid/incomplete data
- **Regulatory compliance**: Audit trail, data lineage, validation logs

Key patterns:
- **Schema-first design**: Define schemas for all data types
- **Validation pipeline**: Multi-stage validation (syntax → semantics → business rules)
- **Transformation layers**: Separate extraction, transformation, loading
- **Error handling**: Quarantine invalid records, notify data owners
- **Idempotency**: Safe to reprocess data multiple times

---

## 🏗️ Data Transformation Architecture

```
┌────────────────────────────────────────────────────────┐
│            Data Transformation Pipeline                │
└────────────────────────────────────────────────────────┘

Source Systems:
┌────────┐  ┌──────────┐  ┌─────────┐  ┌─────────┐
│  SAP   │  │Salesforce│  │ Legacy  │  │  CSV    │
│  (XML) │  │  (JSON)  │  │  (EDI)  │  │  Files  │
└───┬────┘  └────┬─────┘  └────┬────┘  └────┬────┘
    │            │             │            │
    └────────────┼─────────────┼────────────┘
                 │             │
                 ▼             ▼
        ┌─────────────────────────┐
        │   1. Extract & Parse    │
        │   - Parse format        │
        │   - Initial validation  │
        └──────────┬──────────────┘
                   │
                   ▼
        ┌─────────────────────────┐
        │   2. Schema Validation  │
        │   - JSON Schema         │
        │   - Required fields     │
        │   - Data types          │
        └──────────┬──────────────┘
                   │
                   ▼
        ┌─────────────────────────┐
        │   3. Data Transformation│
        │   - Field mapping       │
        │   - Value conversion    │
        │   - Enrichment          │
        └──────────┬──────────────┘
                   │
                   ▼
        ┌─────────────────────────┐
        │   4. Business Rules     │
        │   - Custom validation   │
        │   - Regulatory checks   │
        │   - Data quality        │
        └──────────┬──────────────┘
                   │
       ┌───────────┴──────────┐
       │                      │
       ▼                      ▼
┌──────────────┐      ┌──────────────┐
│Valid Records │      │Invalid Records│
│→ Target DB   │      │→ DLQ + Alert │
└──────────────┘      └──────────────┘
```

---

## 💻 Implementation Examples

### 1. JSON Schema Validation

```javascript
// schema-validation.js
// WHY: Validate data structure before processing
// PREVENTS: Invalid data causing downstream failures

const Ajv = require('ajv');
const addFormats = require('ajv-formats');

const ajv = new Ajv({
  allErrors: true,       // Return all errors, not just first
  removeAdditional: true, // Remove fields not in schema
  useDefaults: true,     // Apply default values
  coerceTypes: true      // Convert types (e.g., "123" → 123)
});
addFormats(ajv);  // Add date, email, url formats

/**
 * Patient record schema
 * WHY: Define data contract for patient data
 * COMPLIANCE: FDA 21 CFR Part 11 requires data integrity
 */
const patientSchema = {
  $schema: 'http://json-schema.org/draft-07/schema#',
  type: 'object',
  required: ['patientId', 'firstName', 'lastName', 'dateOfBirth'],
  properties: {
    patientId: {
      type: 'string',
      pattern: '^PAT[0-9]{7}$',  // PAT followed by 7 digits
      description: 'Unique patient identifier'
      // WHY: Enforce ID format for consistency
    },
    firstName: {
      type: 'string',
      minLength: 1,
      maxLength: 50,
      pattern: '^[A-Za-z\\s-]+$'  // Letters, spaces, hyphens only
      // WHY: Prevent injection attacks, ensure data quality
    },
    lastName: {
      type: 'string',
      minLength: 1,
      maxLength: 50
    },
    dateOfBirth: {
      type: 'string',
      format: 'date',  // YYYY-MM-DD
      // WHY: Standard date format for interoperability
    },
    email: {
      type: 'string',
      format: 'email',
      description: 'Contact email'
      // Optional field (not in required array)
    },
    phoneNumber: {
      type: 'string',
      pattern: '^\\+?[1-9]\\d{1,14}$',  // E.164 format
      description: 'International phone number'
    },
    address: {
      type: 'object',
      required: ['street', 'city', 'state', 'zipCode'],
      properties: {
        street: { type: 'string' },
        city: { type: 'string' },
        state: {
          type: 'string',
          pattern: '^[A-Z]{2}$'  // Two-letter state code
        },
        zipCode: {
          type: 'string',
          pattern: '^\\d{5}(-\\d{4})?$'  // 12345 or 12345-6789
        }
      }
    },
    medications: {
      type: 'array',
      items: {
        type: 'object',
        required: ['drugName', 'dosage'],
        properties: {
          drugName: { type: 'string' },
          dosage: { type: 'string' },
          frequency: { type: 'string' }
        }
      }
    },
    consentDate: {
      type: 'string',
      format: 'date-time',  // ISO 8601
      description: 'HIPAA consent timestamp'
      // WHY: Track consent for regulatory compliance
    }
  },
  additionalProperties: false  // Reject unknown fields
  // WHY: Prevent data leakage, ensure schema compliance
};

/**
 * Validate data against schema
 * WHY: Catch invalid data before processing
 */
function validatePatient(data) {
  const validate = ajv.compile(patientSchema);
  const valid = validate(data);
  
  if (!valid) {
    // Format validation errors
    const errors = validate.errors.map(err => ({
      field: err.instancePath,
      message: err.message,
      params: err.params
    }));
    
    logger.warn('Patient validation failed', {
      errors,
      data: sanitizeForLogging(data)
    });
    
    throw new ValidationError('Invalid patient data', { errors });
  }
  
  logger.info('Patient validation succeeded', {
    patientId: data.patientId
  });
  
  return data;  // Return validated (and possibly coerced) data
}

/**
 * Clinical trial enrollment schema
 * WHY: Ensure trial data meets regulatory requirements
 */
const trialEnrollmentSchema = {
  type: 'object',
  required: ['trialId', 'patientId', 'siteId', 'enrollmentDate', 'consentForm'],
  properties: {
    trialId: {
      type: 'string',
      pattern: '^TRIAL[0-9]{6}$'
    },
    patientId: {
      type: 'string',
      pattern: '^PAT[0-9]{7}$'
    },
    siteId: {
      type: 'string',
      pattern: '^SITE[0-9]{4}$',
      description: 'Clinical trial site'
    },
    enrollmentDate: {
      type: 'string',
      format: 'date'
    },
    consentForm: {
      type: 'object',
      required: ['signed', 'signatureDate', 'version'],
      properties: {
        signed: {
          type: 'boolean',
          const: true  // Must be true to enroll
          // WHY: HIPAA requires informed consent
        },
        signatureDate: {
          type: 'string',
          format: 'date-time'
        },
        version: {
          type: 'string',
          description: 'Consent form version'
          // WHY: Track which consent version signed
        }
      }
    },
    eligibility: {
      type: 'object',
      required: ['ageEligible', 'medicallyEligible'],
      properties: {
        ageEligible: { type: 'boolean' },
        medicallyEligible: { type: 'boolean' },
        notes: { type: 'string' }
      }
    }
  }
};
```

---

### 2. Data Transformation (Field Mapping)

```javascript
// data-transformation.js
// WHY: Transform source schema to target schema
// EXAMPLE: SAP → Salesforce field mapping

const { JSONPath } = require('jsonpath-plus');

/**
 * Field mapping configuration
 * WHY: Declarative mapping, easy to maintain
 */
const sapToSalesforceMapping = {
  // Simple field mapping
  'Account.Name': 'KUNNR',  // Customer number → Account name
  'Account.BillingStreet': 'STRAS',  // Street
  'Account.BillingCity': 'ORT01',    // City
  'Account.BillingPostalCode': 'PSTLZ',  // Postal code
  
  // Nested field mapping (JSONPath)
  'Contact.FirstName': '$.NAME1',
  'Contact.LastName': '$.NAME2',
  
  // Computed field (function)
  'Account.Industry': (source) => {
    const industryCode = source.BRSCH;  // Industry code
    return mapIndustryCode(industryCode);
  },
  
  // Conditional mapping
  'Account.Type': (source) => {
    return source.KTOKD === 'KUNA' ? 'Customer' : 'Prospect';
  },
  
  // Date transformation
  'Account.CreatedDate': (source) => {
    // SAP date: YYYYMMDD → ISO 8601
    const sapDate = source.ERDAT;
    return `${sapDate.substring(0,4)}-${sapDate.substring(4,6)}-${sapDate.substring(6,8)}T00:00:00Z`;
  }
};

/**
 * Transform data using mapping configuration
 * WHY: Reusable transformation engine
 */
function transformData(sourceData, mapping) {
  const result = {};
  
  for (const [targetField, sourceField] of Object.entries(mapping)) {
    try {
      let value;
      
      if (typeof sourceField === 'function') {
        // Computed field
        value = sourceField(sourceData);
      } else if (sourceField.startsWith('$.')) {
        // JSONPath expression
        const matches = JSONPath({ path: sourceField, json: sourceData });
        value = matches[0];
      } else {
        // Simple field reference
        value = sourceData[sourceField];
      }
      
      // Set nested field (e.g., "Account.Name" → result.Account.Name)
      setNestedValue(result, targetField, value);
      
    } catch (error) {
      logger.error('Field transformation error', {
        targetField,
        sourceField,
        error: error.message
      });
      // Continue with other fields
    }
  }
  
  return result;
}

/**
 * Set nested object value
 * WHY: Support "Account.Name" syntax
 */
function setNestedValue(obj, path, value) {
  const parts = path.split('.');
  let current = obj;
  
  for (let i = 0; i < parts.length - 1; i++) {
    const part = parts[i];
    if (!current[part]) {
      current[part] = {};
    }
    current = current[part];
  }
  
  current[parts[parts.length - 1]] = value;
}

/**
 * Industry code mapping
 * WHY: SAP uses numeric codes, Salesforce uses names
 */
const industryCodeMap = {
  '01': 'Pharmaceuticals',
  '02': 'Biotechnology',
  '03': 'Medical Devices',
  '04': 'Healthcare',
  '99': 'Other'
};

function mapIndustryCode(code) {
  return industryCodeMap[code] || 'Other';
}

/**
 * Complex transformation: SAP to Salesforce
 */
async function transformSAPToSalesforce(sapData) {
  // 1. Extract customer data
  const customer = sapData.CUSTOMER[0];
  
  // 2. Apply field mapping
  const salesforceAccount = transformData(customer, sapToSalesforceMapping);
  
  // 3. Enrich with additional data
  salesforceAccount.Account.Source = 'SAP';
  salesforceAccount.Account.LastModifiedDate = new Date().toISOString();
  
  // 4. Validate target schema
  validateSalesforceAccount(salesforceAccount);
  
  // 5. Return transformed data
  return salesforceAccount;
}
```

---

### 3. Multi-Format Parser

```javascript
// format-parser.js
// WHY: Handle multiple data formats (JSON, XML, CSV, EDI)

const xml2js = require('xml2js');
const csv = require('csv-parser');
const { Readable } = require('stream');

/**
 * Parse data based on content type
 * WHY: Unified interface for all formats
 */
async function parseData(data, contentType) {
  switch (contentType) {
    case 'application/json':
      return parseJSON(data);
    
    case 'application/xml':
    case 'text/xml':
      return parseXML(data);
    
    case 'text/csv':
      return parseCSV(data);
    
    case 'application/edi':
      return parseEDI(data);
    
    default:
      throw new Error(`Unsupported content type: ${contentType}`);
  }
}

/**
 * Parse JSON
 * WHY: Catch JSON syntax errors early
 */
function parseJSON(data) {
  try {
    return JSON.parse(data);
  } catch (error) {
    throw new ValidationError('Invalid JSON', {
      error: error.message,
      position: error.position
    });
  }
}

/**
 * Parse XML
 * WHY: Convert XML to JSON for processing
 */
async function parseXML(data) {
  const parser = new xml2js.Parser({
    explicitArray: false,  // Don't wrap single elements in array
    mergeAttrs: true,      // Merge attributes with elements
    trim: true             // Trim whitespace
  });
  
  try {
    const result = await parser.parseStringPromise(data);
    return result;
  } catch (error) {
    throw new ValidationError('Invalid XML', {
      error: error.message,
      line: error.line,
      column: error.column
    });
  }
}

/**
 * Parse CSV
 * WHY: Convert CSV rows to JSON objects
 */
async function parseCSV(data) {
  return new Promise((resolve, reject) => {
    const rows = [];
    
    Readable.from(data)
      .pipe(csv())
      .on('data', (row) => {
        rows.push(row);
      })
      .on('end', () => {
        resolve(rows);
      })
      .on('error', (error) => {
        reject(new ValidationError('Invalid CSV', {
          error: error.message
        }));
      });
  });
}

/**
 * Parse EDI (Electronic Data Interchange)
 * WHY: Legacy systems use EDI format
 * EXAMPLE: EDI 850 (Purchase Order)
 */
function parseEDI(data) {
  // Simplified EDI parser (production would use library)
  const segments = data.split('~');  // Segment terminator
  const result = {};
  
  for (const segment of segments) {
    const elements = segment.split('*');  // Element separator
    const segmentId = elements[0];
    
    switch (segmentId) {
      case 'ISA':  // Interchange Control Header
        result.interchangeControlNumber = elements[13];
        break;
      
      case 'GS':   // Functional Group Header
        result.functionalGroupNumber = elements[6];
        break;
      
      case 'ST':   // Transaction Set Header
        result.transactionSetId = elements[1];
        break;
      
      case 'PO1':  // Purchase Order Line Item
        if (!result.lineItems) result.lineItems = [];
        result.lineItems.push({
          quantity: elements[2],
          unitPrice: elements[4],
          productCode: elements[7]
        });
        break;
    }
  }
  
  return result;
}
```

---

### 4. Data Quality Checks

```javascript
// data-quality.js
// WHY: Ensure data meets business rules and quality standards

/**
 * Data quality checker
 * WHY: Detect and report data quality issues
 */
class DataQualityChecker {
  constructor() {
    this.rules = [];
  }
  
  /**
   * Add quality rule
   */
  addRule(name, checkFn) {
    this.rules.push({ name, checkFn });
  }
  
  /**
   * Check data quality
   * WHY: Run all quality checks, report issues
   */
  async check(data) {
    const issues = [];
    
    for (const rule of this.rules) {
      try {
        const result = await rule.checkFn(data);
        
        if (!result.passed) {
          issues.push({
            rule: rule.name,
            severity: result.severity || 'ERROR',
            message: result.message,
            field: result.field,
            value: result.value
          });
        }
        
      } catch (error) {
        logger.error('Quality check failed', {
          rule: rule.name,
          error: error.message
        });
      }
    }
    
    return {
      passed: issues.length === 0,
      issues
    };
  }
}

// Configure quality checks
const patientQualityChecker = new DataQualityChecker();

// Rule 1: Age validation
patientQualityChecker.addRule('age-validation', (data) => {
  const age = calculateAge(data.dateOfBirth);
  
  if (age < 18 || age > 120) {
    return {
      passed: false,
      severity: 'ERROR',
      message: 'Invalid age',
      field: 'dateOfBirth',
      value: data.dateOfBirth
    };
  }
  
  return { passed: true };
});

// Rule 2: Duplicate detection
patientQualityChecker.addRule('duplicate-check', async (data) => {
  const existing = await findPatientBySSN(data.ssn);
  
  if (existing) {
    return {
      passed: false,
      severity: 'WARNING',
      message: 'Potential duplicate patient',
      field: 'ssn',
      value: data.ssn
    };
  }
  
  return { passed: true };
});

// Rule 3: Completeness check
patientQualityChecker.addRule('completeness', (data) => {
  const requiredFields = ['firstName', 'lastName', 'dateOfBirth', 'email'];
  const missing = requiredFields.filter(field => !data[field]);
  
  if (missing.length > 0) {
    return {
      passed: false,
      severity: 'ERROR',
      message: 'Missing required fields',
      field: missing.join(', ')
    };
  }
  
  return { passed: true };
});

/**
 * Usage: Validate patient data quality
 */
async function validatePatientQuality(patientData) {
  const result = await patientQualityChecker.check(patientData);
  
  if (!result.passed) {
    logger.warn('Data quality issues detected', {
      patientId: patientData.patientId,
      issues: result.issues
    });
    
    // Quarantine record for manual review
    await quarantineRecord(patientData, result.issues);
    
    throw new DataQualityError('Data quality check failed', {
      issues: result.issues
    });
  }
  
  return true;
}
```

---

## 📊 Data Transformation Best Practices

### ✅ Do's
1. **Validate before transformation** (fail fast)
2. **Use schema-first design** (define contracts)
3. **Log transformation errors** (with context)
4. **Quarantine invalid records** (don't discard)
5. **Make transformations idempotent** (safe to retry)
6. **Version schemas** (track changes)
7. **Document field mappings** (business rules)
8. **Test edge cases** (null, empty, invalid)

### ❌ Don'ts
1. **Don't ignore validation errors** (causes downstream issues)
2. **Don't lose data** (preserve original if transformation fails)
3. **Don't hardcode mappings** (use configuration)
4. **Don't skip data quality checks** (garbage in, garbage out)
5. **Don't transform in place** (create new object)

---

## 🎯 Key Takeaways

**Data transformation requires**:
- ✅ **Schema validation**: Catch errors early
- ✅ **Field mapping**: Declarative configuration
- ✅ **Data quality**: Business rule validation
- ✅ **Error handling**: Quarantine, don't discard
- ✅ **Audit trail**: Track transformations for compliance

---

**Estimated Reading Time**: 16-19 minutes  
**Difficulty Level**: ⭐⭐⭐⭐ Advanced  
**Prerequisites**: Data formats, JSON Schema, ETL  
**Bayer Job Alignment**: 100% - Critical integration skill
