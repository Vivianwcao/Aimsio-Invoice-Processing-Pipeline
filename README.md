# Aimsio Document Ingestion & Intelligent Pricebook Engine
## Serverless Document Processing with Vision AI, DuckDB, and AWS Step Functions

### Tech Stack & Tools
* **Orchestration & Compute:** AWS Step Functions, AWS Lambda (Python 3.13 & JavaScript)
* **Embedded Database Engine:** DuckDB (In-Memory SQL)
* **AI Vision & Data Parsing:** OpenAI API (GPT-4o Vision), PyMuPDF (`fitz`), PDFPlumber, Regular Expressions (`re`)
* **Cloud Infrastructure & Networking:** Amazon S3, AWS VPC (Static Whitelisted IPs), AWS Boto3
* **Low-Code Migration:** Migrated from [Make (formerly Integromat)](https://www.make.com)

---

## Moving from Low-Code Limits to Serverless Step Functions

At EMI, we automate field service ticket and invoice submissions for oil and gas suppliers to platforms including OpenInvoice and OpenTicket. Many client suppliers manage field operations using an accounting software called Aimsio. When Aimsio creates a ticket or invoice, it generates a raw JSON file alongside an attached PDF copy.

When these files arrive, they pass through a mapping pipeline before being submitted to downstream platforms. Originally, this processing pipeline was built inside [Make (formerly Integromat)](https://www.make.com), a visual low-code automation platform. 

As clients requested more complex features, keeping custom business logic inside Make scenarios created severe engineering problems:

* **Problem 1: Missing ticket details in blanket JSON exports.** When Aimsio exports blanket invoices containing multiple tickets, the raw JSON payload omits individual ticket header details. The information exists solely as text inside the attached PDF.
  
* **Problem 2: Omitted supplier department fields.** OpenInvoice requires invoices to specify a supplier site or department code. The Aimsio raw JSON export lacks this field entirely, but the PDF document visually displays the department logo on the header.
  
* **Problem 3: The Make workflow became difficult to maintain.** Adding multi-page PDF text parsing, image conversion, and dynamic pricebook matching made the Make flows brittle and difficult to debug. 

---

## Solution & Results

To solve these challenges, I rebuilt the pipeline in AWS Step Functions using Python Lambda functions. The pipeline automates PDF text extraction, visual AI logo recognition, in-memory SQL pricebook matching, and secure VPC code validation.

### Key Results
* **Rebuilt the Pipeline in AWS:** Replaced the Make workflow with an AWS Step Functions pipeline, providing centralized execution history and CloudWatch logging for easier debugging.

* **Parsed Ticket Headers from PDFs:** Used PDFPlumber to extract ticket header information that was present only in attached PDFs and merge it into the processing pipeline.

* **Identified Supplier Departments from Logos:** Used PyMuPDF and GPT-4o Vision to recognize supplier division logos and populate the department code required by OpenInvoice.

* **Millisecond Pricebook Matching:** Used an in memory DuckDB database inside Lambda to match invoice line items against hundreds of pricebook records in milliseconds.

---

## Architecture & Data Flow

I built the AWS Step Functions state machine and core processing Lambdas, coordinating data flow across S3, OpenAI, DuckDB, and platform APIs.

<img width="1606" height="719" alt="fractionstepfunctions_graph" src="https://github.com/user-attachments/assets/245cd97c-7ca4-4fa6-b266-d1a063c00801" />


### Data Processing Steps

1. **Format Routing (`Check isOTFormat`):** Evaluates the incoming `isOTFormat` flag. OpenTicket submissions skip ticket PDF parsing and jump straight to logo extraction, while OpenInvoice submissions pass through PDF header parsing first.

2. **Aimsio PDF Header Parsing (`extract Aimsio tickets header from PDF`):** Reads the PDF from S3 using PDFPlumber to parse individual ticket headers from multi-ticket invoices, merging the extracted data back into the central JSON state.

3. **Multimodal Vision Processing (`chatGpt reads logos`):** Converts PDF pages to JPEG images using PyMuPDF and passes base64 streams to OpenAI GPT-4o to identify division names from logos, enriching the department field.

4. **Pipeline Branching (`Check isOT`):** Routes OpenTicket submissions to the pricebook engine and OpenInvoice submissions to VPC validation.

5. **In-Memory DuckDB Pricebook Engine (`Fraction OT pricebook`):** Runs for OpenTicket submissions. Loads buyer CSV pricebooks into an in-memory DuckDB table, cleans descriptions, validates rate bounds and expiry dates, and updates item IDs in the JSON.

6. **VPC AFE & Code Validation (`Fraction OI AFE validation`):** Executes inside an existing whitelisted AWS VPC to validate AFE numbers and accounting codes against OpenInvoice APIs before final submission.

7. **Final Output (`output to v3 lambda`):** Sends the completed JSON payload to the downstream submission Lambda.

---

## Engineering Challenges & Solutions

### 1. Sub-second CSV pricebook queries using in-memory DuckDB
* **The Challenge:** OpenTicket line items use internal Aimsio descriptions that do not match registered buyer pricebooks. Line items had to be validated against CSV pricebooks containing effective date ranges, rate limits, and unit types. Performing nested Python loops across large pricebooks inside Lambda was slow and inefficient.

* **The Solution:** Embedded DuckDB directly inside the `fraction_pricebook_mapping_complete` Lambda handler. On invocation, the Lambda loads the buyer CSV into an in-memory DuckDB table (`:memory:`), normalizes text descriptions with SQL regex functions, and filters out expired records. It then executes a weighted SQL query that ranks candidates by match weight, string length difference, and rate constraints to return the single best pricebook match.

### 2. Extracting visual department context using PyMuPDF and GPT-4o
* **The Challenge:** OpenInvoice requires a supplier department code, but Aimsio JSON files omit this field. The information is only visible as a graphical company logo on the PDF header.

* **The Solution:** Developed `fraction_logo_extractor` using PyMuPDF (`fitz`) and OpenAI vision models. The Lambda converts PDF pages into base64 JPEG images and submits them to GPT-4o. GPT-4o identifies the division logo and company name, then writes the department information back to the JSON stored in S3.

### 3. Recovering missing ticket headers from blanket invoice PDFs
* **The Challenge:** Aimsio blanket invoices group multiple tickets into one PDF. The exported JSON payload omits individual ticket headers, leaving downstream systems without ticket level dates or PO numbers.

* **The Solution:** Wrote `aimsio_extract_tickets_header_pdf` using PDFPlumber to inspect PDF text streams. The Lambda scans the PDF text labels across multi-page files, extracts missing ticket headers, and merges them into the S3 state JSON before downstream submission.

### 4. Integrating whitelisted API calls using an existing AWS VPC
* **The Challenge:** OpenInvoice API calls for AFE and accounting code validation fail if they do not originate from authorized static IP addresses.

* **The Solution:** Placed the AFE validation Lambda directly inside our existing AWS VPC configured with static Elastic IPs and NAT Gateways. This ensures all outgoing requests match vendor whitelists without changing the existing network architecture.
