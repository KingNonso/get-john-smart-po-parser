# Product Requirements Document: Smart PO Parser

## Smart PO Parser - Azure Functions Microservice

**Version:** 1.0  
**Date:** January 2026  
**Status:** In Development (Customer Roadmap)  
**Owner:** John O'Rourke / Get John & Co

---

## Executive Summary

The **Smart PO Parser** is a Python microservice deployed on Azure Functions that automates the extraction of Purchase Order (PO) data from non-standard, unstructured formats (scanned PDFs, email attachments, images) and converts them into structured cXML/JSON that integrates seamlessly with Get John's Punchout system.

**Problem Statement:**  
B2B buyers submit POs in a variety of formats (hand-scanned PDFs with variable quality, email forwarded documents, vendor-specific forms). Today, these require manual data entry by Get John staff or customers, creating delays and transcription errors. This blocks customer workflows and inflates support burden.

**Solution:**  
A serverless microservice using Tesseract OCR (text extraction) + Python heuristics (field parsing) that:
- Ingests POs as PDFs, images, or email attachments
- Extracts key fields (PO number, line items, quantities, prices, buyer/supplier details)
- Validates data against business rules
- Outputs clean cXML or JSON ready for Punchout integration
- Flags ambiguous results for human review with high-confidence extraction scores

**Business Impact:**
- **Reduce manual data entry time** by 80% for messy POs
- **Accelerate buyer-supplier workflows** (faster order processing)
- **Decrease support tickets** related to "did you receive my PO?"
- **Differentiator vs. competitors** (Ariba, SAP do not offer lightweight OCR PO parsing)

---

## Goals & Success Metrics

### Primary Goals
1. **Accuracy:** 95%+ extraction accuracy on standard PO formats; 75%+ on complex/handwritten POs
2. **Latency:** Process a typical 2-page PO in <10 seconds (end-to-end)
3. **Scalability:** Handle 1000+ POs/day without performance degradation
4. **User Experience:** Reduce PO submission-to-system entry time from 5 minutes to <30 seconds
5. **Reliability:** 99.5% uptime; graceful failures with fallback to manual review

### Success Metrics
| Metric | Target | Measurement |
|--------|--------|-------------|
| **Extraction Accuracy** | 95% for standard formats | Confidence score >90% |
| **Processing Speed** | <10 seconds per PO | CloudWatch logs |
| **False Positive Rate** | <5% | Manual QA samples |
| **User Adoption** | >60% of POs submitted via parser within 6 months | Usage analytics |
| **System Uptime** | 99.5% | Azure Monitor alerts |
| **Cost per PO** | <$0.01 | Azure usage metrics |

---

## Scope

### In Scope
- ✅ PDF-to-text extraction using Tesseract OCR
- ✅ Image (JPEG/PNG) processing for scanned POs
- ✅ Extraction of critical fields: PO number, buyer/supplier names, line items (sku/qty/price), dates
- ✅ Validation against cXML schema requirements
- ✅ Output as structured JSON + optional cXML generation
- ✅ Confidence scoring for each extracted field
- ✅ Fallback to manual review queue for low-confidence extractions
- ✅ Integration with Get John's existing database (store parsing results + audit trail)
- ✅ Webhook/API endpoint to receive PO submissions
- ✅ Email forwarding integration (MIME parsing, PDF attachment extraction)
- ✅ Logging, monitoring, error tracking

