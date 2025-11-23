# DirtyVocal Image Optimization Deployment Guide

This guide covers deploying the AWS Serverless Image Optimization solution specifically for the DirtyVocal (AudioStoryV2) application.

## Overview

This deployment will create:
- **CloudFront Distribution** at `images.dirtyvocal.com` for optimized image delivery
- **Lambda Function** for on-demand image processing (WebP/AVIF conversion, resizing, compression)
- **S3 Bucket** for storing transformed/cached images (private)
- **Origin Access Control (OAC)** for secure, private S3 bucket access

## Architecture

```
User Request (images.dirtyvocal.com)
    ↓
CloudFront CDN (Global Edge Locations)
    ↓
Origin Group:
    1. S3 Transformed Images (Primary - Cache Hit)
    2. Lambda + Sharp (Fallback - Process & Cache)
        ↓
    S3 Original Images (dirtyvocal-assets - PRIVATE)
```

### Security Features
- ✅ Both S3 buckets remain **completely private** (no public access)
- ✅ CloudFront uses **Origin Access Control (OAC)** with SigV4 signing
- ✅ Lambda has IAM permissions to read/write from private buckets
- ✅ All traffic uses HTTPS/TLS
- ✅ CORS enabled for cross-origin image serving

## Prerequisites

### 1. AWS Account Setup
- AWS account with appropriate permissions (S3, CloudFront, Lambda, IAM, ACM, CDK)
- AWS CLI installed and configured
- AWS CDK CLI installed: `npm install -g aws-cdk`

### 2. Existing Resources
- **S3 Bucket**: `dirtyvocal-assets` (in `us-east-1` region)
- **Images Path**: Your original images should be stored in the bucket (e.g., `{userId}/images/{filename}`)

### 3. Domain & SSL Certificate

#### Option A: Request New Certificate (Recommended)
1. Go to AWS Certificate Manager (ACM) **in us-east-1 region** (CloudFront requires certificates in us-east-1)
2. Request a public certificate for `images.dirtyvocal.com`
3. Use DNS validation (add CNAME records to your DNS)
4. Wait for certificate to be issued and validated
5. Copy the Certificate ARN (e.g., `arn:aws:acm:us-east-1:123456789012:certificate/abc-def-123`)

#### Option B: Use Existing Certificate
If you already have a wildcard certificate for `*.dirtyvocal.com`, you can use that instead.

## Deployment Steps

### Step 1: Clone and Install Dependencies

```bash
cd /Users/llmtest/Projects/image-optimization
npm install
```

### Step 2: Bootstrap CDK (First Time Only)

If you haven't used CDK in your AWS account before:

```bash
cdk bootstrap aws://YOUR_ACCOUNT_ID/us-east-1
```

### Step 3: Configure Deployment

The default configuration is already set in [cdk.json](./cdk.json):

```json
{
  "S3_IMAGE_BUCKET_NAME": "dirtyvocal-assets",
  "STORE_TRANSFORMED_IMAGES": true,
  "LAMBDA_MEMORY": 2000,
  "LAMBDA_TIMEOUT": 60,
  "CLOUDFRONT_CORS_ENABLED": true,
  "S3_TRANSFORMED_IMAGE_EXPIRATION_DURATION": "90",
  "MAX_IMAGE_SIZE": 26214400
}
```

**For custom domain**, you'll need to provide these at deployment time:
- `CLOUDFRONT_CUSTOM_DOMAIN`: Your custom domain (e.g., `images.dirtyvocal.com`)
- `CLOUDFRONT_CERTIFICATE_ARN`: ARN of your ACM certificate in us-east-1

### Step 4: Build the Project

```bash
npm run build
```

This will:
- Compile TypeScript to JavaScript
- Install Sharp library for Linux (Lambda runtime compatibility)

### Step 5: Preview Changes (Optional)

```bash
cdk diff \
  -c CLOUDFRONT_CUSTOM_DOMAIN=images.dirtyvocal.com \
  -c CLOUDFRONT_CERTIFICATE_ARN=arn:aws:acm:us-east-1:YOUR_ACCOUNT:certificate/YOUR_CERT_ID
```

### Step 6: Deploy to AWS

#### Without Custom Domain (CloudFront URL only):
```bash
cdk deploy
```

