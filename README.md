# AWS NZ Instructions

# AWS MediaConvert Setup Guide

This guide explains how to set up **AWS MediaConvert** with proper IAM permissions and prerequisites.

---

## 📋 Prerequisites

Before running any MediaConvert jobs, you must:

1. **Have an AWS account**
   - Sign up at [https://aws.amazon.com](https://aws.amazon.com)

2. **Create an IAM User with the following permissions:**
   - `AmazonS3FullAccess`
   - `AWSElementalMediaConvertFullAccess`
   - `AWSElementalMediaTailorFullAccess`

3. **Configure AWS CLI**
   ```bash
   aws configure --profile myprofile
   ```
   Provide your **Access Key ID**, **Secret Access Key**, default region (e.g., `us-east-1`), and output format (`json` recommended).

4. **Create an S3 Bucket**
   - This bucket will store both your input and output files.
   - Example:
     ```
     s3://my-mediaconvert-bucket/
     ```

---

## 🛠 Create IAM Role for MediaConvert

MediaConvert needs a **role** it can assume to access your S3 bucket.

### 1. **Trust Policy**
When creating the role, set the **Trusted Entities** to:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "MediaConvertAssumeRole",
      "Effect": "Allow",
      "Principal": {
        "Service": "mediaconvert.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

This allows MediaConvert to **assume the role**.

---

### 2. **Attach Permissions Policy**
Attach the AWS managed policy:

```
AmazonS3FullAccess
```

This grants MediaConvert permission to **read** and **write** to S3.

---

## 📡 Find Your MediaConvert Endpoint

Before creating a job, you must know your **account-specific endpoint**.

```bash
aws mediaconvert describe-endpoints \
  --region us-east-1 \
  --profile myprofile
```

Output example:
```json
{
  "Endpoints": [
    {
      "Url": "https://abcd1234.mediaconvert.us-east-1.amazonaws.com"
    }
  ]
}
```

Use this URL in your MediaConvert job commands.

---

## ▶️ Running a Job

Example MediaConvert job command:

```bash
aws mediaconvert create-job \
  --region us-east-1 \
  --profile myprofile \
  --endpoint-url https://abcd1234.mediaconvert.us-east-1.amazonaws.com \
  --role arn:aws:iam::<ACCOUNT_ID>:role/MediaConvert_Default_Role \
  --settings file://job.json
```

Where:
- `file://job.json` is your job configuration file.
- `--role` is the ARN of the IAM role you created for MediaConvert.

---

## 📂 Example `job.json`
```json
{
  "TimecodeConfig": { "Source": "ZEROBASED" },
  "OutputGroups": [
    {
      "Name": "Apple HLS",
      "OutputGroupSettings": {
        "Type": "HLS_GROUP_SETTINGS",
        "HlsGroupSettings": {
          "SegmentLength": 2,
          "Destination": "s3://my-mediaconvert-bucket/output/playlist"
        }
      },
      "Outputs": [
        {
          "ContainerSettings": { "Container": "M3U8" },
          "VideoDescription": {
            "CodecSettings": {
              "Codec": "H_264",
              "H264Settings": {
                "MaxBitrate": 5000000
              }
            }
          },
          "AudioDescriptions": [
            {
              "AudioSourceName": "Audio Selector 1",
              "CodecSettings": { "Codec": "AAC" }
            }
          ]
        }
      ]
    }
  ],
  "Inputs": [
    {
      "FileInput": "s3://my-mediaconvert-bucket/input/video.mp4",
      "AudioSelectors": {
        "Audio Selector 1": { "DefaultSelection": "DEFAULT" }
      }
    }
  ]
}
```

---

## ✅ Summary

- Create an IAM user with **full access** to S3, MediaConvert, and MediaTailor.
- Create an S3 bucket for inputs and outputs.
- Create an IAM role with a trust policy for **mediaconvert.amazonaws.com** and `AmazonS3FullAccess` permissions.
- Get your MediaConvert endpoint with `aws mediaconvert describe-endpoints`.
- Run jobs with `aws mediaconvert create-job` using your endpoint and role.

---

## 📚 AWS Documentation
- [AWS MediaConvert Documentation](https://docs.aws.amazon.com/mediaconvert/latest/ug/what-is.html)
- [AWS CLI MediaConvert Commands](https://docs.aws.amazon.com/cli/latest/reference/mediaconvert/index.html)
