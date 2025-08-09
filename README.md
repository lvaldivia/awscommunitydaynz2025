
# AWS MediaConvert, MediaTailor, and MediaLive Setup Guide

This guide explains how to set up **AWS MediaConvert**, **AWS MediaTailor**, and **AWS MediaLive** with proper IAM permissions and prerequisites.

---

## 📋 Prerequisites

Before running any MediaConvert, MediaTailor, or MediaLive jobs, you must:

1. **Have an AWS account**
   - Sign up at [https://aws.amazon.com](https://aws.amazon.com)

2. **Create an IAM User with the following permissions:**
   - `AmazonS3FullAccess`
   - `AWSElementalMediaConvertFullAccess`
   - `AWSElementalMediaTailorFullAccess`
   - `AWSElementalMediaLiveFullAccess`

3. **Configure AWS CLI** (optional if you only use the console)
   ```bash
   aws configure --profile myprofile
   ```

4. **Create an S3 Bucket**
   - This bucket will store both your input and output files.
   - Example:
     ```
     s3://my-media-bucket/
     ```

---

## 🛠 Create IAM Role for MediaConvert

MediaConvert needs a **role** it can assume to access your S3 bucket.

**Trust Policy:**
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

Attach the AWS managed policy:
```
AmazonS3FullAccess
```

---

## 📡 Find Your MediaConvert Endpoint

From AWS CLI:
```bash
aws mediaconvert describe-endpoints --region us-east-1 --profile myprofile
```

From AWS Console:
1. Open **AWS MediaConvert**.
2. Click **Endpoints** in the left menu.
3. Copy your account-specific endpoint.

---

## ▶️ Running a Job (CLI Example)
```bash
aws mediaconvert create-job   --region us-east-1   --profile myprofile   --endpoint-url https://abcd1234.mediaconvert.us-east-1.amazonaws.com   --role arn:aws:iam::<ACCOUNT_ID>:role/MediaConvert_Default_Role   --settings file://job.json
```

Example `job.json`:
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
          "Destination": "s3://my-media-bucket/output/playlist"
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
      "FileInput": "s3://my-media-bucket/input/video.mp4",
      "AudioSelectors": {
        "Audio Selector 1": { "DefaultSelection": "DEFAULT" }
      }
    }
  ]
}
```

---

## 🎯 AWS MediaTailor (Console Setup)

### Create a VOD Source
1. Open **AWS MediaTailor** in the console.
2. Go to **VOD Sources** → **Create VOD Source**.
3. Name your source (e.g., `MyVODSource`).
4. Select **Source Location** and choose your S3 bucket containing the HLS playlist.
5. Click **Create**.

### Create a Basic Channel
1. Go to **Channels** → **Create Channel**.
2. Choose **Basic** type.
3. Set **Default Slate** to your preferred placeholder video.
4. Link the VOD source to the channel.
5. Click **Create**.

### Public Access Policy
1. In **Source Locations**, select your source location.
2. Add a **public policy** to allow access from any viewer.
3. Save changes.

---

## 📡 AWS MediaLive (Console Setup)

### Create IAM Role `MediaLiveAccessRole`
Trust Policy:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "medialive.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

Permissions Policy:
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket",
                "s3:PutObject",
                "s3:GetObject",
                "s3:DeleteObject"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "mediastore:ListContainers",
                "mediastore:PutObject",
                "mediastore:GetObject",
                "mediastore:DeleteObject",
                "mediastore:DescribeObject"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "logs:CreateLogGroup",
                "logs:CreateLogStream",
                "logs:PutLogEvents",
                "logs:DescribeLogStreams",
                "logs:DescribeLogGroups"
            ],
            "Resource": "arn:aws:logs:*:*:*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "mediaconnect:ManagedDescribeFlow",
                "mediaconnect:ManagedAddOutput",
                "mediaconnect:ManagedRemoveOutput"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "ec2:describeSubnets",
                "ec2:describeNetworkInterfaces",
                "ec2:createNetworkInterface",
                "ec2:createNetworkInterfacePermission",
                "ec2:deleteNetworkInterface",
                "ec2:deleteNetworkInterfacePermission",
                "ec2:describeSecurityGroups",
                "ec2:describeAddresses",
                "ec2:associateAddress"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "mediapackage:DescribeChannel"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "mediapackagev2:PutObject",
                "mediapackagev2:GetChannel"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "kms:GenerateDataKey"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "secretsmanager:GetSecretValue"
            ],
            "Resource": "*"
        }
    ]
}
```
Also attach `AmazonSSMReadOnlyAccess` managed policy.

### Create an Input (S3-based)
1. Open **AWS MediaLive** in the console.
2. Go to **Inputs** → **Create Input**.
3. Choose **MP4 or TS File** as input type.
4. Select **S3** as the source.
5. Provide the S3 bucket path to your media.
6. Choose `MediaLiveAccessRole` as the IAM role.
7. Save.

### Create a Channel using Custom Template (S3)
1. Go to **Channels** → **Create Channel**.
2. Select **Custom** template.
3. Choose **S3** input created earlier.
4. Set output groups (e.g., HLS output to S3).
5. Assign `MediaLiveAccessRole` as the channel role.
6. Review and create.

---

## 📚 AWS Documentation
- [AWS MediaConvert](https://docs.aws.amazon.com/mediaconvert/latest/ug/what-is.html)
- [AWS MediaTailor](https://docs.aws.amazon.com/mediatailor/latest/ug/what-is.html)
- [AWS MediaLive](https://docs.aws.amazon.com/medialive/latest/ug/what-is.html)
