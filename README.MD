# Serverless Student Management System on AWS with Terraform

## Overview

A rapidly growing private training institution with **8,000+ active students** and thousands of new enrollments each year was still managing student records through shared Excel spreadsheets. As the school expanded across departments and administrative teams, that manual process became a serious operational bottleneck. Staff were dealing with duplicate entries, overwritten records, version conflicts, and slow searches whenever they needed to retrieve student information. The client needed a more structured and scalable solution that could support continued growth without introducing the cost and maintenance burden of traditional servers.

To solve that, I designed and deployed a **serverless student management system on AWS**. The application allows staff to submit student details through a browser and retrieve stored student records through a simple web interface. The solution uses **Amazon S3, CloudFront, AWS WAF, ACM, Route 53, API Gateway, Lambda, DynamoDB, and Terraform** to deliver a low-maintenance, production-minded cloud architecture.

---

## Real-World Scenario

The client is a fast-growing educational institution with **8,000+ students already enrolled** and a projected path toward **10,000+ student records** as intake continues to increase. Despite that growth, student data was still being managed in spreadsheets shared among administrative staff.

That created several recurring problems:

- Multiple versions of the same spreadsheet were circulating across teams
- Staff occasionally overwrote or deleted records by mistake
- Looking up student records became slower as the files grew into the thousands
- There was no centralized backend for secure and structured record management
- The client wanted to modernize operations without paying for always-on servers or maintaining traditional infrastructure

They needed a solution that was:

- simple for staff to use
- inexpensive to run
- easy to scale as the student population grows
- more secure and maintainable than spreadsheet-based workflows

This project was built to meet that need with a serverless AWS design.

---

## Project Goal

Build a secure, low-cost student management system that can:

- save student details from a browser form
- retrieve stored student records
- avoid server management overhead
- use infrastructure as code for reproducible deployment

The final design provisions the infrastructure with Terraform and separates the application into a static frontend and a serverless backend.

---

## Architecture

The deployed solution uses the following AWS services:

- **Amazon S3** for static frontend hosting
- **Amazon CloudFront** for secure frontend delivery over HTTPS
- **AWS WAF** for basic edge protection
- **Amazon Route 53** for DNS
- **AWS Certificate Manager (ACM)** for TLS
- **Amazon API Gateway HTTP API** for backend endpoints
- **AWS Lambda** for business logic
- **Amazon DynamoDB** for student record storage
- **Terraform** for infrastructure provisioning and wiring

### Request Flow

1. A user visits the frontend through the custom domain.
2. Route 53 resolves the domain to CloudFront.
3. CloudFront uses ACM for TLS and WAF for edge-layer protection.
4. CloudFront serves the frontend from a private S3 bucket using Origin Access Control.
5. The frontend JavaScript calls API Gateway directly.
6. API Gateway invokes Lambda functions for read/write operations.
7. Lambda reads from or writes to the DynamoDB `studentData` table.

One important design detail is that **CloudFront serves only the frontend**. The browser calls **API Gateway directly** for student data operations.

---

## Architecture Diagram

![AWS Architecture Diagram](aws-architecture-drawio-style.png)

---

## Repository Structure

```text
AWS-SERVERLESS-DEPLOYMENT-main/
  getStudents.py
  insertStudentData.py
  index.html
  scripts.js

infra/
  acm.tf
  api.tf
  cloudfront.tf
  dynamodb.tf
  iam.tf
  lambda.tf
  providers.tf
  route53.tf
  s3.tf
  terraform.tfvars
  templates/
    scripts.js.tftpl
  variables.tf
  waf.tf
```

The application code lives separately from the Terraform code. The infra/ folder provisions the AWS infrastructure, while the application folder contains the Lambda source files and frontend assets.

What the Application Does

The application supports two core actions:

Insert student data
Retrieve student records

The frontend provides a simple form and table-based interface. On deployment, Terraform uploads the frontend files to S3 and injects the live API Gateway URL into the deployed JavaScript so the application points to the correct backend endpoint automatically.

Backend Logic

Two Lambda functions power the backend:

getStudents.py scans the DynamoDB table and returns stored records
insertStudentData.py accepts a request body and stores a student record

Both Lambda functions use the same IAM execution role with permissions scoped to DynamoDB access and CloudWatch logging.

Security and Delivery Design

