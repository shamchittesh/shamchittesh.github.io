## Deploying a NextJS App App on AWS S3 🪣

You don't need any fancy webserver configuration to host your static NextJS App, you only need a bucket, preferably green :D 

---

### 1. Building a NextJS App
Creating a NextJS App is fairly easy, you will need to install the latest [here](https://nodejs.org/en/)

##### a. To create a new Next.js project, run in your terminal:
>npx create-next-app@latest

##### b. In your Next.js app, ensure you use static export:
in your `next.config.js` file add this in the NextConfig block
>  output: "export",

##### c. Building static nextjs app:
Running the following command, will generate a static website in a ./out directory
> npm run build


### AWS S3 and Cloudfront configuration - Terraform

##### Terraform setup
You can find more detail about setting up terraform for aws [here](https://developer.hashicorp.com/terraform/tutorials/aws-get-started)

##### Setting up provider.tf

```
provider "aws" {
  region = "us-east-1" # Update with your preferred region
}
```

##### Configuring s3.tf
```
/*
Creates an S3 bucket to store the static files.
acl: Sets permissions for the bucket. public-read allows public access to the files.
website: Configures the bucket to act as a static website.:
•	index_document: The default page (e.g., index.html).
•	error_document: A fallback page for 404 errors.
*/

resource "aws_s3_bucket" "nextjs_bucket" {
  bucket = "my-nextjs-app-bucket" # set a global unique name
  acl    = "public-read"

  website {
    index_document = "index.html"
    error_document = "404.html"
  }

  tags = {
    Name        = "NextJSAppBucket"
    Environment = "Production"
  }
}

/* 
Adds a policy to the bucket to allow public access to the files.
	•	Effect: Allow: Grants permission to perform the specified actions.
	•	Principal: *: anyone can access the files.
	•	Action: s3:GetObject: Allows reading objects in the bucket.
	•	Resource: Targets all files in the bucket.
*/

resource "aws_s3_bucket_policy" "nextjs_bucket_policy" {
  bucket = aws_s3_bucket.nextjs_bucket.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect    = "Allow"
        Principal = "*"
        Action    = "s3:GetObject"
        Resource  = "${aws_s3_bucket.nextjs_bucket.arn}/*"
      }
    ]
  })
}
```
##### Configuring Cloudfront Distribution

```
/*
Sets up a CloudFront distribution to serve the statuc app globally using AWS’s CDN.
•	origin: Connects the CloudFront distribution to the S3 bucket.
•	domain_name: Specifies the bucket’s regional domain.
•	origin_access_identity: Secures access to S3, preventing direct access from the public.
•	default_root_object: Sets index.html as the default page for the root URL.
•	default_cache_behavior: Defines how files are served:
•	viewer_protocol_policy: Redirects HTTP traffic to HTTPS.
•	allowed_methods: Restricts methods to GET and HEAD (safe for static content).
•	cookies: Specifies no cookies are forwarded.
•	viewer_certificate: Uses the default CloudFront SSL certificate for HTTPS.
*/

resource "aws_cloudfront_distribution" "nextjs_distribution" {
  origin {
    domain_name = aws_s3_bucket.nextjs_bucket.bucket_regional_domain_name
    origin_id   = "S3-nextjs-origin"

    s3_origin_config {
      origin_access_identity = aws_cloudfront_origin_access_identity.nextjs_identity.cloudfront_access_identity_path
    }
  }

  enabled             = true
  is_ipv6_enabled     = true
  default_root_object = "index.html"

  default_cache_behavior {
    target_origin_id       = "S3-nextjs-origin"
    viewer_protocol_policy = "redirect-to-https"
    allowed_methods        = ["GET", "HEAD"]
    cached_methods         = ["GET", "HEAD"]

    forwarded_values {
      query_string = false

      cookies {
        forward = "none"
      }
    }
  }

  viewer_certificate {
    cloudfront_default_certificate = true
  }

  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }

  tags = {
    Name = "NextJSCloudFrontDistribution"
  }
}

/*
Creates an Origin Access Identity (OAI) to secure the S3 bucket.
CloudFront uses this identity to access the bucket, ensuring files are served only via CloudFront.
*/

resource "aws_cloudfront_origin_access_identity" "nextjs_identity" {
  comment = "OAI for Next.js S3 Bucket"
}```
##### outputs.tf
```
/* 
Output the CloudFront distribution’s URL after Terraform deploys the resources to test app 
*/
output "cloudfront_url" {
  value = aws_cloudfront_distribution.nextjs_distribution.domain_name
}
```

###### Run `terraform init` `terraform plan` & `terraform apply` to initialize the terraform workload, plan and see the changes made to the aws infrastructure and apply those changes to your infra.

### Additional work: hook it up with your domain



