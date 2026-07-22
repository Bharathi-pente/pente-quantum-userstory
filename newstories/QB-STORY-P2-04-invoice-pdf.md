# QuantumBilling User Story: Invoice PDF Generation

> Aligned with ADR-001 (2026-07-01).

---

## Story ID & Metadata

| Field | Value |
|-------|-------|
| **Story ID** | QB-STORY-P2-04 |
| **Sprint** | Sprint 12-13 |
| **Phase** | Billing Core |
| **Domain** | Invoices — Document Generation |
| **Priority** | P2 — Medium |

---

## Title

**Invoice PDF Generation** — auto-generated, downloadable invoice PDFs with proper formatting, tax breakdown, and company branding

---

## Badges

| Backend | UI | Auth / RBAC | Billing Engine | Priority |
|---------|----|-------------|----------------|----------|
| Backend | UI | Auth / RBAC | Billing Engine | P2 — Medium |

---

## Description

**As a customer**, I want to **download a PDF version of my invoice** with proper formatting, tax breakdown, and company logo so that I can **submit it for accounting and compliance purposes.**

### Current State

OpenAPI spec defines `GET /invoices/{invoiceId}/pdf` endpoint. But **no implementation code exists** anywhere in the codebase — no PDF generation service, no storage, no download endpoint.

---

## Acceptance Criteria

| # | Criterion |
|---|-----------|
| AC-1 | Every finalized invoice has an auto-generated PDF stored in S3-compatible object storage |
| AC-2 | PDF includes: invoice number, dates, seller info, buyer info, line items with quantities and prices, adjustments, subtotal, tax breakdown, total, payment terms, QR code |
| AC-3 | Customer portal has "Download PDF" button on every invoice |
| AC-4 | API endpoint `GET /api/v1/invoices/:id/pdf` returns the PDF binary |
| AC-5 | PDF template is customizable via configuration (logo, colors, footer text) |
| AC-6 | PDF generation is async — invoice finalization triggers a background job, invoice shows "PDF generating..." until complete |

---

## Code Changes Required

### 1. PDF Generation Service (NestJS)

**New file:** `control-plane/src/billing/pdf-generator.service.ts`

```typescript
import { PDFDocument, StandardFonts, rgb } from 'pdf-lib';  // or puppeteer

@Injectable()
export class PdfGeneratorService {
    async generateInvoicePdf(invoice: InvoiceDto): Promise<Buffer> {
        const pdfDoc = await PDFDocument.create();
        const page = pdfDoc.addPage([612, 792]); // US Letter
        
        // Add company logo
        // Add invoice header (number, dates, customer info)
        // Add line items table
        // Add subtotal, tax, total
        // Add payment terms
        // Add QR code for payment
        
        return Buffer.from(await pdfDoc.save());
    }
}
```

### 2. Object Storage Integration

**New file:** `control-plane/src/billing/invoice-storage.service.ts`

```typescript
@Injectable()
export class InvoiceStorageService {
    constructor(@Inject('S3') private readonly s3: S3Client) {}
    
    async uploadPdf(invoiceId: string, pdf: Buffer): Promise<string> {
        const key = `invoices/${invoiceId}.pdf`;
        await this.s3.send(new PutObjectCommand({
            Bucket: process.env.INVOICE_STORAGE_BUCKET,
            Key: key,
            Body: pdf,
            ContentType: 'application/pdf',
        }));
        return key;
    }
    
    async getPdf(invoiceId: string): Promise<Buffer> {
        const key = `invoices/${invoiceId}.pdf`;
        const response = await this.s3.send(new GetObjectCommand({
            Bucket: process.env.INVOICE_STORAGE_BUCKET,
            Key: key,
        }));
        return Buffer.from(await response.Body.transformToByteArray());
    }
}
```

### 3. Background Job Trigger

**File:** `control-plane/src/billing/invoices.service.ts`

```typescript
async onInvoiceFinalized(invoiceId: string) {
    // Trigger async PDF generation
    await this.queue.add('generate-invoice-pdf', { invoiceId });
}
```

### 4. Download Endpoint

```typescript
@Get(':invoiceId/pdf')
@Roles('ORG_ADMIN', 'SUPER_ADMIN', 'CUSTOMER')
async downloadPdf(@Param('invoiceId') id: string, @Res() res: Response) {
    const pdf = await this.pdfGeneratorService.getInvoicePdf(id);
    res.setHeader('Content-Type', 'application/pdf');
    res.setHeader('Content-Disposition', `attachment; filename="invoice-${id}.pdf"`);
    res.send(pdf);
}
```

---

## Estimate

**1-2 sprints** (PDF generation 1, storage + endpoint + UI 1)

**Dependencies:** QB-STORY-OPS-06 (object storage — S3 or compatible)
