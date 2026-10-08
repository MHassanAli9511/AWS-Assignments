# Assignment 3 – S3 Static Website, CloudFront CDN and CI/CD

## Overview

In this assignment, I built and deployed a static website using Amazon S3 and placed Amazon CloudFront in front of it as a Content Delivery Network (CDN).

I then configured HTTPS using AWS Certificate Manager (ACM), connected a custom domain using Cloudflare DNS, tested CloudFront caching and invalidation, and implemented an automated deployment pipeline using GitHub Actions.

The main architecture was:

```text
User
  ↓
Custom Domain / DNS
  ↓
CloudFront
  ↓
S3 Static Website
```

I also implemented a CI/CD workflow:

```text
GitHub Push
    ↓
GitHub Actions
    ↓
AWS Authentication
    ↓
Sync Website Files to S3
    ↓
Invalidate CloudFront Cache
```

<!-- IMAGE: Final architecture diagram -->

---

## 1. Creating the S3 Bucket

I first created an Amazon S3 bucket to store the static website files.

I used a General Purpose bucket because it supports normal object storage features such as:

- Static website hosting
- Bucket policies
- HTML files
- Standard S3 objects

<!-- IMAGE: S3 bucket configuration -->

I used the traditional global S3 namespace, which means the bucket name must be globally unique.

---

## 2. Starting with Secure Defaults

I initially created the bucket using AWS's secure default settings.

By default, S3 blocks public access.

This is important because S3 buckets can contain sensitive data such as backups, application data, logs and customer information.

Rather than immediately making the bucket public, I first created the bucket securely and then deliberately enabled only the permissions required for the static website.

<!-- IMAGE: S3 bucket created -->

---

## 3. Enabling Static Website Hosting

Under the bucket properties, I enabled:

`Static website hosting`

This allows S3 to act as a basic web server and serve static files such as HTML and images.

<!-- IMAGE: Static website hosting configuration -->

I configured:

- Index document: `index.html`
- Error document: `error.html`

The index document is served when a visitor accesses the website without specifying a particular file.

The error document is returned when a requested page does not exist.

---

## 4. Creating the Website Files

I created two HTML files:

```text
index.html
error.html
```

The `index.html` file contained the main website content.

The `error.html` file was used as the custom error page.

I created these files locally and then uploaded them to the S3 bucket.

<!-- IMAGE: index.html and error.html uploaded -->

---

## 5. Configuring Public Access

Uploading the files alone does not make the static website publicly accessible.

S3 enables Block Public Access by default.

Because this bucket was being intentionally used as a public static website, I changed the bucket's public access settings and configured a bucket policy.

<!-- IMAGE: S3 Block Public Access configuration -->

The bucket policy allowed:

`s3:GetObject`

for the website files.

<!-- IMAGE: S3 bucket policy -->

The important parts of the policy were:

- `Effect: Allow` – allows the action
- `Principal: *` – allows public access
- `Action: s3:GetObject` – allows objects to be read
- `Resource: bucket/*` – applies the permission to files inside the bucket

---

## 6. Troubleshooting the Bucket Policy

While creating the policy, I initially received an error because the `Principal` field was missing.

The `Principal` determines who receives the permission.

Because this assignment required the website objects to be publicly readable, I configured:

```json
"Principal": "*"
```

After correcting this, the bucket policy saved successfully.


This helped me understand that an IAM or resource policy needs both an action and a defined principal where appropriate.

---

## 7. Testing the S3 Static Website

I went to:

`S3 → Properties → Static website hosting`

and opened the bucket website endpoint.

<!-- IMAGE: S3 website endpoint -->

The website loaded successfully and displayed the contents of `index.html`.

<!-- IMAGE: S3 static website working -->

This confirmed that S3 was correctly serving the static website.

---

# CloudFront CDN

## 8. Creating the CloudFront Distribution

I then created an Amazon CloudFront distribution.

CloudFront acts as a CDN and can cache website content at edge locations closer to users.

