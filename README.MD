# Serverless Student Management System on AWS with Terraform

## Overview

A fast-growing private educational institution became my client at a point when its student records workflow was no longer keeping up with operational growth. The school had expanded to **12,000+ active students**, with **800 to 1,000 new enrollments per intake cycle**, while administrative teams across admissions, records, and student support were still relying heavily on shared Excel spreadsheets to manage student information.

That process had become increasingly difficult to manage. Multiple spreadsheet copies existed across teams, records could be overwritten or duplicated, and student lookups were getting slower as files grew larger and more fragmented. What had once been “good enough” for a smaller operation had become an operational risk as the institution scaled. The client needed a more structured and scalable way to store and retrieve student records, but they did not want the overhead of managing traditional servers or a heavier infrastructure stack than the use case required.

To solve that, I designed and deployed a **serverless student management system on AWS**, provisioned with **Terraform**. The platform gives staff a browser-based interface to submit and retrieve student records while keeping the backend lightweight, scalable, and easier to maintain than a traditional VM-based application.

---

## What This Project Does

The application supports two core operations:

- Insert student data
- Retrieve stored student records

From the user’s perspective, the workflow is intentionally simple. Staff access a browser-based interface, enter student information, and retrieve records from a central backend rather than depending on spreadsheet copies shared across teams.

The frontend is delivered securely through **CloudFront**, backed by a **private S3 origin**. The backend uses **API Gateway**, **Lambda**, and **DynamoDB** to handle student record operations. The result is a serverless architecture that removes the need to manage always-on servers while still providing a practical internal tool.

---

## Architecture Summary

This project uses the following AWS services:

- **Route 53** for DNS
- **CloudFront** as the public frontend delivery layer
- **WAF** for request inspection at the edge
- **ACM** for TLS certificates
- **S3** as a **private origin** for frontend assets
- **API Gateway HTTP API** for backend endpoints
- **Lambda** for application logic
- **DynamoDB** for student record storage
- **IAM** for service permissions
- **Terraform** for infrastructure provisioning

### Key design point

**S3 is not used as a public website bucket.**

Instead:

- **CloudFront is the public front door**
- **S3 stores the frontend files privately**
- **Only CloudFront can read from the bucket**
- **The browser calls API Gateway directly for data operations**

That distinction matters because it changes both the security model and the delivery model of the application. Users never browse the bucket directly. They access the application through CloudFront, which fetches the frontend files from S3 behind the scenes.

### Request paths

**Frontend delivery path**  
User → Route 53 → CloudFront → private S3

**Application data path**  
User → API Gateway → Lambda → DynamoDB

This separation made the stack easier to secure, easier to reason about, and easier to document.

---

## Architecture Diagram

![AWS Architecture Diagram](serverless_student_management_system_architecture.png)

**Caption:** High-level architecture of the serverless student management platform, showing the frontend delivery path through Route 53, CloudFront, and a private S3 origin, and the backend data path through API Gateway, Lambda, and DynamoDB.

---

## Project Structure

```text
project-root/
├── AWS-SERVERLESS-DEPLOYMENT-main/
│   ├── getStudents.py
│   ├── insertStudentData.py
│   ├── index.html
│   └── scripts.js
│
└── infra/
    ├── acm.tf
    ├── api.tf
    ├── cloudfront.tf
    ├── dynamodb.tf
    ├── iam.tf
    ├── lambda.tf
    ├── providers.tf
    ├── route53.tf
    ├── s3.tf
    ├── terraform.tfvars
    ├── variables.tf
    ├── waf.tf
    └── templates/
        └── scripts.js.tftpl
```

The project is split into two major parts:

AWS-SERVERLESS-DEPLOYMENT-main/ contains the Lambda source code and frontend assets
infra/ contains the Terraform configuration that provisions and wires the AWS services together

This separation made it easier to maintain the codebase, troubleshoot issues, and explain what belonged to the application versus the infrastructure.

Terraform layout by responsibility

```text
infra/
├── providers.tf      # provider configuration
├── variables.tf      # input variables
├── terraform.tfvars  # deployment values
├── route53.tf        # hosted zone lookup and DNS records
├── acm.tf            # certificate request and DNS validation
├── s3.tf             # private frontend bucket and object uploads
├── cloudfront.tf     # OAC, distribution, bucket policy
├── waf.tf            # WAF web ACL
├── api.tf            # API Gateway HTTP API, routes, stage
├── iam.tf            # IAM role, policies, Lambda permissions
├── lambda.tf         # Lambda functions and environment variables
├── dynamodb.tf       # student data table
└── templates/
    └── scripts.js.tftpl
```

