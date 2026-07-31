# Aimsio Document Ingestion & Intelligent Pricebook Engine
## Automated Multimodal Ingestion, Vision AI Logo Parsing, In-Memory DuckDB Pricebook Engine, and VPC Integration

### Tech Stack & Tools
* **Orchestration & Compute:** AWS Step Functions, AWS Lambda (Python 3.13 & JavaScript)
* **Embedded Database Engine:** DuckDB (In-Memory SQL)
* **AI Vision & Data Parsing:** OpenAI API (GPT-4o Vision), PyMuPDF (`fitz`), PDFPlumber, Regular Expressions (`re`)
* **Cloud Infrastructure & Networking:** Amazon S3, AWS VPC (Static Whitelisted IPs), AWS Boto3
* **Low-Code Migration:** Migrated from [Make (formerly Integromat)](https://www.make.com)

---

## Moving from Low-Code Limits to Serverless Step Functions

At EMI, we automate field service ticket and invoice submissions for oil and gas suppliers to platforms including OpenInvoice and OpenTicket. Many client suppliers manage field operations using an accounting software called Aimsio. When Aimsio creates a ticket or invoice, it generates a raw JSON file alongside an attached PDF copy.

When these files arrive in our ingestion system, they route through a mapping pipeline to prepare data for downstream platforms. Originally, this processing pipeline was built inside [Make (formerly Integromat)](https://www.make.com), a visual low-code automation platform. 

As clients requested more complex features, keeping custom business logic inside Make scenarios created severe engineering problems:

* **Problem 1: Missing ticket details in blanket JSON exports.** When Aimsio exports blanket invoices containing multiple tickets, the raw JSON payload omits individual ticket header details. The information exists solely as text inside the attached PDF.
  
* **Problem 2: Omitted supplier department fields.** OpenInvoice requires invoices to specify a supplier site or department code. The Aimsio raw JSON export lacks this field entirely, but the PDF document visually displays the department logo on the header.
  
* **Problem 3: Inflexible low-code architecture.** Adding multi-page PDF text parsing, image conversion, and dynamic pricebook matching made the Make flows brittle and difficult to debug. Realizing that low-code workflows could not scale with these expanding requirements, I took the initiative to consolidate the entire pipeline natively inside AWS Step Functions.

---

## Solution & Results

To solve these challenges, I built an AWS Step Functions state machine along with a suite of custom Python Lambdas. The pipeline automates PDF text extraction, visual AI logo recognition, in-memory SQL pricebook matching, and secure VPC code validation.

### Key Results
* **Native Serverless Consistency:** Replaced fragile Make webhooks with a native AWS Step Functions state machine featuring clear visual execution logs and instant CloudWatch debugging.

* **Automated Visual Logo Extraction:** Integrated PyMuPDF image conversion and GPT-4o vision calls to extract division logos and populate missing department codes automatically.

* **Sub-Second DuckDB Matching:** Leveraged an in-memory DuckDB database inside AWS Lambda to run weighted SQL matches against thousands of CSV pricebook records in milliseconds.

* **Complete Ticket Data Recovery:** Extracted missing ticket headers directly from PDFs using PDFPlumber, achieving 100 percent data coverage for blanket invoices.

---

## Architecture & Data Flow

I built the AWS Step Functions state machine and core processing Lambdas, orchestrating data movement across S3, OpenAI, DuckDB, and platform APIs.

<img width="1606" height="719" alt="fractionstepfunctions_graph" src="https://github.com/user-attachments/assets/245cd97c-7ca4-4fa6-b266-d1a063c00801" />


### Data Processing Steps

1. **Format Routing (`Check isOTFormat`):** Evaluates the incoming `isOTFormat` flag. OpenTicket submissions skip ticket PDF parsing and jump straight to logo extraction, while OpenInvoice submissions pass through PDF header parsing first.

2. **Aimsio PDF Header Parsing (`extract Aimsio tickets header from PDF`):** Reads the PDF from S3 using PDFPlumber to parse individual ticket headers from multi-ticket invoices, merging the extracted data back into the central JSON state.

3. **Multimodal Vision Processing (`chatGpt reads logos`):** Converts PDF pages to JPEG images using PyMuPDF and passes base64 streams to OpenAI GPT-4o to identify division names from logos, enriching the department field.

4. **Pipeline Branching (`Check isOT`):** Routes OpenTicket submissions to the pricebook engine and OpenInvoice submissions to VPC validation.

5. **In-Memory DuckDB Pricebook Engine (`Fraction OT pricebook`):** Runs for OpenTicket submissions. Loads buyer CSV pricebooks into an in-memory DuckDB table, cleans descriptions, validates rate bounds and expiry dates, and updates item IDs in the JSON.

6. **VPC AFE & Code Validation (`Fraction OI AFE validation`):** Executes inside an existing whitelisted AWS VPC to validate AFE numbers and accounting codes against OpenInvoice APIs before final submission.

7. **Final Output (`output to v3 lambda`):** Passes the fully enriched and validated JSON payload to downstream submission Lambdas.

---

## Engineering Challenges & Solutions

### 1. Sub-second CSV pricebook queries using in-memory DuckDB
* **The Challenge:** OpenTicket line items use internal Aimsio descriptions that do not match registered buyer pricebooks. Line items had to be validated against CSV pricebooks containing effective date ranges, rate limits, and unit types. Performing nested Python loops across large pricebooks inside Lambda was slow and inefficient.

* **The Solution:** Embedded DuckDB directly inside the `fraction_pricebook_mapping_complete` Lambda handler. On invocation, the Lambda loads the buyer CSV into an in-memory DuckDB table (`:memory:`), normalizes text descriptions with SQL regex functions, and filters out expired records. It then executes a weighted SQL query that ranks candidates by match weight, string length difference, and rate constraints to return the single best pricebook match in milliseconds.

### 2. Extracting visual department context using PyMuPDF and GPT-4o
* **The Challenge:** OpenInvoice requires a supplier department code, but Aimsio JSON files omit this field. The information is only visible as a graphical company logo on the PDF header.

* **The Solution:** Developed `fraction_logo_extractor` using PyMuPDF (`fitz`) and OpenAI vision models. The Lambda converts PDF pages into base64 JPEG images and submits them to GPT-4o. The vision model inspects the header, extracts division names and parent company text, and writes the structured department data back to the central JSON file in S3.

### 3. Recovering missing ticket headers from blanket invoice PDFs
* **The Challenge:** Aimsio blanket invoices group multiple tickets into one PDF. The exported JSON payload omits individual ticket headers, leaving downstream systems without ticket level dates or PO numbers.

* **The Solution:** Wrote `aimsio_extract_tickets_header_pdf` using PDFPlumber to inspect PDF text streams. The Lambda scans dedicated text labels across multi-page files, extracts missing ticket headers, and merges them into the S3 state JSON before downstream submission.

### 4. Integrating whitelisted API calls using an existing AWS VPC
* **The Challenge:** OpenInvoice API calls for AFE and accounting code validation fail if they do not originate from authorized static IP addresses.

* **The Solution:** Placed the AFE validation Lambda directly inside our existing AWS VPC configured with static Elastic IPs and NAT Gateways. This ensures all outgoing requests match vendor whitelists without requiring complex network redesigns.
