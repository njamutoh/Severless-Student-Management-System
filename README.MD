# Serverless Student Management System on AWS with Terraform

## Overview

A fast-growing private educational institution became my client at a point when its student records workflow was no longer keeping up with operational growth. The school had expanded to **12,000+ active students**, with **800 to 1,000 new enrollments per intake cycle**, while administrative teams across admissions, records, and student support were still relying heavily on shared Excel spreadsheets.

That process had become increasingly difficult to manage. Multiple spreadsheet copies existed across teams, records could be overwritten or duplicated, and student lookups were getting slower as files grew larger and more fragmented. The client needed a more structured and scalable way to store and retrieve student records, but they did not want the overhead of managing traditional servers or a heavier infrastructure stack than the use case required.

To solve that, I designed and deployed a **serverless student management system on AWS**, provisioned with **Terraform**. The platform gives staff a browser-based interface to submit and retrieve student records while keeping the backend lightweight, scalable, and easier to maintain.

---

## What This Project Does

The application supports two core operations:

- Insert student data
- Retrieve stored student records

The frontend is delivered securely through **CloudFront**, backed by a **private S3 origin**. The backend uses **API Gateway**, **Lambda**, and **DynamoDB** to handle student record operations.

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

### Request paths

**Frontend delivery path**  
User → Route 53 → CloudFront → private S3

**Application data path**  
User → API Gateway → Lambda → DynamoDB

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
Terraform layout by responsibility
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
How the System Works
Frontend flow
User visits the custom domain
Route 53 resolves the domain to CloudFront
CloudFront terminates HTTPS using ACM
WAF inspects the request
CloudFront serves index.html and scripts.js
If needed, CloudFront fetches those files from the private S3 bucket using Origin Access Control
Backend flow
The deployed frontend script contains the live API Gateway URL
The browser sends POST /students to save student data
API Gateway invokes the insert Lambda function
Lambda writes the record to DynamoDB
The browser sends GET /students to retrieve records
API Gateway invokes the read Lambda function
Lambda fetches records from DynamoDB and returns them to the browser
Important note

CloudFront does not proxy the API in this project.
It serves only the static frontend. The browser talks to API Gateway directly for student data operations.

Why I Chose This Architecture

The client needed a solution that was:

easy for staff to use
lighter than a traditional server-based application
scalable enough for continued enrollment growth
more structured and reliable than spreadsheet-based workflows

A serverless design fit that need well.

Why these services
S3 stores static frontend files without needing a web server
CloudFront provides secure delivery, caching, and a public layer in front of a private bucket
WAF adds basic protection at the edge
Route 53 + ACM provide custom-domain access over HTTPS
API Gateway HTTP API is a simple fit for a small Lambda-backed API
Lambda removes the need for an always-on backend server
DynamoDB provides a managed data store without database server administration
Terraform makes the infrastructure reproducible and easier to maintain
Example Infrastructure Snippets
CloudFront with private S3 origin
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
Key Issues Encountered
1. Lambda zip artifacts were missing

The Lambda resources expected zipped deployment packages, but initially only the Python source files existed. Terraform failed because the referenced zip files did not exist yet.

Fix: created the deployment zip artifacts alongside the .py source files.

2. CloudFront returned AccessDenied

At first, the custom domain returned an XML AccessDenied response. The security model itself was correct: the bucket was private, public access was blocked, and website hosting was off. The actual issue was that the bucket had no frontend objects yet.

Fix: uploaded index.html and scripts.js through Terraform so CloudFront had content to serve.

3. CORS had to be handled correctly

The frontend was served through CloudFront, but the browser called API Gateway directly. That meant cross-origin requests had to be allowed properly.

Fix: configured the API to return the correct CORS headers for browser-based requests.

4. API Gateway needed explicit Lambda invoke permissions

Even with routes and integrations defined, API Gateway could not invoke Lambda unless the correct permission resources were present.

Fix: added explicit Lambda permissions for apigateway.amazonaws.com.

5. The deployed frontend needed the live API URL

A placeholder API endpoint in the frontend would cause the site to load while data operations failed.

Fix: rendered the deployed scripts.js with the live API Gateway URL through Terraform.

What I Learned

This project reinforced a few practical lessons:

A private S3 origin behind CloudFront is cleaner than using a public website bucket
API Gateway HTTP API is often a better fit than a heavier REST setup for small Lambda-backed applications
Terraform should generate deployment-time values like the live API URL instead of relying on manual edits
Visible errors like AccessDenied can hide a deeper origin-content issue
Even a relatively small serverless system still requires careful thinking across DNS, TLS, IAM, routing, packaging, and service-to-service permissions
Possible Improvements

Next improvements I would make:

add clearer Terraform outputs for frontend and API URLs
automate CloudFront invalidations for frontend updates
add stronger validation and error handling in the Lambda functions
add CI/CD for Terraform and packaging
improve student record retrieval patterns as the application grows