Instead of using the normal S3 REST endpoint, I used the:

`S3 website endpoint`

as the CloudFront origin.

This is because the website endpoint represents S3 acting as a web server.

<!-- IMAGE: CloudFront distribution creation -->

---

## 9. Selecting the S3 Website Endpoint

I selected my S3 bucket and chose:

`Use website endpoint`

<!-- IMAGE: CloudFront S3 website endpoint selection -->

The difference I learned was:

```text
S3 website endpoint
= S3 acting as a website host

S3 REST endpoint
= S3 acting as an object-storage API
```

For this assignment, the website endpoint was the correct origin.

---

## 10. Configuring CloudFront Origin Settings

I customised the CloudFront origin settings.

The S3 static website endpoint used:

`HTTP only`

between CloudFront and the origin.

<!-- IMAGE: CloudFront origin HTTP configuration -->

This is because S3 website endpoints do not provide HTTPS directly to CloudFront.

CloudFront itself can still provide HTTPS to visitors.

---

## 11. Configuring Cache Behaviour

I configured the CloudFront cache behaviour with:

- Viewer protocol policy: Redirect HTTP to HTTPS
- Allowed HTTP methods: GET, HEAD
- Cache policy: CachingOptimized

<!-- IMAGE: CloudFront cache settings -->

`Redirect HTTP to HTTPS` means visitors using an HTTP URL are redirected to the secure HTTPS version.

Allowing only `GET` and `HEAD` is appropriate because users only need to retrieve static content.

The `CachingOptimized` policy allows CloudFront to efficiently cache the website content.

---

## 12. Testing CloudFront

After deploying CloudFront, I accessed the CloudFront distribution URL.

<!-- IMAGE: CloudFront website test -->

The website loaded successfully.

This confirmed the following request path was working:

```text
Browser
   ↓
CloudFront
   ↓
S3 Static Website Endpoint
```

I also confirmed that HTTP traffic redirected to HTTPS.

---

# CloudFront Caching

## 13. Testing Cached Content

To test CloudFront caching, I modified my local:

`index.html`

file and uploaded the new version to S3.

When I opened the S3 website endpoint, I could immediately see the updated content.

<!-- IMAGE: Updated S3 website -->

However, when I accessed the CloudFront URL, CloudFront was still displaying the previous version.

<!-- IMAGE: Old CloudFront cached version -->

This happened because CloudFront was serving the cached copy of the file.

---

## 14. CloudFront Invalidation

To force CloudFront to fetch the latest version of `index.html`, I created a CloudFront invalidation.

<!-- IMAGE: CloudFront invalidation -->

A CloudFront invalidation tells CloudFront to stop serving the cached copy of a specified file.

For example:

```text
/index.html
```

After the invalidation, CloudFront retrieved the latest version of the file from S3.

CloudFront invalidations are useful when:

- Content has been updated
- An urgent fix has been deployed
- The cached TTL has not yet expired

---

## 15. Versioned Files and Caching

I also learned that production environments can reduce the need for invalidations by using versioned filenames.

For example:

```text
app-v1.js
```

can be replaced with:

```text
app-v2.js
```

CloudFront sees the second filename as a completely new object and therefore fetches it from the origin.

This means:

```text
Same filename
→ cached version may exist
→ invalidation may be required

New filename
→ CloudFront sees a new object
→ latest file is fetched
```

---

# Custom Domain and HTTPS

## 16. Route 53 Domain Registration Attempt

The original plan was to register a custom domain using Amazon Route 53.

Route 53 is AWS's DNS service and allows a human-readable domain name to point to AWS resources.

I attempted to register a new domain through Route 53.

<!-- IMAGE: Route 53 domain registration attempt -->

However, I encountered a problem during AWS domain registration and submitted an enquiry to AWS Support.

Rather than blocking the rest of the assignment, I used an existing domain that I already owned through Cloudflare.

This allowed me to continue implementing the custom-domain and HTTPS requirements.

---

## 17. Existing Cloudflare Domain