## How the System Works

### Frontend flow

1. User visits the custom domain.
2. Route 53 resolves the domain to CloudFront.
3. CloudFront terminates HTTPS using ACM.
4. WAF inspects the request.
5. CloudFront serves `index.html` and `scripts.js`.
6. If needed, CloudFront fetches those files from the private S3 bucket using **Origin Access Control (OAC)**.

### Backend flow

1. The deployed frontend script contains the live API Gateway URL.
2. The browser sends `POST /students` to save student data.
3. API Gateway invokes the insert Lambda function.
4. Lambda writes the record to DynamoDB.
5. The browser sends `GET /students` to retrieve records.
6. API Gateway invokes the read Lambda function.
7. Lambda fetches records from DynamoDB and returns them to the browser.

### Important note

**CloudFront does not proxy the API in this project.**  
It serves only the static frontend. The browser talks to **API Gateway directly** for student data operations.

That was an important architectural boundary to keep clear, especially when troubleshooting frontend behavior versus backend request handling.

---

## Why I Chose This Architecture

The client needed a solution that was:

- easy for staff to use
- lighter than a traditional server-based application
- scalable enough for continued enrollment growth
- more structured and reliable than spreadsheet-based workflows

A serverless design fit that need well because it reduced operational overhead while still providing a clean application experience.

### Why these services

- **S3** stores static frontend files without needing a web server.
- **CloudFront** provides secure delivery, caching, and a public layer in front of a private bucket.
- **WAF** adds basic protection at the edge.
- **Route 53 + ACM** provide custom-domain access over HTTPS.
- **API Gateway HTTP API** is a simple fit for a small Lambda-backed API.
- **Lambda** removes the need for an always-on backend server.
- **DynamoDB** provides a managed data store without database server administration.
- **Terraform** makes the infrastructure reproducible and easier to maintain.

This combination gave the client a system that was modern enough to replace spreadsheet-based operations, while still being appropriately sized for the real scope of the problem.

---

## How the Services Work Together

One of the cleanest ways to understand the system is to separate **provisioning-time communication** from **runtime communication**.

### Provisioning-time flow

At apply time, Terraform coordinated the system in roughly this order:

1. Requested the ACM certificate.
2. Created the Route 53 validation records.
3. Waited for certificate validation.
4. Created the private S3 bucket.
5. Created the CloudFront distribution.
6. Attached ACM and WAF to CloudFront.
7. Created Origin Access Control.
8. Created the bucket policy allowing only that distribution to read objects.
9. Created the HTTP API, routes, integrations, and stage.
10. Created the Lambda functions, IAM role, and permissions.
11. Created the DynamoDB table.
12. Generated the deployed frontend script with the live API URL.
13. Uploaded `index.html` and `scripts.js` to S3.

Terraform is effectively the coordinator here. The AWS services are not discovering each other automatically. Terraform wires them together by reference and ensures the dependencies are created in the right order.

### Runtime communication summary

**Static path:**

User → Route 53 → CloudFront → private S3

**Application data path:**

User → API Gateway → Lambda → DynamoDB

### Security and control layers

- Route 53 maps the domain to CloudFront.
- ACM gives CloudFront TLS capability.
- WAF filters requests at the CloudFront layer.
- OAC lets CloudFront fetch from private S3.
- IAM authorizes Lambda and API interactions.

---

## Frontend and Backend Responsibilities

From the user’s perspective, the application supports two core actions:

- insert student data
- retrieve student records

### Frontend responsibilities

The frontend provides:

- a form for entering student data
- the ability to trigger record retrieval
- a browser-based interface for viewing the returned data

The deployed frontend does not rely on a hardcoded API placeholder. Instead, Terraform renders the deployed `scripts.js` file with the real API Gateway URL so the hosted frontend points to the correct backend automatically.

### Backend responsibilities

The backend logic is handled by two Lambda functions:

- `getStudents.py` scans the DynamoDB table and returns records
- `insertStudentData.py` accepts the request body and stores a student record

Both functions use the same IAM execution role, with permissions scoped to CloudWatch logging and DynamoDB access. This kept the compute layer small, stateless, and focused only on request handling and data interaction.

---