I wanted the frontend bucket to remain private rather than exposing it as a public S3 website. To achieve that:

S3 public access is blocked
CloudFront accesses S3 through Origin Access Control (OAC)
the bucket policy allows only the CloudFront distribution to read the frontend objects

For public access and delivery:

Route 53 provides DNS records
ACM issues and validates the TLS certificate
CloudFront handles HTTPS termination
WAF applies the AWS managed common rule set at the edge

This results in a cleaner and more production-oriented setup than a public website bucket.

Why This Architecture Made Sense

This architecture fit the client’s needs because it removed the operational burden of:

provisioning EC2 instances
patching servers
maintaining a traditional backend host
running a self-managed relational database for a growing internal application

Instead, the application uses managed services that are easier to scale, automate, and secure for a use case like student registration and record lookup.

For a fast-growing institution managing thousands of student records, that tradeoff made practical sense: lower operational overhead, cleaner scaling, and a simpler support model for internal staff.

Deployment Notes

A few practical deployment details mattered in this implementation:

Lambda functions are deployed as zip artifacts
frontend files must be uploaded to S3 for CloudFront to serve them
the frontend needs the real API Gateway invoke URL after deployment
the actual S3 bucket name comes from frontend_bucket_name, not the Terraform resource label

One issue encountered during deployment was a CloudFront AccessDenied response. The root cause was not simply permissions; the S3 bucket was private and also empty, so CloudFront had no index.html or scripts.js object to serve. Uploading the frontend assets through Terraform resolved that issue.

Challenges I Encountered and How I Resolved Them
1. Lambda zip artifacts were missing

Terraform expected deployment packages for both Lambda functions, but only the Python source files existed initially. The fix was to create zip packages alongside the .py files and use those as the Lambda deployment artifacts while keeping the source code in the repository.

2. The initial API design was heavier than needed

The project originally leaned toward a more verbose REST-style API structure. After reviewing the actual use case, I simplified the backend to API Gateway HTTP API, which reduced complexity and made routing and CORS management more straightforward for this two-route application.

3. There was confusion about where the API endpoint belonged

At first, it seemed like the API URL might need to be manually managed in multiple places. Reviewing the request flow made it clear that the public API endpoint belongs to the frontend, while Lambda only needs internal values like the DynamoDB table name. I resolved this by having Terraform inject the real API URL into the deployed frontend script.

4. CloudFront returned AccessDenied

The frontend was initially inaccessible through the custom domain. Investigation showed that the bucket was private as intended, but the real problem was that there were no frontend objects uploaded for CloudFront to fetch. Uploading index.html and scripts.js fixed the issue.

5. Terraform resource labels were confused with actual AWS names

The Terraform resource label for the S3 bucket was not the same as the deployed AWS bucket name. I verified the real value came from frontend_bucket_name and documented that distinction to make troubleshooting clearer.

6. Too many variables made the Terraform harder to reason about

As the project grew, some variables added indirection without improving flexibility. I simplified the configuration by inlining fixed values and keeping only genuinely useful inputs such as aws_region, students_table_name, and default_root_object.

7. The local frontend script was not identical to the deployed one

The repository’s local scripts.js contained a placeholder API URL. The deployed version is generated from a Terraform template so it receives the real API endpoint at deploy time. That kept the source code readable while ensuring the live application points to the correct backend URL.

8. CloudFront serves the frontend, not the API

Part of documenting the solution correctly was clarifying that CloudFront is only in front of the static frontend. API requests go from the browser straight to API Gateway. Making that boundary explicit improved both troubleshooting and documentation.

What I Learned

This project reinforced several practical cloud engineering lessons:

private S3 + CloudFront + OAC is a cleaner secured frontend pattern than public S3 website hosting
API Gateway HTTP API is a better fit than REST API for small Lambda-backed applications
Terraform should derive values like the live API URL from created resources rather than rely on manual copy-paste
even a simple serverless application still requires careful wiring across DNS, certificates, CDN behavior, API routing, IAM, and object delivery
Future Improvements

If I continue evolving the project, the next improvements would be:

add Terraform outputs for the frontend and API URLs
add CloudFront invalidation automation for frontend changes
replace the current lightweight frontend with a more modern client
add stronger validation and error handling in the Lambda functions
add CI/CD for Terraform, Lambda packaging, and frontend deployment