I used my existing domain:

`ha9511.co.uk`

and selected the hostname:

`www.ha9511.co.uk`

<!-- IMAGE: Cloudflare domain -->

The resulting architecture became:

```text
www.ha9511.co.uk
       ↓
Cloudflare DNS
       ↓
CloudFront
       ↓
S3
```

---

## 18. Requesting an ACM Certificate

I used AWS Certificate Manager (ACM) to request an SSL/TLS certificate for the custom domain.

<!-- IMAGE: ACM certificate request -->

ACM provides certificates that allow custom domains to use HTTPS securely.

The certificate also required DNS validation to prove that I controlled the domain.

---

## 19. DNS Certificate Validation

AWS provided a DNS CNAME record for validating the certificate.

I copied the supplied CNAME name and value into Cloudflare DNS.

<!-- IMAGE: Cloudflare certificate validation CNAME -->

Once the DNS record was available, AWS was able to validate the domain.

The ACM certificate changed to:

`Issued`

<!-- IMAGE: ACM certificate issued -->

This confirmed that AWS successfully verified control of the domain.

---

## 20. Attaching the Certificate to CloudFront

I edited the CloudFront distribution and added:

`www.ha9511.co.uk`

as an alternate domain name.

I then selected the ACM certificate as the CloudFront custom SSL certificate.

This allowed CloudFront to serve the website securely over HTTPS using my own domain name.

---

## 21. Pointing the Custom Domain to CloudFront

The final DNS step was to make:

`www.ha9511.co.uk`

point to the CloudFront distribution.

In Cloudflare DNS, I configured the `www` record to use the CloudFront distribution domain as its CNAME target.

<!-- IMAGE: Cloudflare CNAME pointing to CloudFront -->

The flow was now:

```text
www.ha9511.co.uk
       ↓
Cloudflare DNS
       ↓
CloudFront
       ↓
S3
```

---

## 22. Testing the Custom Domain

I accessed:

`https://www.ha9511.co.uk`

in the browser.

<!-- IMAGE: Custom domain website working -->

The static website loaded successfully.

This confirmed that:

- Cloudflare resolved the domain
- CloudFront served the website
- ACM provided HTTPS
- S3 remained the website origin

---

# Bonus – GitHub Actions CI/CD

## 23. Automating Website Deployment

I then created a basic CI/CD workflow using GitHub Actions.

Instead of manually uploading a new `index.html` file to S3 every time the website changed, I wanted deployments to happen automatically.

The new deployment flow was:

```text
Edit Website
     ↓
Push to GitHub
     ↓
GitHub Actions
     ↓
Upload Files to S3
     ↓
Invalidate CloudFront Cache
```

<!-- IMAGE: Custom domain website working2 -->

---

## 24. Creating an IAM User for GitHub Actions

GitHub Actions needed permission to interact with AWS.

I created a dedicated IAM user for the deployment workflow.

<!-- IMAGE: GitHub Actions IAM user -->

I then created a custom IAM policy that gave the user only the AWS permissions needed by the deployment process.

<!-- IMAGE: GitHub Actions IAM permissions -->

The workflow required permission to:

1. Upload and update website files in S3
2. Invalidate the CloudFront distribution

---

## 25. Connecting GitHub Actions to AWS

I created AWS access credentials for the deployment user.

The credentials were stored as GitHub repository secrets rather than being written directly into the workflow file.

<!-- IMAGE: GitHub repository secrets -->

The workflow could then authenticate to AWS using those secrets.

> AWS access keys and secret access keys should never be committed directly into the repository.

---

## 26. Creating the GitHub Actions Workflow

I created a GitHub Actions workflow called:

`deploy.yml`

<!-- IMAGE: GitHub Actions workflow -->

The workflow was configured so that every push to:

`main`

triggered the deployment.

The workflow:

1. Checked out the repository
2. Authenticated to AWS
3. Synced the website files to S3
4. Created a CloudFront invalidation

The deployment pipeline became:

