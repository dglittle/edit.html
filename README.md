# edit.html

A minimal self-editing website. One HTML file + an S3 bucket = a website that can edit itself.

## What is this?

`edit.html` turns any S3 bucket into a self-hosted, self-editing website. Visit `yourdomain.com/edit#index` and you get a split-pane editor: code on the left, live preview on the right. Press Cmd+S to save directly to S3.

Features:
- Monaco editor (same as VS Code)
- Live preview
- AI assistance via Claude (Cmd+B)
- Code formatting via Prettier (Cmd+P)
- Edits any file in the bucket, including itself

## Quick Start

1. Create an S3 bucket named after your domain (e.g., `example.com`)
2. Upload `edit` to the bucket
3. Configure S3 for static website hosting
4. Point your domain to the bucket (via CloudFront or directly)
5. Set up AWS credentials in your browser's localStorage
6. Visit `yourdomain.com/edit#index` to start editing

## Setup Guide

### 1. Create S3 Bucket

```bash
aws s3 mb s3://yourdomain.com
```

Enable static website hosting:
```bash
aws s3 website s3://yourdomain.com --index-document index
```

### 2. Set Bucket Policy

First, disable S3 Block Public Access:
```bash
aws s3api put-public-access-block --bucket yourdomain.com --public-access-block-configuration "BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false"
```

Then allow public read access:
```bash
aws s3api put-bucket-policy --bucket yourdomain.com --policy '{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PublicReadGetObject",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::yourdomain.com/*"
        }
    ]
}'
```

### 3. Create IAM User for Editing

Create an IAM user:
```bash
aws iam create-user --user-name yourdomain-editor
```

Attach a policy with write and list access:
```bash
aws iam put-user-policy --user-name yourdomain-editor --policy-name s3-access --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:PutObjectAcl",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::yourdomain.com",
                "arn:aws:s3:::yourdomain.com/*"
            ]
        }
    ]
}'
```

Create access keys and save them:
```bash
aws iam create-access-key --user-name yourdomain-editor
```

### 4. Upload the File

```bash
aws s3 cp edit s3://yourdomain.com/edit --content-type "text/html"
```

### 5. Set Up CloudFront (HTTPS)

**Request an SSL certificate** (must be in us-east-1 for CloudFront):
```bash
aws acm request-certificate --domain-name yourdomain.com --validation-method DNS --region us-east-1
```

**Get the DNS validation record**:
```bash
aws acm describe-certificate --certificate-arn YOUR_CERT_ARN --region us-east-1 --query 'Certificate.DomainValidationOptions[0].ResourceRecord'
```

Add the CNAME record to your DNS provider and wait for validation:
```bash
aws acm describe-certificate --certificate-arn YOUR_CERT_ARN --region us-east-1 --query 'Certificate.Status'
```

**Create CloudFront distribution** (after certificate is validated):
```bash
aws cloudfront create-distribution --distribution-config '{
    "CallerReference": "yourdomain-'$(date +%s)'",
    "Origins": {
        "Quantity": 1,
        "Items": [{
            "Id": "S3-yourdomain.com",
            "DomainName": "yourdomain.com.s3-website-YOUR_REGION.amazonaws.com",
            "CustomOriginConfig": {
                "HTTPPort": 80,
                "HTTPSPort": 443,
                "OriginProtocolPolicy": "http-only"
            }
        }]
    },
    "DefaultCacheBehavior": {
        "TargetOriginId": "S3-yourdomain.com",
        "ViewerProtocolPolicy": "redirect-to-https",
        "AllowedMethods": {
            "Quantity": 2,
            "Items": ["GET", "HEAD"],
            "CachedMethods": {"Quantity": 2, "Items": ["GET", "HEAD"]}
        },
        "ForwardedValues": {"QueryString": true, "Cookies": {"Forward": "none"}},
        "MinTTL": 0,
        "DefaultTTL": 86400,
        "MaxTTL": 31536000,
        "Compress": true
    },
    "Aliases": {"Quantity": 1, "Items": ["yourdomain.com"]},
    "ViewerCertificate": {
        "ACMCertificateArn": "YOUR_CERT_ARN",
        "SSLSupportMethod": "sni-only",
        "MinimumProtocolVersion": "TLSv1.2_2021"
    },
    "Enabled": true,
    "Comment": "yourdomain.com",
    "DefaultRootObject": "index"
}'
```

**Update DNS**: Point your domain to the CloudFront distribution domain (e.g., `d1234567890.cloudfront.net`).

For root/apex domains, use an ALIAS record (if your DNS provider supports it):
- **Host**: `@`
- **Points to**: `d1234567890.cloudfront.net`

For subdomains, use a CNAME record.

**Alternative: Direct S3 (HTTP only)**

If you don't need HTTPS, point your domain CNAME to:
```
yourdomain.com.s3-website-YOUR_REGION.amazonaws.com
```

### 6. Configure Browser Credentials

Open your browser's developer console on your site and run:

```javascript
localStorage.aws_id = 'YOUR_ACCESS_KEY_ID'
localStorage.aws_secret = 'YOUR_SECRET_ACCESS_KEY'
localStorage.aws_region = 'us-east-1'  // or your bucket's region
```

### 7. Start Editing

Visit `https://yourdomain.com/edit#index` to create/edit your homepage.

## Usage

| URL | Description |
|-----|-------------|
| `/edit#index` | Edit the homepage |
| `/edit#about` | Edit a page called "about" |
| `/edit#styles.css` | Edit CSS files |
| `/edit#app.js` | Edit JavaScript files |

### Keyboard Shortcuts

| Shortcut | Action |
|----------|--------|
| Cmd+S / Ctrl+S | Save to S3 |
| Cmd+Enter | Save to S3 |
| Cmd+B | Ask Claude AI for help |
| Cmd+Shift+B | Select AI model |
| Cmd+P | Format code with Prettier |

## AI Features

The editor includes Claude AI integration (calls Anthropic API directly from the browser). To use it:

1. Set your API key in the browser console:
   ```javascript
   localStorage.anthropic_api_key = 'sk-ant-...'
   ```
2. Select text (or use entire file)
3. Press Cmd+B
4. Review the diff and accept or cancel

Choose between Claude Sonnet (faster) and Opus (more capable) with Cmd+Shift+B.

## File Structure

```
your-bucket/
├── edit          # The editor (just this one file!)
├── index         # Your homepage (create this)
└── ...           # Any other files you create
```

## Security Notes

- AWS credentials are stored in localStorage (browser-only, not transmitted)
- The bucket policy allows public read access
- Write access is controlled by IAM credentials
- Consider using IAM roles or more restrictive policies for production

## How It Works

1. `edit` loads in a split-pane layout with Monaco editor on the left and a preview iframe on the right
2. When you save, it uses the AWS SDK to PUT the file directly to S3
3. The preview pane reloads to show your changes

The magic is that the editor can edit itself - visit `/edit#edit` to customize it.

## License

MIT
