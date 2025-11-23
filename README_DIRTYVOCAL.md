# DirtyVocal Image Optimization - Quick Start

This repository has been configured for deploying image optimization specifically for the **DirtyVocal (AudioStoryV2)** application.

## 🎯 What This Does

Deploys a serverless image optimization service at `images.dirtyvocal.com` that:
- ✅ Automatically converts images to WebP/AVIF (60-80% smaller files)
- ✅ Resizes images on-demand based on URL parameters
- ✅ Caches transformed images globally via CloudFront CDN
- ✅ Keeps your S3 buckets completely private (no public access)
- ✅ Serves optimized images with `?format=auto&width=500&quality=85`

## 📋 Prerequisites

- [ ] AWS Account with permissions (S3, CloudFront, Lambda, IAM, ACM)
- [ ] AWS CLI installed and configured
- [ ] CDK CLI: `npm install -g aws-cdk`
- [ ] Existing S3 bucket: `dirtyvocal-assets` (in us-west-2)
- [ ] ACM Certificate for `images.dirtyvocal.com` (in us-east-1 region)

## 🚀 Quick Deploy

### 1. Install Dependencies
```bash
cd /Users/llmtest/Projects/image-optimization
npm install
```

### 2. Bootstrap CDK (First time only)
```bash
cdk bootstrap
```

### 3. Build Project
```bash
npm run build
```

### 4. Deploy with Custom Domain
```bash
cdk deploy \
  -c CLOUDFRONT_CUSTOM_DOMAIN=images.dirtyvocal.com \
  -c CLOUDFRONT_CERTIFICATE_ARN=arn:aws:acm:us-east-1:YOUR_ACCOUNT:certificate/YOUR_CERT_ID
```

Replace `YOUR_ACCOUNT` and `YOUR_CERT_ID` with your actual values.

### 5. Configure DNS

After deployment, add a CNAME record in Cloudflare:

- **Type**: CNAME
- **Name**: `images`
- **Value**: `{CloudFront-Domain}` (from deployment output)
- **Proxy**: DNS Only (not proxied)

### 6. Test

```bash
# Test an existing image
curl -I https://images.dirtyvocal.com/{userId}/images/your-image.jpg?format=auto&width=500

# Should return headers including:
# x-aws-image-optimization: v1.0
```

## 📚 Documentation

### For Deployment & AWS Setup:
👉 **[DIRTYVOCAL_DEPLOYMENT.md](./DIRTYVOCAL_DEPLOYMENT.md)** - Complete deployment guide, DNS setup, monitoring, troubleshooting

### For Application Integration:
👉 **[AUDIOSTORY_INTEGRATION.md](./AUDIOSTORY_INTEGRATION.md)** - How to update your AudioStoryV2 code to use optimized images

## 🔧 Configuration

Default settings (in [cdk.json](./cdk.json)):

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

Override at deployment with `-c KEY=VALUE`.

## 🖼️ URL Format

### Original (current):
```
https://media.dirtyvocal.com/{userId}/images/profile.jpg
```

### Optimized (new):
```
https://images.dirtyvocal.com/{userId}/images/profile.jpg?format=auto&width=500&quality=85
```

### Supported Parameters:
- `format`: `auto`, `webp`, `avif`, `jpeg`, `png`
- `width`: `1-9999` (pixels)
- `height`: `1-9999` (pixels)
- `quality`: `1-100` (for lossy formats)

## 🏗️ Architecture

```
User Request
    ↓
CloudFront CDN (images.dirtyvocal.com)
    ↓
┌─────────────────────┐
│ Origin Group        │
├─────────────────────┤
│ 1. S3 Transformed   │ ← Cache Hit (fast)
│    (Primary)        │
│                     │
│ 2. Lambda + Sharp   │ ← Cache Miss (slower, first time)
│    (Fallback)       │
│      ↓              │
│   S3 Original       │
│   (dirtyvocal-      │
│    assets)          │
└─────────────────────┘
```

## 💰 Cost Estimate

~$25-35/month for 1M requests (assuming 50% cache hit rate):
- CloudFront: ~$8-12
- Lambda: ~$15-20
- S3: ~$2

Higher cache hit rate = lower costs.

## 🔍 Monitoring

### CloudWatch Logs:
```
/aws/lambda/ImageOptimizationStack-imageoptimization{hash}
```

### CloudFront Metrics:
- Cache Hit Rate (target: >90%)
- Origin Response Time
- 4xx/5xx Errors

### Cost Explorer:
Filter by: CloudFront, Lambda, S3

## 🧹 Cleanup

To remove all resources:
```bash
cdk destroy
```

**Note**: This does NOT delete your `dirtyvocal-assets` bucket.

## 🔐 Security

- ✅ Both S3 buckets are completely private
- ✅ CloudFront uses Origin Access Control (OAC) with SigV4
- ✅ SSL/TLS enforced
- ✅ CORS enabled for cross-origin requests
- ✅ IAM least-privilege permissions

## 📞 Support

- **Deployment Issues**: See [DIRTYVOCAL_DEPLOYMENT.md](./DIRTYVOCAL_DEPLOYMENT.md)
- **Integration Issues**: See [AUDIOSTORY_INTEGRATION.md](./AUDIOSTORY_INTEGRATION.md)
- **AWS Issues**: Check CloudWatch Logs

## 🎯 Next Steps

1. ✅ Deploy this stack (you're here!)
2. 📱 Update AudioStoryV2 app (see [AUDIOSTORY_INTEGRATION.md](./AUDIOSTORY_INTEGRATION.md))
3. 📊 Monitor performance and costs
4. 🎉 Enjoy faster image loading!

---

**Original AWS Sample**: https://github.com/aws-samples/image-optimization
**Configured for**: DirtyVocal (AudioStoryV2)
**Domain**: images.dirtyvocal.com
**Source Bucket**: dirtyvocal-assets (private)