## Security and Delivery Design

I wanted the frontend delivery path to follow a more production-minded pattern than exposing a public bucket.

So the design used:

- blocked public access on S3
- no S3 website hosting
- CloudFront Origin Access Control
- a bucket policy allowing only that CloudFront distribution to read objects
- TLS termination at CloudFront
- WAF at the edge
- Route 53 and ACM for domain and certificate handling

That approach made the frontend delivery path cleaner and better controlled. It also meant the S3 bucket could remain private while still serving the application through CloudFront.

---

## Example Infrastructure Snippets

### CloudFront with private S3 origin

```hcl
resource "aws_cloudfront_origin_access_control" "oac" {
  name                              = "student-management-oac"
  origin_access_control_origin_type = "s3"
  signing_behavior                  = "always"
  signing_protocol                  = "sigv4"
}

resource "aws_cloudfront_distribution" "frontend" {
  origin {
    domain_name              = aws_s3_bucket.student_frontend.bucket_regional_domain_name
    origin_access_control_id = aws_cloudfront_origin_access_control.oac.id
    origin_id                = "studentFrontendS3"
  }

  default_root_object = "index.html"
}
API routes
resource "aws_apigatewayv2_api" "students_api" {
  name          = "students-api"
  protocol_type = "HTTP"
}

resource "aws_apigatewayv2_route" "get_students" {
  api_id    = aws_apigatewayv2_api.students_api.id
  route_key = "GET /students"
  target    = "integrations/${aws_apigatewayv2_integration.get_students.id}"
}

resource "aws_apigatewayv2_route" "insert_student" {
  api_id    = aws_apigatewayv2_api.students_api.id
  route_key = "POST /students"
  target    = "integrations/${aws_apigatewayv2_integration.insert_student.id}"
}
```

These snippets reflect the two most important architectural relationships in the project: CloudFront serving the frontend from a private S3 origin, and API Gateway routing browser requests to Lambda-backed backend operations.

## Key Issues Encountered

### 1. Lambda zip artifacts were missing

The Lambda resources expected zipped deployment packages, but initially only the Python source files existed. Terraform failed because the referenced zip files did not exist yet.

**Fix:** Created the deployment zip artifacts alongside the `.py` source files.

### 2. CloudFront returned `AccessDenied`

At first, the custom domain returned an XML `AccessDenied` response. The security model itself was correct: the bucket was private, public access was blocked, and website hosting was off. The actual issue was that the bucket had no frontend objects yet.

**Fix:** Uploaded `index.html` and `scripts.js` through Terraform so CloudFront had content to serve.

### 3. CORS had to be handled correctly

The frontend was served through CloudFront, but the browser called API Gateway directly. That meant cross-origin requests had to be allowed properly.

**Fix:** Configured the API to return the correct CORS headers for browser-based requests.

### 4. API Gateway needed explicit Lambda invoke permissions

Even with routes and integrations defined, API Gateway could not invoke Lambda unless the correct permission resources were present.

**Fix:** Added explicit Lambda permissions for `apigateway.amazonaws.com`.

### 5. The deployed frontend needed the live API URL

A placeholder API endpoint in the frontend would cause the site to load while data operations failed.

**Fix:** Rendered the deployed `scripts.js` with the live API Gateway URL through Terraform.

These issues were valuable because they moved the project beyond a basic happy-path deployment and forced careful thinking about packaging, permissions, browser behavior, and service boundaries.

---

## What I Learned

This project reinforced a few practical lessons:

- A private S3 origin behind CloudFront is cleaner than using a public website bucket.
- API Gateway HTTP API is often a better fit than a heavier REST setup for small Lambda-backed applications.
- Terraform should generate deployment-time values like the live API URL instead of relying on manual edits.
- Visible errors like `AccessDenied` can hide a deeper origin-content issue.
- Even a relatively small serverless system still requires careful thinking across DNS, TLS, IAM, routing, packaging, and service-to-service permissions.

It also reinforced how important it is to understand not just what each AWS service does individually, but how they interact once the system is actually live.

---

## Possible Improvements

Next improvements I would make:

- add clearer Terraform outputs for frontend and API URLs
- automate CloudFront invalidations for frontend updates
- add stronger validation and error handling in the Lambda functions
- add CI/CD for Terraform and packaging
- improve student record retrieval patterns as the application grows

These would make the project more robust operationally and closer to a repeatable production workflow.w.