```text
GitHub Push
    ↓
GitHub Actions
    ↓
Authenticate to AWS
    ↓
Sync Website Files to S3
    ↓
Invalidate CloudFront
```

---

## 27. Troubleshooting the GitHub Actions Deployment

When I first triggered the GitHub Actions workflow, it failed.

<!-- IMAGE: GitHub Actions deployment failed -->

The error showed that the IAM user was not authorised to perform:

`s3:ListBucket`

I checked the IAM policy and found that the resource configuration was incorrect.

<!-- IMAGE: Incorrect IAM policy -->

I updated the policy so that the correct S3 bucket and object resources were used.

<!-- IMAGE: Corrected IAM policy -->

After fixing the policy, I triggered the workflow again.

This time the deployment succeeded.

<!-- IMAGE: Successful GitHub Actions deployment -->

This demonstrated how IAM permissions directly affect automated AWS workflows.

---

## 28. Testing the Automated Deployment

To test the completed deployment pipeline, I edited the website heading in `index.html`.

I changed the content and committed the update to GitHub.

<!-- IMAGE: Website code changed in GitHub -->

The commit triggered the GitHub Actions workflow.

The workflow:

```text
Push
 ↓
GitHub Actions
 ↓
S3 Updated
 ↓
CloudFront Invalidated
```

I then opened the live website.

<!-- IMAGE: Website updated from GitHub -->

The website displayed the new heading:

`Changed from GitHub`

This confirmed that the automated deployment pipeline was working successfully.

---

# What I Learned

Through this assignment, I developed a better understanding of:

- Amazon S3
- S3 static website hosting
- Public access settings
- S3 bucket policies
- `s3:GetObject`
- Amazon CloudFront
- CDN caching
- CloudFront invalidations
- HTTP to HTTPS redirection
- Cache policies
- Route 53 and DNS concepts
- AWS Certificate Manager
- DNS certificate validation
- CNAME records
- Cloudflare DNS
- Custom domains
- HTTPS certificates
- GitHub Actions
- CI/CD deployment workflows
- IAM permissions for automated deployments
- GitHub repository secrets
- Troubleshooting IAM access errors

I also learned how different services work together to deliver a website securely and efficiently.

---

# Challenges and How I Solved Them

## Route 53 Domain Registration

I initially attempted to register a new domain through Route 53.

The registration process failed, so I submitted an enquiry to AWS.

Instead of allowing this issue to stop the rest of the assignment, I used an existing domain that I owned through Cloudflare.

I configured Cloudflare DNS to point the custom domain to CloudFront and used ACM for HTTPS.

This allowed me to successfully complete the custom-domain configuration despite the Route 53 registration problem.

## S3 Bucket Policy

While creating the public-read bucket policy, I initially received an error because the `Principal` field was missing.

I identified that the policy needed to define who was receiving the permission.

Because the static website needed to be publicly readable, I configured the principal as `*`.

The policy then saved successfully.

## GitHub Actions IAM Permission

My first GitHub Actions deployment failed because the IAM user did not have the correct `s3:ListBucket` permission.

I checked the IAM policy and found that the resource configuration was incorrect.

After correcting the S3 resource ARNs and permissions, I reran the workflow and it completed successfully.

---

# Testing and Validation

I tested the project throughout the build.

I first confirmed that the S3 static website endpoint successfully served `index.html`.

I then verified that CloudFront could retrieve the website from the S3 website endpoint and redirect HTTP requests to HTTPS.

I tested CloudFront caching by changing `index.html`, confirming that CloudFront initially returned the old cached version, and then creating an invalidation so the updated version appeared.

I configured the custom domain and confirmed that `www.ha9511.co.uk` successfully loaded the website through CloudFront using HTTPS.

Finally, I tested the GitHub Actions deployment pipeline by changing the website content, committing the change, and confirming that the live website updated automatically.

These tests confirmed that the static hosting, CDN, HTTPS, custom domain and automated deployment pipeline were all functioning successfully.