#### With Custom Domain (Recommended):
```bash
cdk deploy \
  -c CLOUDFRONT_CUSTOM_DOMAIN=images.dirtyvocal.com \
  -c CLOUDFRONT_CERTIFICATE_ARN=arn:aws:acm:us-east-1:YOUR_ACCOUNT:certificate/YOUR_CERT_ID
```

**Note**: Replace `YOUR_ACCOUNT` and `YOUR_CERT_ID` with your actual values.

The deployment will take 5-10 minutes. CDK will:
1. Create the transformed images S3 bucket
2. Create Lambda function with Sharp for image processing
3. Create CloudFront distribution with custom domain (if provided)
4. Set up OAC for secure private bucket access
5. Configure IAM roles and permissions

### Step 7: Note the Outputs

After deployment completes, CDK will output:

```
Outputs:
ImageDeliveryDomain = d1234abcd.cloudfront.net
CustomDomain = images.dirtyvocal.com
CloudFrontDistributionId = E1234ABCD5678
OriginalImagesS3Bucket = dirtyvocal-assets
```

**Save these values** - you'll need them for DNS configuration.

## Post-Deployment Configuration

### Step 8: Configure DNS

Add a CNAME record in your DNS provider (Cloudflare):

**Record Type**: CNAME
**Name**: `images` (or `images.dirtyvocal.com`)
**Value**: `d1234abcd.cloudfront.net` (use the value from `ImageDeliveryDomain` output)
**TTL**: 300 (or Auto)
**Proxy Status**: DNS Only (not proxied through Cloudflare)

**Important**: Do NOT proxy through Cloudflare - point directly to CloudFront.

### Step 9: Wait for DNS Propagation

DNS propagation can take 5-60 minutes. Test with:

```bash
dig images.dirtyvocal.com
# or
nslookup images.dirtyvocal.com
```

You should see it pointing to the CloudFront distribution.

### Step 10: Test Image Optimization

Once DNS propagates, test with an existing image in your S3 bucket:

**Original Image** (via current setup):
```
https://media.dirtyvocal.com/{userId}/images/profile-photo.jpg
```

**Optimized Image** (via new CloudFront):
```
https://images.dirtyvocal.com/{userId}/images/profile-photo.jpg
```

**With Transformations**:
```
# Auto format (WebP/AVIF based on browser)
https://images.dirtyvocal.com/{userId}/images/profile-photo.jpg?format=auto&width=500

# Specific format and size
https://images.dirtyvocal.com/{userId}/images/profile-photo.jpg?format=webp&width=300&quality=85

# Resize with height
https://images.dirtyvocal.com/{userId}/images/profile-photo.jpg?format=auto&width=800&height=600
```

**Expected Results**:
- First request: Slower (Lambda processing)
- Subsequent requests: Fast (CloudFront cache)
- Smaller file sizes (WebP/AVIF typically 60-80% smaller)
- Response headers should include: `x-aws-image-optimization: v1.0`

## Image URL Parameters

### Supported Transformations

| Parameter | Values | Description | Example |
|-----------|--------|-------------|---------|
| `format` | `auto`, `jpeg`, `webp`, `avif`, `png` | Output format. `auto` selects best format based on browser support | `?format=auto` |
| `width` | `1-9999` | Target width in pixels (maintains aspect ratio) | `?width=500` |
| `height` | `1-9999` | Target height in pixels (maintains aspect ratio) | `?height=300` |
| `quality` | `1-100` | Quality for lossy formats (default: varies by format) | `?quality=85` |

### Common Use Cases for DirtyVocal

```bash
# User Profile Thumbnails (200x200)
?format=auto&width=200&quality=85

# Song/Album/Playlist Cards (500x500)
?format=auto&width=500&quality=85

# Full Page Headers (1000px wide)
?format=auto&width=1000&quality=90

# Podcast Cover Art (600x600)
?format=auto&width=600&quality=85

# Player Artwork (400x400)
?format=auto&width=400&quality=85
```

## Configuration Reference

### CDK Context Variables

All configurable via `cdk.json` or `-c` flag at deployment:

