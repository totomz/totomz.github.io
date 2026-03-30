---
title: Hosting Static Website in AWS                    
date: 2026-03-30T07:58:00+01:00
tags: 
  - aws
  - terraform
  - opentofu
  

img_folder: '/images/2026-03-30-static_hosting'  

author: totomz


---

> Yes, another blog post about hosting a static website in AWS S3 + CloudFront.
> This simple post is giving me the opportunity to revamp this blog, update Hugo, and check
> that the pipeline actually deploy it to a live web page

# What? Really? I can easily host a website on AWS S3?
Maybe you don't know yet, but [since February 2011, it is possible to host a static website on AWS S3 + CloudFront](https://aws.amazon.com/about-aws/whats-new/2011/02/17/Amazon-S3-Website-Features/) almost for free.

This post shows a simple way to generate all terraform/OpenTofu resources to easily create all the AWS components to quickly have a static website online

# AWS Resources
First of all, these resources work only with [OpenTofu](https://opentofu.org/), the community-supported Terraform fork, because
of the APEX DNS record. If a canonical website DNS record could be a third-level domain like `www.croccocode.com`,
an APEX domain or naked domain is a second level name like `corccocode.com`.

Not all websites require an apex domain, usually only landing pages and websites. Here we generate the DNS records dinamycally
and the apex domain are generated only if a variable is set - and this conditionality is expressed with the `enabled` meta-argument -> https://opentofu.org/docs/language/meta-arguments/enabled/

{{< post-img "shut-up-show-me-the-code.png" "alt text" >}}

Here we are

**main.tf** 
```terraform
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "= 6.38.0"
    }
  }
  backend "s3" {
    bucket = "<your bucket>"
    key = "factory/www/v1"
  }
}

locals {
  dns_root_zone = "croccocode.com"      # <-- root zone name
  site-url-dash = "www-croccocode-com"  # <-- website domain
  apex = "croccocode.com"               # <-- apex - leave it empty to skip the APEX dns record
}

provider "aws" {
  region = "eu-west-1"
}

data "aws_caller_identity" "current" {}

````


**aws-s3.tf**
```terraform
resource "aws_s3_bucket" "website" {
  bucket = local.site-url-dash
}

resource "aws_s3_bucket_public_access_block" "website" {
  bucket = aws_s3_bucket.website.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

```

**aws-cloudfront.tf**
```terraform
locals {
  acm_asterisco = "<certificate arn>"
}

resource "aws_cloudfront_origin_access_control" "website" {
  name                              = local.site-url-dash
  origin_access_control_origin_type = "s3"
  signing_behavior                  = "always"
  signing_protocol                  = "sigv4"
}

# CloudFront Distribution
resource "aws_cloudfront_distribution" "website" {
  enabled             = true
  is_ipv6_enabled     = true
  default_root_object = "index.html"

  aliases = compact([
    replace(local.site-url-dash, "-", "."),
    local.apex != "" ? local.apex : null,
  ])
  
  viewer_certificate {
    acm_certificate_arn = local.acm_asterisco
    minimum_protocol_version = "TLSv1.3_2025"
    ssl_support_method = "sni-only"
  }
  
  origin {
    domain_name              = aws_s3_bucket.website.bucket_regional_domain_name
    origin_id                = local.site-url-dash
    origin_access_control_id = aws_cloudfront_origin_access_control.website.id
  }
  
  default_cache_behavior {
    allowed_methods        = ["GET", "HEAD", "OPTIONS"]
    cached_methods         = ["GET", "HEAD", "OPTIONS"]
    target_origin_id       = local.site-url-dash
    viewer_protocol_policy = "redirect-to-https"
    compress               = true

    cache_policy_id          = "658327ea-f89d-4fab-a63d-7e88639e58f6"  # CachingOptimized
    origin_request_policy_id = "88a5eaf4-2fd4-4709-b370-b4c650ea3fcf"  # CORS-S3Origin
  }
  
  

  # Per React Router - redirect 404/403 a index.html
  custom_error_response {
    error_code            = 403
    response_code         = 200
    response_page_path    = "/index.html"
    error_caching_min_ttl = 0
  }

  custom_error_response {
    error_code            = 404
    response_code         = 200
    response_page_path    = "/index.html"
    error_caching_min_ttl = 0
  }

  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }
}

# Policy S3 per CloudFront
resource "aws_s3_bucket_policy" "website" {
  bucket = aws_s3_bucket.website.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Service = "cloudfront.amazonaws.com"
        }
        Action   = "s3:GetObject"
        Resource = "${aws_s3_bucket.website.arn}/*"
        Condition = {
          StringEquals = {
            "AWS:SourceArn" = aws_cloudfront_distribution.website.arn
          }
        }
      }
    ]
  })
}

output "cloudfront_url" {
  value = <<EOF
#### Cloudfront URL for CNAM Record ####
https://${aws_cloudfront_distribution.website.domain_name}
EOF
}

output "bucket_name" {
  value = aws_s3_bucket.website.id
}

output "cloudfront_id" {
  value = aws_cloudfront_distribution.website.id
}
```

And finally, the DNS records!
```terraform
data "aws_route53_zone" "root" {
  name         = local.dns_root_zone
  private_zone = false
}

resource "aws_route53_record" "apex_a" {
  zone_id = data.aws_route53_zone.root.zone_id
  name    = local.apex
  type    = "A"

  alias {
    name                   = aws_cloudfront_distribution.website.domain_name
    zone_id                = aws_cloudfront_distribution.website.hosted_zone_id
    evaluate_target_health = false
  }
  
  # OpenTofu Only!
  lifecycle {
    enabled = local.apex != ""
  }
}

resource "aws_route53_record" "apex_aaaa" {
  zone_id = data.aws_route53_zone.root.zone_id
  name    = local.apex
  type    = "AAAA"

  alias {
    name                   = aws_cloudfront_distribution.website.domain_name
    zone_id                = aws_cloudfront_distribution.website.hosted_zone_id
    evaluate_target_health = false
  }
  
  # OpenTofu Only!
  lifecycle {
    enabled = local.apex != ""
  }
}

resource "aws_route53_record" "www_a" {
  zone_id = data.aws_route53_zone.root.zone_id
  name    = replace(local.site-url-dash, "-", ".")
  type    = "A"

  alias {
    name                   = aws_cloudfront_distribution.website.domain_name
    zone_id                = aws_cloudfront_distribution.website.hosted_zone_id
    evaluate_target_health = false
  }
}

resource "aws_route53_record" "www_aaaa" {
  zone_id = data.aws_route53_zone.root.zone_id
  name    = replace(local.site-url-dash, "-", ".")
  type    = "AAAA"

  alias {
    name                   = aws_cloudfront_distribution.website.domain_name
    zone_id                = aws_cloudfront_distribution.website.hosted_zone_id
    evaluate_target_health = false
  }
}

```