### Out of Scope
- ❌ Handwriting recognition (beyond Tesseract's basic capability)
- ❌ Custom training of Tesseract for Get John-specific PO formats (Phase 2)
- ❌ Real-time invoice matching or PO-to-Receipt reconciliation
- ❌ Multi-language support (English only, Phase 2)
- ❌ Mobile app UI for manual corrections (Web UI only, Phase 2)
- ❌ Integration with downstream accounting/ERP systems (Xero, QuickBooks)

---

## Technical Architecture

### High-Level Flow

```
┌─────────────────────┐
│   PO Input Sources  │
├─────────────────────┤
│ • Email (MIME)      │
│ • Web Upload (PDF)  │
│ • Image (PNG/JPEG)  │
│ • Webhook (binary)  │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────────────────────────────┐
│   Azure Function: parse_po_handler()        │
├─────────────────────────────────────────────┤
│ 1. Input validation (file size, MIME type) │
│ 2. Queue to Blob Storage                   │
└────────────┬────────────────────────────────┘
             │
             ▼
┌────────────────────────────────────────────────────┐
│   Azure Function: po_parsing_processor() (async)  │
├────────────────────────────────────────────────────┤
│ 1. Download file from Blob Storage               │
│ 2. Pre-process (deskew, denoise, resize)         │
│ 3. Extract text using Tesseract OCR              │
│ 4. Parse structured fields (heuristics)          │
│ 5. Validate against schema                       │
│ 6. Score confidence for each field               │
│ 7. Store in Database                             │
│ 8. Generate cXML/JSON output                     │
└────────────┬─────────────────────────────────────┘
             │
             ▼
┌──────────────────────────────────────────┐
│    Decision Logic: Confidence Check      │
├──────────────────────────────────────────┤
│ IF all fields >90% confidence:           │
│   → Auto-accept, emit webhook to Buyer  │
│ ELSE:                                    │
│   → Flag for manual review queue         │
│   → Notify Get John staff via Slack      │
└──────────────────────────────────────────┘
             │
             ▼
┌───────────────────────────────────┐
│   Output Storage & Delivery       │
├───────────────────────────────────┤
│ • DB: Parsed result + metadata   │
│ • Blob: Raw + preprocessed image │
│ • Webhook: To buyer's system     │
│ • Audit log: All transformations │
└───────────────────────────────────┘
```

### Technology Stack

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| **OCR Engine** | Tesseract 5.x (pytesseract) | Open-source, 95%+ accuracy on clean docs, lightweight, no licensing |
| **Language** | Python 3.11+ | Async support, rich data science libraries (pandas), Azure SDK maturity |
| **Compute** | Azure Functions (Consumption Plan) | Serverless, auto-scales, <$1/1M requests |
| **Storage** | Azure Blob Storage | Cheap, reliable, integrates with Functions |
| **Database** | PostgreSQL (existing Get John DB) | Existing infrastructure, JSONB support for parsed results |
| **Message Queue** | Azure Service Bus or Queue Storage | Async processing, guaranteed delivery, dead-letter handling |
| **Monitoring** | Azure Monitor + Application Insights | Cost-effective, native Azure integration |
| **Container Runtime** | Docker (for local dev + testing) | Consistency across environments |

### Deployment Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Azure Deployment                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────────────┐         ┌──────────────────────────┐ │
│  │  HTTP Trigger Func   │         │  Queue Trigger Func      │ │
│  │  (parse_po_handler)  │────┐    │  (po_parsing_processor)  │ │
│  │                      │    │    │                          │ │
│  │ • File upload        │    │    │ • Long-running OCR       │ │
│  │ • Validation         │    │    │ • Parsing + validation   │ │
│  │ • Queue message      │    │    │ • DB storage             │ │
│  └──────────────────────┘    │    └──────────────────────────┘ │
│                              │                                  │
│                              ▼                                  │
│                  ┌────────────────────────┐                    │
│                  │  Azure Queue Storage   │                    │
│                  │  (or Service Bus)      │                    │
│                  └────────────────────────┘                    │
│                              │                                  │
│              ┌───────────────┼───────────────┐                 │
│              ▼               ▼               ▼                 │
│        ┌──────────┐   ┌──────────┐   ┌──────────┐             │
│        │  Blob    │   │ Postgres │   │ App      │             │
│        │ Storage  │   │   DB     │   │ Insights │             │
│        └──────────┘   └──────────┘   └──────────┘             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Core Features & Requirements

### Feature 1: Multi-Format Input Handling

**Requirement:**
- Accept PO submissions via:
  1. **HTTP POST** with multipart/form-data (PDF, PNG, JPEG)
  2. **Email forwarding** (extract PDF attachment from MIME)
  3. **Blob Storage trigger** (batch processing)
  4. **Webhook** (from buyer's procurement system)

**Technical Specs:**
- Max file size: 10 MB per PO
- Supported formats: PDF (raster + vector), PNG, JPEG
- Validate MIME type on upload
- Normalize all formats to standard image for OCR (300+ DPI recommended)

**Implementation:**
```python
# Pseudo-code: Input Handler
@app.route('/api/po/upload', methods=['POST'])
async def upload_po(req: func.HttpRequest) -> func.HttpResponse:
    file = req.files.get('file')
    
    # Validate
    if not file or file.size > 10 * 1024 * 1024:
        return func.HttpResponse("File too large", status_code=400)
    
    # Save to Blob
    blob_name = f"{uuid.uuid4()}.pdf"
    blob_client = BlobServiceClient(...).get_blob_client(blob_name)
    blob_client.upload_blob(file.stream)
    
    # Queue for processing
    queue_client.send_message(blob_name)
    
    return func.HttpResponse(f"Job queued: {blob_name}", status_code=202)
```

---

### Feature 2: OCR Text Extraction with Preprocessing

**Requirement:**
Extract text from scanned PO images with >90% accuracy on standard documents.

**Problem:** Raw scanned PDFs often have:
- Skewed text (rotation)
- Low contrast
- Noise and artifacts
- Inconsistent resolution

**Solution: Preprocessing Pipeline**

Using OpenCV + Pillow:

1. **Convert to grayscale** - Reduce noise
2. **Deskew** - Correct rotation using line detection
3. **Denoise** - Morphological operations (erosion/dilation)
4. **Threshold** - Binary conversion (OTSU method) for clarity
5. **Upscale** - Resize to 300+ DPI if needed (Lanczos resampling)
6. **Pass to Tesseract** - Run OCR with LSTM engine mode (OEM 1)

**Tesseract Configuration:**
```python
import pytesseract
from PIL import Image, ImageOps, ImageFilter, ImageEnhance
import cv2
import numpy as np

def preprocess_and_extract_text(image_path):
    # Load image
    img = cv2.imread(image_path, cv2.IMREAD_GRAYSCALE)
    
    # Deskew using Hough Line Transform
    angle = detect_skew(img)
    if abs(angle) > 0.5:
        img = rotate_image(img, angle)
    
    # Denoise
    img = cv2.medianBlur(img, 5)
    
    # Threshold (OTSU)
    _, img = cv2.threshold(img, 0, 255, cv2.THRESH_BINARY + cv2.THRESH_OTSU)
    
    # Upscale if DPI is low
    if estimate_dpi(img) < 300:
        img = cv2.resize(img, None, fx=2, fy=2, interpolation=cv2.INTER_LANCZOS4)
    
    # OCR with Tesseract (LSTM engine, multiple PSM)
    config = '--oem 1 --psm 1 -c tessedit_char_whitelist=0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz,.-/$'
    
    text = pytesseract.image_to_string(img, config=config)
    data = pytesseract.image_to_data(img, output_type=pytesseract.Output.DICT)
    
    return text, data  # data includes confidence scores per word
```

**Confidence Scoring:**
- Tesseract's `image_to_data()` returns per-word confidence (0-100)
- Average confidence for extracted text: `mean([conf for conf in confidences if conf > 0])`
- Flag text blocks <75% confidence for manual review

---

### Feature 3: Structured Field Extraction (Heuristics + Regex)

**Requirement:**
Parse raw OCR text and extract key PO fields into structured JSON.

**Target Fields:**

| Field | Type | Regex / Heuristic | Priority |
|-------|------|-------------------|----------|
| **PO Number** | String | `PO[#-]?\d{6,12}`, invoice/order number patterns | High |
| **PO Date** | ISO Date | `\d{1,2}[-/]\d{1,2}[-/]\d{2,4}` (US/UK formats) | High |
| **Buyer Name** | String | First mention of company name in header | High |
| **Buyer Address** | String | Street, city, postal code block detection | High |
| **Supplier Name** | String | "Ship to", "Bill to", "Supplier" sections | High |
| **Line Items** | Array | Tables with SKU, Description, Qty, Unit Price | High |
| **Total Amount** | Decimal | Search for "Total", "Amount Due" + currency symbol | High |
| **Currency** | String | `$`, `£`, `€`, `USD`, `GBP` symbols | Medium |
| **Payment Terms** | String | "Net 30", "2/10 Net 30" patterns | Medium |
| **Delivery Date** | ISO Date | "Delivery", "Due Date" sections | Medium |

**Implementation: Field Parser Class**

```python
import re
from datetime import datetime
from typing import Dict, List, Optional

class POFieldParser:
    def __init__(self, raw_text: str, ocr_data: dict):
        self.text = raw_text
        self.ocr_data = ocr_data  # word-level confidence from Tesseract
        self.confidence_map = {}  # Track per-field confidence
    
    def extract_po_number(self) -> Dict[str, any]:
        """Extract PO number with confidence score."""
        patterns = [
            r'(?:PO|Order|Invoice)[\s#\-]*(\d{6,12})',
            r'(?:PO|ORDER)[\s#]?:?\s*(\d{6,12})',
            r'^\s*(\d{6,12})\s*$'  # Standalone number on line
        ]
        
        for pattern in patterns:
            match = re.search(pattern, self.text, re.IGNORECASE | re.MULTILINE)
            if match:
                po_num = match.group(1)
                confidence = self._score_field_confidence(match.group(0))
                return {
                    'value': po_num,
                    'confidence': confidence,
                    'raw': match.group(0)
                }
        
        return {'value': None, 'confidence': 0, 'raw': None}
    
    def extract_date(self, label: str = "Date") -> Dict[str, any]:
        """Extract date fields (PO Date, Delivery Date, etc.)."""
        patterns = [
            r'(\d{1,2})/(\d{1,2})/(\d{2,4})',  # MM/DD/YYYY
            r'(\d{1,2})-(\d{1,2})-(\d{2,4})',  # MM-DD-YYYY
            r'(\d{1,2})\s+(?:Jan|Feb|Mar|Apr|May|Jun|Jul|Aug|Sep|Oct|Nov|Dec)[a-z]*[\s,]*(\d{2,4})',  # 15 Jan 2025
        ]
        
        for pattern in patterns:
            match = re.search(pattern, self.text, re.IGNORECASE)
            if match:
                try:
                    date_str = self._parse_date_match(match)
                    confidence = self._score_field_confidence(match.group(0))
                    return {
                        'value': date_str,
                        'confidence': confidence,
                        'raw': match.group(0)
                    }
                except ValueError:
                    continue
        
        return {'value': None, 'confidence': 0, 'raw': None}
    
    def extract_line_items(self) -> List[Dict]:
        """Extract line items (SKU, Description, Qty, Price)."""
        # Heuristic: Look for table-like structures
        # Pattern: SKU/ID, Description, Quantity, Unit Price, Line Total
        
        lines = self.text.split('\n')
        items = []
        
        # Detect table boundaries (lines with 4+ numeric/alphanumeric fields)
        in_table = False
        for line in lines:
            if re.search(r'\d+\s+\d+\s+[\d.,]+\s+[\d.,]+', line):  # Qty + Price pattern
                in_table = True
                # Parse line
                fields = re.split(r'\s{2,}', line.strip())  # Multi-space delimiter
                if len(fields) >= 3:
                    items.append({
                        'sku': fields[0] if len(fields) > 0 else None,
                        'description': fields[1] if len(fields) > 1 else None,
                        'quantity': self._parse_number(fields[2] if len(fields) > 2 else ''),
                        'unit_price': self._parse_number(fields[3] if len(fields) > 3 else ''),
                        'confidence': self._score_field_confidence(line)
                    })
            elif in_table and len(line.strip()) == 0:
                in_table = False
        
        return items
    
    def extract_total_amount(self) -> Dict[str, any]:
        """Extract total order amount."""
        pattern = r'(?:Total|Amount\s+Due|Grand\s+Total)[\s:]*([£$€]?[\s]?[\d,]+\.?\d{0,2})'
        match = re.search(pattern, self.text, re.IGNORECASE)
        
        if match:
            amount_str = match.group(1)
            amount = self._parse_currency(amount_str)
            confidence = self._score_field_confidence(match.group(0))
            return {
                'value': amount,
                'confidence': confidence,
                'raw': amount_str,
                'currency': self._extract_currency(amount_str)
            }
        
        return {'value': None, 'confidence': 0, 'raw': None, 'currency': 'USD'}
    
    def _score_field_confidence(self, matched_text: str) -> float:
        """Score confidence based on Tesseract word confidence."""
        # Look up confidence scores for words in matched text
        words = matched_text.split()
        confidences = []
        
        for word in words:
            # Match word against OCR data
            for i, ocr_word in enumerate(self.ocr_data.get('text', [])):
                if ocr_word.lower() == word.lower():
                    confidences.append(int(self.ocr_data['conf'][i]))
        
        return round(sum(confidences) / len(confidences), 1) if confidences else 50.0
    
    def _parse_number(self, text: str) -> Optional[float]:
        """Parse numeric values, handling commas/decimals."""
        try:
            return float(re.sub(r'[^\d.]', '', text))
        except:
            return None
    
    def _parse_currency(self, text: str) -> Optional[float]:
        """Extract numeric value from currency string."""
        match = re.search(r'[\d,]+\.?\d{0,2}', text)
        return float(match.group().replace(',', '')) if match else None
    
    def _extract_currency(self, text: str) -> str:
        """Detect currency code from symbol."""
        symbols = {'$': 'USD', '£': 'GBP', '€': 'EUR'}
        for sym, code in symbols.items():
            if sym in text:
                return code
        return 'USD'

# Usage:
parser = POFieldParser(ocr_text, tesseract_ocr_data)
po_data = {
    'po_number': parser.extract_po_number(),
    'po_date': parser.extract_date('PO Date'),
    'line_items': parser.extract_line_items(),
    'total_amount': parser.extract_total_amount(),
}
```

---

### Feature 4: Validation & Error Handling

**Requirement:**
Validate parsed PO data against cXML schema and business rules.

**Validation Rules:**

```python
class POValidator:
    """Validate parsed PO against cXML requirements + business logic."""
    
    def validate(self, po_data: dict) -> Dict[str, any]:
        errors = []
        warnings = []
        
        # Rule 1: PO Number is mandatory
        if not po_data.get('po_number', {}).get('value'):
            errors.append("PO number is required")
        elif po_data['po_number']['confidence'] < 70:
            warnings.append(f"PO number confidence low ({po_data['po_number']['confidence']}%)")
        
        # Rule 2: PO Date must be valid and not in future
        if not po_data.get('po_date', {}).get('value'):
            errors.append("PO date is required")
        else:
            try:
                po_date = datetime.fromisoformat(po_data['po_date']['value'])
                if po_date > datetime.now():
                    errors.append("PO date cannot be in the future")
            except:
                errors.append("Invalid PO date format")
        
        # Rule 3: At least one line item
        line_items = po_data.get('line_items', [])
        if len(line_items) == 0:
            errors.append("PO must contain at least one line item")
        else:
            for i, item in enumerate(line_items):
                if not item.get('quantity'):
                    errors.append(f"Line item {i+1}: quantity missing")
                if not item.get('unit_price'):
                    errors.append(f"Line item {i+1}: unit price missing")
                if item.get('confidence', 0) < 60:
                    warnings.append(f"Line item {i+1}: low confidence ({item['confidence']}%)")
        
        # Rule 4: Total amount should match line items sum
        calculated_total = sum([
            (item.get('quantity', 0) * item.get('unit_price', 0))
            for item in line_items
        ])
        
        reported_total = po_data.get('total_amount', {}).get('value', 0)
        if reported_total and abs(calculated_total - reported_total) > 0.01:
            warnings.append(f"Total mismatch: calculated {calculated_total}, reported {reported_total}")
        
        # Rule 5: Buyer and Supplier must be identified
        if not po_data.get('buyer_name', {}).get('value'):
            warnings.append("Buyer name not clearly identified")
        if not po_data.get('supplier_name', {}).get('value'):
            warnings.append("Supplier name not clearly identified")
        
        return {
            'is_valid': len(errors) == 0,
            'errors': errors,
            'warnings': warnings,
            'overall_confidence': self._calculate_overall_confidence(po_data)
        }
    
    def _calculate_overall_confidence(self, po_data: dict) -> float:
        """Weighted average of field confidences."""
        weights = {
            'po_number': 0.25,
            'po_date': 0.15,
            'buyer_name': 0.15,
            'supplier_name': 0.15,
            'total_amount': 0.15,
            'line_items': 0.15
        }
        
        scores = []
        for field, weight in weights.items():
            value = po_data.get(field, {})
            if isinstance(value, dict):
                conf = value.get('confidence', 0)
            elif isinstance(value, list):
                conf = sum([item.get('confidence', 0) for item in value]) / len(value) if value else 0
            else:
                conf = 0
            scores.append(conf * weight)
        
        return round(sum(scores), 1)
```

---

### Feature 5: Output Formatting (cXML + JSON)

**Requirement:**
Convert parsed PO to standard cXML and JSON formats compatible with Punchout.

**cXML PurchaseOrderRequest Structure:**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE cXML SYSTEM "http://xml.cxml.org/schemas/cXML/1.2.038/cXML.dtd">
<cXML payloadID="1104201113430571@buyer.example.com" timestamp="2025-01-04T13:14:30-08:00">
  <Header>
    <From>
      <Credential domain="NetworkId">
        <Identity>buyer@company.com</Identity>
      </Credential>
    </From>
    <To>
      <Credential domain="NetworkId">
        <Identity>supplier@company.com</Identity>
      </Credential>
    </To>
    <Sender>
      <Credential domain="NetworkId">
        <Identity>getjohn@getjohn.co.uk</Identity>
        <SharedSecret>shared-secret</SharedSecret>
      </Credential>
      <UserAgent>SmartPOParser/1.0</UserAgent>
    </Sender>
  </Header>
  <Request deploymentMode="production">
    <PurchaseOrderRequest>
      <PurchaseOrderHeader orderDate="2025-01-04" orderID="PO-12345">
        <Total>
          <Money currency="GBP">1250.50</Money>
        </Total>
        <ShipTo>
          <Address>
            <Name xml:lang="en">ACME Corp</Name>
            <PostalAddress>
              <Street>123 Main St</Street>
              <City>London</City>
              <PostalCode>SW1A 1AA</PostalCode>
              <Country isoCountryCode="GB">United Kingdom</Country>
            </PostalAddress>
          </Address>
        </ShipTo>
        <BillTo>
          <Address>
            <Name xml:lang="en">ACME Corp Accounts</Name>
          </Address>
        </BillTo>
      </PurchaseOrderHeader>
      <PurchaseOrderItemDetail>
        <LineNumber>1</LineNumber>
        <Description xml:lang="en">Promotional Merchandise - T-Shirts</Description>
        <UnitPrice>
          <Money currency="GBP">15.00</Money>
        </UnitPrice>
        <OrderQuantity unitCode="EA">50</OrderQuantity>
        <LineTotal>
          <Money currency="GBP">750.00</Money>
        </LineTotal>
      </PurchaseOrderItemDetail>
      <PurchaseOrderItemDetail>
        <LineNumber>2</LineNumber>
        <Description xml:lang="en">Mugs with company branding</Description>
        <UnitPrice>
          <Money currency="GBP">10.50</Money>
        </UnitPrice>
        <OrderQuantity unitCode="EA">50</OrderQuantity>
        <LineTotal>
          <Money currency="GBP">525.00</Money>
        </LineTotal>
      </PurchaseOrderItemDetail>
    </PurchaseOrderRequest>
  </Request>
</cXML>
```

**JSON Output (Internal Storage):**

```json
{
  "parsing_job_id": "uuid-123",
  "source_file_name": "PO_2025_ACME.pdf",
  "source_file_size_bytes": 245000,
  "ingestion_timestamp": "2025-01-04T13:14:30Z",
  "processing_duration_ms": 8500,
  "extracted_data": {
    "po_number": {
      "value": "PO-12345",
      "confidence": 98.5,
      "raw": "PO #12345"
    },
    "po_date": {
      "value": "2025-01-04",
      "confidence": 97.0,
      "raw": "January 4, 2025"
    },
    "buyer": {
      "name": {
        "value": "ACME Corporation",
        "confidence": 95.0
      },
      "address": {
        "street": "123 Main Street",
        "city": "London",
        "postal_code": "SW1A 1AA",
        "country_code": "GB",
        "confidence": 92.0
      }
    },
    "supplier": {
      "name": {
        "value": "XYZ Supplier Ltd",
        "confidence": 94.0
      }
    },
    "line_items": [
      {
        "line_number": 1,
        "sku": "TSHIRT-BLK-L",
        "description": "T-Shirts (Black, Large)",
        "quantity": 50,
        "unit_price": 15.00,
        "line_total": 750.00,
        "confidence": 96.5
      }
    ],
    "total_amount": {
      "value": 1250.50,
      "currency": "GBP",
      "confidence": 95.0
    }
  },
  "validation_result": {
    "is_valid": true,
    "errors": [],
    "warnings": [],
    "overall_confidence": 95.2
  },
  "processing_status": "auto_accepted",
  "ready_for_punchout": true,
  "cxml_output": "<?xml version=\"1.0\"...truncated...",
  "audit_trail": [
    {
      "action": "uploaded",
      "timestamp": "2025-01-04T13:14:30Z",
      "actor": "buyer@acme.com"
    },
    {
      "action": "preprocessing_completed",
      "timestamp": "2025-01-04T13:14:33Z",
      "deskew_angle": 0.2
    },
    {
      "action": "ocr_completed",
      "timestamp": "2025-01-04T13:14:35Z",
      "avg_confidence": 95.8
    },
    {
      "action": "parsing_completed",
      "timestamp": "2025-01-04T13:14:38Z"
    },
    {
      "action": "validation_passed",
      "timestamp": "2025-01-04T13:14:38Z"
    },
    {
      "action": "auto_accepted",
      "timestamp": "2025-01-04T13:14:39Z"
    }
  ]
}
```

---

### Feature 6: Confidence-Based Routing (Auto-Accept vs. Manual Review)

**Requirement:**
Route parsing results to auto-acceptance or manual review queue based on overall confidence score.

**Decision Logic:**

```python
class PORoutingEngine:
    """Route parsed PO to auto-accept or manual review."""
    
    AUTO_ACCEPT_THRESHOLD = 0.90  # 90% confidence
    MANUAL_REVIEW_THRESHOLD = 0.75  # 75% = escalate to support
    REJECT_THRESHOLD = 0.50  # Below 50% = too risky
    
    def route(self, po_data: dict, validation_result: dict) -> Dict[str, any]:
        overall_conf = validation_result['overall_confidence']
        is_valid = validation_result['is_valid']
        
        if not is_valid:
            return {
                'route': 'manual_review',
                'reason': 'Validation errors detected',
                'errors': validation_result['errors'],
                'severity': 'high',
                'escalate_to': 'support_team'
            }
        
        if overall_conf >= self.AUTO_ACCEPT_THRESHOLD:
            return {
                'route': 'auto_accept',
                'reason': f'High confidence ({overall_conf}%)',
                'action': 'emit_to_punchout',
                'notify': 'webhook_to_buyer',
                'severity': 'none'
            }
        
        elif overall_conf >= self.MANUAL_REVIEW_THRESHOLD:
            return {
                'route': 'manual_review',
                'reason': f'Medium confidence ({overall_conf}%)',
                'action': 'queue_for_review',
                'notify': 'slack_alert',
                'escalate_to': 'support_team',
                'severity': 'medium'
            }
        
        else:
            return {
                'route': 'rejected',
                'reason': f'Low confidence ({overall_conf}%). Manual resubmission recommended.',
                'action': 'reject',
                'notify': 'slack_alert + email_to_buyer',
                'escalate_to': 'support_team',
                'severity': 'high',
                'retry_advice': 'Resubmit with higher quality scan (300+ DPI, even lighting)'
            }
```

**Manual Review Queue:**
- Stored in `po_parsing_manual_review` table
- Simple web UI for Get John staff to:
  - View OCR extraction + highlighted low-confidence fields
  - Correct misread values
  - Approve or reject
  - Flag patterns for retraining

---

### Feature 7: Database Schema

**Requirement:**
Store parsing results, audit trail, and manual review queue.

```sql
-- Parsed PO records (immutable)
CREATE TABLE po_parsing_results (
    id UUID PRIMARY KEY,
    job_id UUID UNIQUE NOT NULL,
    original_filename VARCHAR(255) NOT NULL,
    original_file_blob_url TEXT NOT NULL,
    ingestion_timestamp TIMESTAMP DEFAULT NOW(),
    
    -- Extracted fields (JSONB for flexibility)
    extracted_data JSONB NOT NULL,
    validation_result JSONB NOT NULL,
    overall_confidence NUMERIC(3, 1) NOT NULL,
    
    -- Processing metadata
    processing_duration_ms INT,
    ocr_engine_version VARCHAR(50),
    preprocessing_steps TEXT[],  -- Array of applied preprocessing steps
    
    -- Status tracking
    processing_status VARCHAR(50),  -- 'auto_accepted', 'manual_review', 'rejected'
    ready_for_punchout BOOLEAN DEFAULT FALSE,
    punchout_emission_timestamp TIMESTAMP,
    
    -- Generated outputs
    cxml_output TEXT,
    json_output JSONB,
    
    -- Audit
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    
    CONSTRAINT fk_po_validation FOREIGN KEY (id) REFERENCES po_validation_audits(po_id)
);

-- Manual review queue
CREATE TABLE po_parsing_manual_review (
    id UUID PRIMARY KEY,
    po_parsing_result_id UUID NOT NULL REFERENCES po_parsing_results(id),
    
    reason VARCHAR(255),  -- 'low_confidence', 'validation_error', 'ambiguous_field'
    confidence_threshold NUMERIC(3, 1),
    
    -- Reviewer assignment
    assigned_to_user_id UUID,
    assigned_timestamp TIMESTAMP,
    
    -- Review data
    reviewer_corrections JSONB,  -- User's corrected values
    review_status VARCHAR(50),  -- 'pending', 'approved', 'rejected', 'needs_help'
    review_completed_at TIMESTAMP,
    
    -- Feedback loop
    review_notes TEXT,
    feedback_confidence_delta NUMERIC(3, 1),  -- Confidence change after review
    
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Audit trail (event log)
CREATE TABLE po_parsing_audit_trail (
    id UUID PRIMARY KEY,
    po_parsing_result_id UUID NOT NULL REFERENCES po_parsing_results(id),
    
    action VARCHAR(100),  -- 'uploaded', 'preprocessing_completed', 'ocr_completed', etc.
    action_timestamp TIMESTAMP,
    actor VARCHAR(255),  -- User email or system name
    
    -- Context
    metadata JSONB,  -- Action-specific data (e.g., deskew_angle, avg_confidence)
    error_message TEXT,  -- If action failed
    
    created_at TIMESTAMP DEFAULT NOW(),
    INDEX idx_po_id (po_parsing_result_id),
    INDEX idx_timestamp (action_timestamp)
);

-- Statistics & analytics
CREATE TABLE po_parsing_metrics (
    metric_date DATE NOT NULL,
    
    -- Volume
    total_pos_submitted INT,
    auto_accepted INT,
    manual_review INT,
    rejected INT,
    
    -- Accuracy
    avg_overall_confidence NUMERIC(3, 1),
    field_confidence_breakdown JSONB,  -- {"po_number": 96.5, "date": 94.0, ...}
    
    -- Performance
    avg_processing_time_ms INT,
    p95_processing_time_ms INT,
    p99_processing_time_ms INT,
    
    -- Review metrics
    manual_review_approval_rate NUMERIC(3, 1),
    avg_review_time_minutes INT,
    
    created_at TIMESTAMP DEFAULT NOW(),
    PRIMARY KEY (metric_date)
);
```

---

## Integration Points

### 1. Get John Existing System Integration

**Endpoint:** POST `/api/po/parse`

**Request:**
```http
POST https://api.getjohn.co.uk/api/po/parse HTTP/1.1
Content-Type: multipart/form-data
Authorization: Bearer {JWT_TOKEN}

[Binary PO PDF or image]
```

**Response:**
```json
{
  "job_id": "uuid-456",
  "status": "processing",
  "status_check_url": "https://api.getjohn.co.uk/api/po/parse/uuid-456",
  "estimated_completion_ms": 8000
}
```

**Status Check:** GET `/api/po/parse/{job_id}`
```json
{
  "job_id": "uuid-456",
  "status": "completed",
  "result": {
    "po_number": "PO-12345",
    "po_date": "2025-01-04",
    "processing_status": "auto_accepted",
    "ready_for_punchout": true,
    "cxml_output": "<?xml version=\"1.0\"...>",
    "overall_confidence": 95.2
  },
  "webhook_sent": true,
  "webhook_response_status": 200
}
```

### 2. Punchout Integration

**Output Trigger:** When PO is auto-accepted:
- Generate cXML PurchaseOrderRequest
- Submit to buyer's Punchout endpoint (via Get John's existing cXML sender)
- Track delivery in audit trail

### 3. Email Forwarding

**Setup:** 
- Create Get John email address: `po-parser@getjohn.co.uk`
- Configure Azure Logic Apps to:
  1. Trigger on incoming email to address
  2. Extract PDF attachment
  3. Call `/api/po/parse` with PDF
  4. Reply to sender with job_id and status URL

### 4. Webhook Notification (Buyer-Facing)

**POST to buyer's webhook URL** (provided during registration):
```json
{
  "event": "po_parsing_completed",
  "job_id": "uuid-456",
  "po_number": "PO-12345",
  "status": "auto_accepted",
  "cxml_output": "<?xml version=\"1.0\"...",
  "overall_confidence": 95.2,
  "timestamp": "2025-01-04T13:14:39Z",
  "hmac_signature": "sha256=abc123def456"
}
```

**Buyer Integration:** Buyer's system can:
- Auto-import if status = auto_accepted
- Manual review if status = manual_review
- Alert user if status = rejected

### 5. Slack Integration (Get John Operations)

**Manual Review Alert:**
```
Channel: #po-parsing
---
⚠️ New manual review item

PO Job: uuid-456
Filename: acme_po_2025.pdf
Confidence: 78% (below threshold)
Issue: "Line item quantity unreadable"

Review URL: https://getjohn.co.uk/admin/po-review/uuid-456
```

**Daily Analytics:**
```
Daily PO Parsing Report (2025-01-04)

📊 Volume
  • Submitted: 87 POs
  • Auto-accepted: 72 (83%)
  • Manual review: 12 (14%)
  • Rejected: 3 (3%)

📈 Quality
  • Avg confidence: 94.2%
  • Avg processing time: 7.8 seconds
  • Manual review approval rate: 96%
```

---

## Performance & Scalability

### Throughput Requirements

| Metric | Requirement | Rationale |
|--------|------------|-----------|
| **Peak Throughput** | 100 POs/minute | Support simultaneous uploads from 80+ customers |
| **Single PO Latency** | <10 seconds end-to-end | User expectation for responsive UI |
| **99th Percentile Latency** | <30 seconds | Account for worst-case complex POs |
| **Daily Volume** | 1000+ POs/day | Growing customer base |

### Azure Functions Scaling Strategy

**HTTP Trigger (parse_po_handler):**
- **Plan:** Consumption (auto-scale)
- **Expected per-instance:** 100-200 requests/second
- **Scale trigger:** Queue length > 10 messages
- **Max instances:** 200 (configurable)

**Queue Trigger (po_parsing_processor):**
- **Plan:** Premium (for better concurrency)
- **Worker processes:** 4 per instance (CPU-bound: Tesseract OCR)
- **Language worker threads:** Number of cores (8 typically)
- **Max instances:** 50

**Cost Estimation:**
- **HTTP function:** ~$0.0000002 per execution
- **Queue processing:** ~$0.001 per 1M queue operations
- **Blob storage:** ~$0.0184 per GB/month
- **Database:** PostgreSQL flexible server (B1S tier): ~$30/month for dev, $100+/month for production

**Example:** 1000 POs/day
- HTTP executions: 1000 = $0.0002
- Queue operations: 1000 = $0.000001
- Storage: 1GB/month (assuming 1MB avg) = $0.0184
- **Total variable cost: <$0.05/day for compute**

---

## Monitoring & Observability

### Key Metrics to Track

```python
# Application Insights custom metrics

# Throughput
application_insights.track_metric("po_parsing_submitted", 1)  # per submission
application_insights.track_metric("po_parsing_auto_accepted", 1)
application_insights.track_metric("po_parsing_manual_review", 1)
application_insights.track_metric("po_parsing_rejected", 1)

# Latency
application_insights.track_metric("po_parsing_duration_ms", duration)
application_insights.track_metric("ocr_duration_ms", ocr_time)
application_insights.track_metric("field_extraction_duration_ms", extraction_time)

# Quality
application_insights.track_metric("overall_confidence_score", confidence)
application_insights.track_metric("po_number_confidence", po_num_conf)
application_insights.track_metric("line_item_accuracy", items_conf)

# Errors
application_insights.track_exception(exception)
application_insights.track_trace(f"OCR failed: {error}", severity="error")
```

### Alert Thresholds

| Alert | Condition | Severity |
|-------|-----------|----------|
| **High Error Rate** | >5% processing failures | Critical |
| **Latency P95** | >30 seconds | Warning |
| **Low Avg Confidence** | <80% daily average | Warning |
| **Queue Backlog** | >500 messages in queue | Critical |
| **Manual Review Backlog** | >50 pending reviews | Warning |

### Dashboards

**Operations Dashboard:**
- Daily PO volume (chart)
- Auto-accept rate (gauge)
- Avg confidence score (gauge)
- Processing latency distribution (histogram)
- Manual review queue length (gauge)
- Error rate (chart)

**Data Quality Dashboard:**
- Per-field confidence scores (bar chart)
- Validation error frequency (pie chart)
- Confidence distribution (histogram)
- Top failure patterns (table)

---

## Testing & QA Strategy

### Unit Tests

```python
# tests/test_field_parser.py

import pytest
from po_parser.field_parser import POFieldParser

class TestPOFieldParser:
    
    def test_extract_po_number_standard(self):
        """Test extraction of standard PO number format."""
        text = "PO #12345 dated January 4, 2025"
        ocr_data = {
            'text': ['PO', '#12345', 'dated', ...],
            'conf': [95, 98, 90, ...]
        }
        parser = POFieldParser(text, ocr_data)
        result = parser.extract_po_number()
        assert result['value'] == '12345'
        assert result['confidence'] >= 90
    
    def test_extract_po_number_with_prefix(self):
        """Test PO number with 'ORDER-' prefix."""
        text = "ORDER-67890 placed by ACME Corp"
        parser = POFieldParser(text, ocr_data)
        result = parser.extract_po_number()
        assert result['value'] == '67890'
    
    def test_extract_po_date_us_format(self):
        """Test US date format MM/DD/YYYY."""
        text = "Order Date: 01/04/2025"
        parser = POFieldParser(text, ocr_data)
        result = parser.extract_date()
        assert result['value'] == '2025-01-04'
    
    def test_extract_po_date_uk_format(self):
        """Test UK date format DD/MM/YYYY."""
        text = "Order Date: 04/01/2025"  # Should be interpreted as 4 Jan
        parser = POFieldParser(text, ocr_data)
        result = parser.extract_date()
        assert result['value'] == '2025-01-04'
    
    def test_extract_line_items(self):
        """Test extraction of line items from table."""
        text = """
        SKU              Description           Qty    Unit Price   Total
        TSHIRT-BLK-L    T-Shirts (Black)       50     15.00       750.00
        MUGS-LOGO       Mugs w/ Logo           50     10.50       525.00
        """
        parser = POFieldParser(text, ocr_data)
        items = parser.extract_line_items()
        assert len(items) == 2
        assert items[0]['sku'] == 'TSHIRT-BLK-L'
        assert items[0]['quantity'] == 50
        assert items[0]['unit_price'] == 15.00
```

### Integration Tests

```python
# tests/test_end_to_end.py

import pytest
from azure.storage.blob import BlobServiceClient
from po_parser.parser import parse_po_from_blob

class TestEndToEnd:
    
    @pytest.fixture
    def sample_po_pdf(self):
        """Load sample PO PDF."""
        return open('tests/fixtures/sample_po_acme_2025.pdf', 'rb').read()
    
    def test_parse_standard_pdf(self, sample_po_pdf):
        """Test end-to-end parsing of standard PO PDF."""
        result = parse_po_from_blob(sample_po_pdf)
        
        assert result['processing_status'] == 'auto_accepted'
        assert result['extracted_data']['po_number']['value'] == 'PO-12345'
        assert result['extracted_data']['overall_confidence'] >= 0.90
        assert result['validation_result']['is_valid'] == True
        assert len(result['extracted_data']['line_items']) == 2
    
    def test_parse_low_quality_scan(self):
        """Test handling of low-quality scanned PO."""
        low_quality_pdf = open('tests/fixtures/low_quality_po.pdf', 'rb').read()
        result = parse_po_from_blob(low_quality_pdf)
        
        # Should route to manual review, not auto-accept
        assert result['processing_status'] == 'manual_review'
        assert result['extracted_data']['overall_confidence'] < 0.90
    
    def test_cxml_output_generation(self, sample_po_pdf):
        """Test cXML output matches cXML schema."""
        result = parse_po_from_blob(sample_po_pdf)
        cxml = result['cxml_output']
        
        # Validate cXML structure
        assert '<?xml version="1.0"' in cxml
        assert '<PurchaseOrderRequest>' in cxml
        assert '<PurchaseOrderHeader' in cxml
        assert result['extracted_data']['po_number']['value'] in cxml
```

### Regression Tests (Golden Dataset)

**Maintain a "golden dataset" of 50-100 representative POs:**
- Various formats (invoice-style, table-style, handwritten amounts)
- Different industries (retail, healthcare, tech)
- Multiple languages (Phase 2: Chinese, Spanish)
- Edge cases (missing fields, illegible sections)

**Run regression tests on every deployment:**
```python
def test_regression_suite():
    """Test parser against golden dataset."""
    golden_dataset = load_golden_dataset()
    
    results = {
        'passed': 0,
        'failed': 0,
        'degraded': 0,  # Confidence decreased
    }
    
    for po_fixture in golden_dataset:
        result = parse_po_from_blob(po_fixture['pdf'])
        expected = po_fixture['expected_output']
        
        # Check key fields match
        assert result['extracted_data']['po_number']['value'] == expected['po_number']
        assert result['extracted_data']['total_amount']['value'] == expected['total']
        
        # Check confidence hasn't degraded
        if result['overall_confidence'] < expected['min_confidence']:
            results['degraded'] += 1
        else:
            results['passed'] += 1
    
    assert results['failed'] == 0, f"Regression test failed: {results}"
    assert results['degraded'] < 3, f"Confidence degradation in {results['degraded']} cases"
```

---

## Data Privacy & Security

### Security Requirements

1. **Input Validation:**
   - File size limit: 10 MB
   - MIME type whitelist: application/pdf, image/jpeg, image/png
   - Virus scanning: ClamAV or Microsoft Defender for Cloud (optional)

2. **Data Encryption:**
   - In Transit: HTTPS/TLS 1.2+
   - At Rest: Azure Blob Storage encryption (AES-256 default)
   - Database: PostgreSQL SSL connections

3. **Access Control:**
   - JWT authentication for API endpoints
   - Role-based access: `po_viewer`, `po_reviewer`, `po_admin`
   - Audit logging: All data access tracked with user/timestamp

4. **PII Handling:**
   - PO data (buyer/supplier names, addresses) considered PII
   - GDPR compliance: Support data retention policies, right to deletion
   - Implement: Auto-delete processed POs after 30 days (configurable)

### GDPR Compliance

```python
class DataRetentionPolicy:
    """Implement GDPR data retention and deletion."""
    
    RETENTION_DAYS = 30  # Configurable per customer
    
    def cleanup_old_records(self):
        """Delete PO parsing records after retention period."""
        cutoff_date = datetime.now() - timedelta(days=self.RETENTION_DAYS)
        
        # Delete from manual review queue
        deleted_manual = db.delete(
            'po_parsing_manual_review',
            where=f'created_at < {cutoff_date}'
        )
        
        # Delete from audit trail
        deleted_audit = db.delete(
            'po_parsing_audit_trail',
            where=f'created_at < {cutoff_date}'
        )
        
        # Delete from main results (keep only aggregated metrics)
        deleted_results = db.delete(
            'po_parsing_results',
            where=f'created_at < {cutoff_date}'
        )
        
        logger.info(f"Deleted {deleted_manual + deleted_audit + deleted_results} records")
    
    def anonymize_record(self, po_id: str):
        """GDPR: Right to be forgotten."""
        # Replace PII with hashes
        db.update('po_parsing_results', po_id, {
            'extracted_data': anonymize_pii(extracted_data)
        })
```

---

## Rollout & Deployment Plan

### Phase 1: Internal Beta (Jan-Feb 2026)
- Deploy to **Azure Functions Dev** environment
- Test with 10-20 representative Get John customer POs
- Gather feedback on UX, accuracy, latency
- Refine field extraction rules based on real data

### Phase 2: Closed Beta (Mar 2026)
- Deploy to **Staging** with production-like data volume
- Invite 3-5 friendly customers to test
- Monitor for edge cases, error rates
- Load testing (100 concurrent uploads)

### Phase 3: General Availability (Apr 2026)
- **Production deployment**
- Announce to all customers via email + blog
- Offer as optional feature (disabled by default)
- Monitor closely for first 2 weeks

### Deployment Checklist
- [ ] Database migrations tested
- [ ] Rollback plan documented (blue-green deployment)
- [ ] Monitoring dashboards live
- [ ] Alert thresholds configured
- [ ] Support playbook written
- [ ] Customer documentation complete
- [ ] API versioning strategy in place

---

## Success Criteria (MVP Launch)

### Technical Success
- ✅ Auto-accept rate: >70% of submissions (confidence >90%)
- ✅ Processing latency: P95 <10 seconds
- ✅ OCR accuracy: 95%+ on standard formats
- ✅ System uptime: 99.5%

### Business Success
- ✅ Customer adoption: >30% of eligible customers enabled within 6 months
- ✅ Support ticket reduction: 20% fewer "PO submission" related tickets
- ✅ NPS impact: +5 point improvement in customer satisfaction
- ✅ Cost per transaction: <$0.01

### User Experience Success
- ✅ Time from submission to system entry: <30 seconds (vs. 5 minutes manual)
- ✅ False positive rate: <5% (user frustration from errors)
- ✅ User-initiated corrections: <2% of submissions (field misparsing)

---

## Risks & Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|-----------|-----------|
| **OCR Accuracy <90%** | High false positives, support burden | Medium | Invest in preprocessing, custom Tesseract training (Phase 2) |
| **Tesseract Latency** | Can't meet <10s target at scale | Low | Profile with production load, consider parallel workers |
| **Handwritten POs** | Can't parse, manual review queue | Medium | Document limitation upfront, guide users to print/scan clearly |
| **Layout Variability** | Different vendor PO formats fail | High | Build heuristic-based parser (not rule-based), invest in training data |
| **Security Breach** | Customer PII exposure | Low | Implement encryption, RBAC, audit logging, regular pen testing |
| **Cost Overruns** | Azure consumption exceeds budget | Low | Implement throttling, rate limits, cost monitoring alerts |

---

## Future Enhancements (Phase 2)

- 🚀 **Custom Tesseract Training:** Fine-tune OCR on Get John customer POs (improves accuracy 5-10%)
- 🌍 **Multi-Language Support:** Chinese, Spanish, French POs
- 🤖 **LLM Integration:** Use Claude/GPT for complex, unstructured POs (for Phase 2 consideration)
- 📱 **Manual Review Web UI:** Rich editor for staff to correct extractions
- 📊 **Analytics Dashboard:** Customer-facing metrics (submission history, accuracy trends)
- 🔗 **ERP Integration:** Direct sync to Xero, QuickBooks, NetSuite
- ✉️ **Smart Email Reply:** Auto-respond to PO email submissions with status updates
- 📧 **Invoice Matching:** Match POs to receipts/invoices for full procure-to-pay automation

---

## Appendix A: cXML Schema Reference

**cXML Purchase Order Key Elements:**

```xml
<!-- Header section -->
<PurchaseOrderRequest>
  <PurchaseOrderHeader 
    orderDate="YYYY-MM-DD"
    orderID="string"
    requisitionID="string" (optional)
    orderType="new|change|delete" (default: new)
  >
    <!-- Buyer info -->
    <ShipTo>
      <Address>
        <Name xml:lang="en">Company Name</Name>
        <PostalAddress>
          <Street>Street Address</Street>
          <City>City Name</City>
          <State>State/Province</State>
          <PostalCode>Postal Code</PostalCode>
          <Country isoCountryCode="GB">Country Name</Country>
        </PostalAddress>
      </Address>
    </ShipTo>
    
    <!-- Supplier info -->
    <BillTo>
      <Address>
        <Name xml:lang="en">Supplier Company</Name>
      </Address>
    </BillTo>
    
    <!-- Order total -->
    <Total>
      <Money currency="GBP">Amount</Money>
    </Total>
    
    <!-- Delivery terms -->
    <ShippingDate>YYYY-MM-DD</ShippingDate>
  </PurchaseOrderHeader>
  
  <!-- Line items -->
  <PurchaseOrderItemDetail>
    <LineNumber>1</LineNumber>
    <Description xml:lang="en">Item Description</Description>
    <UnitPrice>
      <Money currency="GBP">Price</Money>
    </UnitPrice>
    <OrderQuantity unitCode="EA|CS|BOX|...">Quantity</OrderQuantity>
    <LineTotal>
      <Money currency="GBP">LineAmount</Money>
    </LineTotal>
  </PurchaseOrderItemDetail>
  
</PurchaseOrderRequest>
```

---

## Appendix B: Tesseract Configuration Best Practices

```python
# Recommended Tesseract settings for B2B PO documents

TESSERACT_CONFIG = {
    'oem': 1,           # LSTM engine (more accurate than legacy)
    'psm': 1,           # Full page auto PSM (best for mixed layouts)
    'lang': 'eng',      # English (add 'equ' for equations if needed)
    
    # Character set (restrict to expected PO characters)
    'tessedit_char_whitelist': '0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz,./-$£€:;()@#&',
    
    # Numeric confidence
    'preserve_interword_spaces': '1',
    'classify_bln_numeric_mode': '1',
}

# Image preprocessing order (critical for OCR accuracy):
# 1. Convert to grayscale
# 2. Deskew/rotate
# 3. Denoise (median blur)
# 4. Threshold (binary)
# 5. Upscale if needed (300+ DPI)
# 6. Run Tesseract
```

---

**Document Version:** 1.0  
**Last Updated:** January 4, 2026  
**Next Review:** April 2026 (Post-MVP Retrospective)
