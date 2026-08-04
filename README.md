# Aimsio Document Ingestion & Intelligent Pricebook Engine
## Serverless Document Processing with Vision AI, DuckDB, and AWS Step Functions

### Tech Stack & Tools
* **Orchestration & Compute:** AWS Step Functions, AWS Lambda (Python 3.13 & JavaScript)
* **Embedded Database Engine:** DuckDB (In-Memory SQL)
* **Vision AI & Data Parsing:** OpenAI API (GPT-4o Vision), PyMuPDF (`fitz`), PDFPlumber, Regular Expressions (`re`)
* **Cloud Infrastructure & Networking:** Amazon S3, AWS VPC (Static Whitelisted IPs), AWS Boto3
* ****Security & Authentication:** Mutual TLS (mTLS) Client Certificates

---

## Business Requirements

At EMI, we automate field service ticket and invoice submissions for oil and gas suppliers to platforms including OpenInvoice and OpenTicket. Many client suppliers manage field operations using an accounting software called Aimsio. When Aimsio creates a ticket or invoice, it generates a raw JSON file alongside an attached PDF copy.

Aimsio exports data into three distinct JSON payload formats depending on whether field staff save a transaction as a ticket, an invoice, or a blanket invoice. Upstream users select the ticket or invoice format based on whether the downstream buyer expects an OpenTicket or OpenInvoice submission. About 99% of the data required is provided and can be directly processed in the structured JSON. However, several pieces of information required by downstream platforms are omitted from the JSON and are only available in the attached PDF documents.

To prepare each submission, the pipeline performs four processing steps:

* **Determine the supplier department:** The JSON export does not include the supplier department. Instead, the selected department is indicated by the logo shown in the PDF header, so the pipeline identifies the logo to determine which supplier department to submit as.
  
* **Recover ticket information:** Blanket invoice JSON exports omit the individual ticket details. The pipeline extracts the missing information from the attached ticket PDFs before submission.
  
* **Match buyer pricebooks:** Aimsio item descriptions differ from buyer approved pricebooks in OpenTicket, so each line item must be matched against the correct buyer pricebook using the description, rate, and unit before submission.

* **Validate AFE and accounting codes:** Before submitting to OpenInvoice, the pipeline validates AFE (Authorization for Expenditure) through the platform API to reduce submission failures caused by invalid or mistyped values.

---

## Engineering Challenges & Solutions

### 1. Recovering missing ticket headers from blanket invoice PDFs
* **The Challenge:** When Aimsio exports a blanket invoice, it generates one invoice JSON but leaves out the individual ticket details. Those details are still available in the attached ticket PDFs and are required for downstream processing.

* **The Solution:** I wrote `aimsio_extract_tickets_header_pdf` using PDFPlumber. The Lambda scans PDF text, extracts the missing ticket headers, and merges them into the S3 state JSON before downstream submission.

### 2. Extracting visual department context using PyMuPDF and GPT-4o
* **The Challenge:** OpenInvoice requires a supplier department, but Aimsio does not include it in the JSON export. Instead, the selected department is indicated by the logo shown in the header of each page of the attached tickets PDF.

* **The Solution:** I developed `fraction_logo_extractor` using PyMuPDF (`fitz`) and OpenAI GPT-4o Vision. The Lambda converts PDF pages into base64 JPEG images, sends them to GPT-4o Vision, identifies the division logo and company name, then writes the department information back to the JSON stored in S3.

### 3. Sub-second CSV pricebook queries using in-memory DuckDB
* **The Challenge:** OpenTicket line items use internal Aimsio descriptions that do not match registered buyer pricebooks. Line items must be matched against CSV pricebooks containing effective date ranges, rate limits, and unit types. Searching large CSV pricebooks with nested Python loops would require repeatedly scanning every record for every invoice line item. That approach became slower as pricebooks grew.

* **The Solution:** I embedded DuckDB directly inside the Lambda. Each invocation loads the buyer's CSV pricebook into an in memory table, filters expired entries, normalizes text descriptions with SQL regex functions, and runs a weighted SQL query to return the best match.

### 4. Validating AFE codes before OpenInvoice submission
* **The Challenge:** OpenInvoice validates AFE and accounting codes during submission. Mistyped or invalid values cause the submission to fail. Because invoices are also grouped by AFE, the extracted values must be correct before the pipeline splits and submits the invoice.

* **The Solution:** I added a validation Lambda that queries the OpenInvoice API for every extracted AFE and accounting code before submission. The function runs inside an existing VPC with static outbound IP addresses that satisfy the platform's whitelist requirements. Client certificates are loaded from SSM Parameter Store during cold starts and written to temporary files under /tmp, allowing the Python requests library to perform mutual TLS authentication efficiently.

---

## Architecture & Data Flow

I built the AWS Step Functions state machine and core processing Lambdas, coordinating data flow across S3, OpenAI, DuckDB, and platform APIs.

The diagram below shows a typical execution of the pipeline. Each state displays its input and output, making it easy to trace data through the workflow. During debugging, developers can open the related Lambda or CloudWatch logs directly from the execution view, then retry a failed state or rerun the entire workflow after making changes.

<img width="1606" height="719" alt="fractionstepfunctions_graph" src="https://github.com/user-attachments/assets/245cd97c-7ca4-4fa6-b266-d1a063c00801" />


### Data Processing Steps

1. **Format Routing (`Check isOTFormat`):** Determines whether the submission is an OpenTicket ticket, an OpenInvoice invoice, or a blanket ticket. The pipeline then routes the document through the required processing steps.

2. **Aimsio PDF Header Parsing (`extract Aimsio tickets header from PDF`):** Reads the PDF from S3 using PDFPlumber to extract ticket level information that is not included in blanket invoice JSON exports and merges it into the processing state.

3. **Multimodal Vision Processing (`chatGpt reads logos`):** Converts PDF pages to JPEG images using PyMuPDF and passes the images to OpenAI GPT-4o to identify the supplier logo to determine which supplier department the document should be submitted as.

4. **Pipeline Branching (`Check isOT`):** Routes OpenTicket submissions to the pricebook engine and OpenInvoice submissions to VPC validation.

5. **In-Memory DuckDB Pricebook Engine (`Fraction OT pricebook`):** Runs for OpenTicket submissions. Loads the buyer CSV pricebook into an in-memory DuckDB table, cleans descriptions, validates effective dates, rate bounds, and units, then updates item IDs in the JSON.

6. **VPC AFE & Code Validation (`Fraction OI AFE validation`):** Runs inside an existing whitelisted AWS VPC to validate AFE numbers and accounting codes against OpenInvoice APIs.

7. **Final Output (`output to v3 lambda`):** Sends the completed JSON payload to the downstream submission Lambda.

