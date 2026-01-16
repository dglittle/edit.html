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

Allow public read access:

```json
{
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
}
```

### 3. Create IAM User for Editing

Create an IAM user with write access to your bucket:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:PutObjectAcl"
            ],
            "Resource": "arn:aws:s3:::yourdomain.com/*"
        }
    ]
}
```

Save the Access Key ID and Secret Access Key.

### 4. Upload the File

```bash
aws s3 cp edit s3://yourdomain.com/edit --content-type "text/html"
```

### 5. Set Up DNS

**Option A: Direct S3 (HTTP only)**

Point your domain to the S3 website endpoint:
```
yourdomain.com → yourdomain.com.s3-website-us-east-1.amazonaws.com
```

**Option B: CloudFront (HTTPS)**

1. Create a CloudFront distribution with your S3 bucket as origin
2. Add your domain as an alternate domain name (CNAME)
3. Request an SSL certificate via ACM
4. Point your domain to the CloudFront distribution

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
   localStorage.setItem('anthropic_api_key', 'sk-ant-...')
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