| Variable | Default | Description |
|----------|---------|-------------|
| `S3_IMAGE_BUCKET_NAME` | `dirtyvocal-assets` | Existing S3 bucket with original images |
| `STORE_TRANSFORMED_IMAGES` | `true` | Cache transformed images in S3 (recommended) |
| `LAMBDA_MEMORY` | `2000` | Lambda memory in MB (1500-3000 recommended) |
| `LAMBDA_TIMEOUT` | `60` | Lambda timeout in seconds |
| `CLOUDFRONT_CORS_ENABLED` | `true` | Enable CORS headers |
| `CLOUDFRONT_CUSTOM_DOMAIN` | - | Custom domain (e.g., `images.dirtyvocal.com`) |
| `CLOUDFRONT_CERTIFICATE_ARN` | - | ACM certificate ARN (must be in us-east-1) |
| `S3_TRANSFORMED_IMAGE_EXPIRATION_DURATION` | `90` | Days to keep transformed images in S3 |
| `MAX_IMAGE_SIZE` | `26214400` | Max image size in bytes (~25MB) |

### Performance Tuning

**Lambda Memory**: Higher memory = faster CPU
- 1500 MB: Budget option (~3-5s for large images)
- 2000 MB: Recommended (default, ~2-3s for large images)
- 3000 MB: Premium performance (~1-2s for large images)

**Cost Trade-off**: Higher memory costs more per invocation, but faster processing = shorter duration = potentially lower total cost.

## Monitoring & Troubleshooting

### CloudWatch Logs

Lambda logs are available in CloudWatch Logs (1-day retention):
```
Log Group: /aws/lambda/ImageOptimizationStack-imageoptimization{hash}
```

### CloudFront Metrics

Monitor in CloudFront console:
- Cache Hit Rate (should be >90% after warmup)
- Origin Response Time
- 4xx/5xx Error Rates

### Common Issues

#### Issue: Images not loading
- **Check DNS**: Ensure CNAME points to CloudFront distribution
- **Check SSL**: Ensure certificate is valid and covers the domain
- **Check S3 Path**: Verify original images exist in `dirtyvocal-assets` bucket

#### Issue: Slow performance
- **Cold Start**: First request is slower (Lambda + processing)
- **Increase Lambda Memory**: Try 2500-3000 MB for faster processing
- **Check Origin Shield Region**: Should be close to your S3 bucket region

#### Issue: 403 Forbidden
- **Verify OAC**: Ensure CloudFront has permission to access Lambda
- **Check S3 Permissions**: Ensure Lambda can read from `dirtyvocal-assets`
- **Check Image Path**: Ensure image exists at the specified S3 key

## Cost Estimation

### Monthly Cost Breakdown (Assuming 1M image requests, 50% cache hit rate)

| Service | Usage | Estimated Cost |
|---------|-------|----------------|
| CloudFront | 1M requests, 100GB transfer | ~$8-12 |
| Lambda | 500K invocations @ 2GB, 2s avg | ~$15-20 |
| S3 Storage | 50GB transformed images | ~$1.15 |
| S3 Requests | 500K GET, 500K PUT | ~$0.50 |
| **Total** | | **~$25-35/month** |

**Cost Optimization Tips**:
- Higher cache hit rate = Lower Lambda costs
- S3 lifecycle policy (90 days) keeps storage costs low
- Origin Shield reduces Lambda invocations

## Updating the Stack

To update configuration or code:

```bash
# Make changes to code or cdk.json
npm run build

# Preview changes
cdk diff

# Deploy changes
cdk deploy
```

## Rollback / Destroy

To remove all resources:

```bash
cdk destroy
```

**Warning**: This will delete:
- CloudFront distribution
- Lambda function
- Transformed images S3 bucket (and all cached images)

**It will NOT delete**:
- Original `dirtyvocal-assets` bucket
- Your domain or DNS records

## Next Steps

After successful deployment, proceed to the [AudioStoryV2 Integration Guide](./AUDIOSTORY_INTEGRATION.md) to update your application code to use the optimized image URLs.

## Support

For issues with:
- **AWS Resources**: Check CloudWatch Logs and CloudFront metrics
- **CDK Deployment**: Review CDK error messages and IAM permissions
- **Image Processing**: Check Lambda function logs for Sharp errors
- **DNS/SSL**: Verify ACM certificate and DNS CNAME records

## Security Best Practices

✅ **Currently Implemented**:
- Private S3 buckets (no public access)
- Origin Access Control (OAC) with SigV4
- SSL/TLS enforcement
- CORS configured
- IAM least-privilege permissions
- Automatic resource cleanup on deletion

⚠️ **Additional Recommendations**:
- Enable CloudFront access logging for audit trails
- Set up CloudWatch alarms for error rates
- Use AWS WAF for DDoS protection (optional, adds cost)
- Implement CloudFront geo-restrictions if needed
