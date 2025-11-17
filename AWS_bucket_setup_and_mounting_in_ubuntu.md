# AWS bucket setup and mounting in Ubuntu server, and setting up permissions

This guide describes how to:
- Create an IAM user and generate access keys
- Create an S3 bucket
- Create IAM policies, groups, roles and attach permissions
- Add a CORS policy to the bucket
- Install and configure AWS CLI on an Ubuntu server
- (Optional) mount S3 using tools such as s3fs or goofys (instructions included)

> Replace all placeholders (bucket names, ARNs, usernames, IPs, domains, etc.) with your actual values.

---

## 1) Create an IAM user and generate Access Key / Secret Key

1. Open the AWS Console and go to the IAM service.
2. Click "Users" in the left sidebar, then "Create user".
3. Provide a username, select programmatic access if you want access keys.
4. Set permissions (attach policies or add user to a group).
5. After creating, go to the "Security credentials" / "Access keys" section and create an access key.
6. Download the CSV or copy the Access Key ID and Secret Access Key and store them securely.

---

## 2) Create an S3 bucket

1. Login to AWS Management Console and go to S3.
2. Click "Create bucket".
3. Bucket configuration suggestions:
   - Bucket type: General purpose
   - Bucket name: provide a globally unique name
   - Object Ownership: ACLs disabled (recommended)
   - Bucket Versioning: Disable (enable if you require versioning)
   - Leave other options default unless you need encryption, logging, or special settings
4. Create the bucket.

---

## 3) Create a group (optional)

1. In IAM, go to "User groups".
2. Click "Create group" and provide a name.
3. Attach policies to the group as needed (or add users later).

---

## 4) Create IAM policy for S3 access (example)

In IAM → Policies → Create policy → JSON. Adjust the bucket ARN to match your bucket.

Example policy (allows Get/Put on objects, ListBucket on bucket, and STS basic calls):
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowS3Access",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject"
      ],
      "Resource": "arn:aws:s3:::tnd-bucket-styra/*"
    },
    {
      "Sid": "AllowListBucket",
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::tnd-bucket-styra"
    },
    {
      "Sid": "AllowSTS",
      "Effect": "Allow",
      "Action": [
        "sts:AssumeRole",
        "sts:GetCallerIdentity"
      ],
      "Resource": "*"
    }
  ]
}
```

- Provide a name for the policy and create it.

---

## 5) Create an inline permission policy for a specific user (example)

IAM → Users → Select the user → Permissions → Add inline policy → JSON. Update the bucket ARN accordingly.

Example inline policy:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "VisualEditor0",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": "arn:aws:s3:::tnd-bucket-styra/"
    },
    {
      "Sid": "VisualEditor1",
      "Effect": "Allow",
      "Action": [
        "sts:AssumeRole",
        "sts:GetCallerIdentity"
      ],
      "Resource": "*"
    }
  ]
}
```

- Click Next, give the policy a name, and attach it to the user.
- You can also add additional policies (e.g., AdministratorAccess) if required.

---

## 6) Add user to a group

1. Go to the user in IAM.
2. Under "Groups", click "Add user to groups".
3. Select the previously created group and add the user.

---

## 7) Create a Role and attach permission to the role

1. IAM → Roles → Create role.
2. Trusted entity type: choose AWS service (or other needed trust type).
3. For use case you can select S3 or other depending on your scenario.
4. Attach required permissions — e.g., the S3 policy you created or AdministratorAccess (careful with admin).
5. Provide a role name.
6. Edit the role trust policy if you need to allow a specific principal (example below).

Example trust policy snippet (replace ARN and principal as needed):
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "s3.amazonaws.com",
        "AWS": "arn:aws:iam::247789200269:user/karthik"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

7. Attach the previously created policy to the role:
   - IAM → Roles → select role → Add permissions → Attach policies → choose the policy you created.

---

## 8) Create a CORS policy for the bucket

Go to S3 → select bucket → Permissions → CORS configuration. Update AllowedOrigins to your domain.

Example CORS configuration:
```json
[
  {
    "AllowedHeaders": [
      "*"
    ],
    "AllowedMethods": [
      "GET",
      "PUT",
      "HEAD",
      "DELETE"
    ],
    "AllowedOrigins": [
      "https://cmss.styra.in"
    ],
    "ExposeHeaders": [
      "Access-Control-Allow-Origin",
      "ETag"
    ],
    "MaxAgeSeconds": 3000
  }
]
```

- Replace `"https://cmss.styra.in"` with the appropriate origin(s) for your application.

---

## 9) Installing and configuring AWS CLI on Ubuntu

Install prerequisites, download and install AWS CLI v2:
```bash
sudo apt update
sudo apt install -y curl unzip

curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

Verify installation:
```bash
aws --version
```

Configure AWS CLI (this will prompt for your Access Key ID, Secret Access Key, region, and output format):
```bash
aws configure
```

Provide:
- AWS Access Key ID
- AWS Secret Access Key
- Default region name (e.g., us-east-1)
- Default output format (e.g., json)

---

## 10) Mounting S3 on Ubuntu (optional)

There are multiple tools to mount S3 as a filesystem. Two popular options are `s3fs` and `goofys`. Choose one depending on your needs (POSIX behavior vs performance).

### a) Using s3fs (POSIX-like behavior, can be slower)
Install dependencies and s3fs:
```bash
sudo apt update
sudo apt install -y build-essential libfuse-dev libcurl4-openssl-dev libxml2-dev pkg-config
# Install s3fs (on Ubuntu you may find a package or build from source; example using apt if available)
sudo apt install -y s3fs
```

Create a credentials file:
```bash
echo "ACCESS_KEY_ID:SECRET_ACCESS_KEY" > ~/.passwd-s3fs
chmod 600 ~/.passwd-s3fs
```

Mount bucket:
```bash
sudo mkdir -p /mnt/my-s3-bucket
sudo s3fs tnd-bucket-styra /mnt/my-s3-bucket -o passwd_file=~/.passwd-s3fs -o url=https://s3.amazonaws.com -o use_path_request_style
```

To mount at boot, add a line in `/etc/fstab`:
```
s3fs#tnd-bucket-styra /mnt/my-s3-bucket fuse _netdev,passwd_file=/home/<user>/.passwd-s3fs,allow_other 0 0
```
Replace `<user>`, bucket name and options to fit your environment.

### b) Using goofys (faster, optimized for object stores, less POSIX compliance)
Install goofys:
```bash
# download latest release (example)
curl -Lo goofys https://github.com/kahing/goofys/releases/latest/download/goofys
chmod +x goofys
sudo mv goofys /usr/local/bin/
```

Mount bucket:
```bash
sudo mkdir -p /mnt/my-s3-bucket
sudo goofys tnd-bucket-styra /mnt/my-s3-bucket
```

Note: goofys does not require credentials file if environment variables or AWS CLI credentials are configured.

---

## 11) Example: Test S3 access via AWS CLI

List bucket contents:
```bash
aws s3 ls s3://tnd-bucket-styra
```

Upload a file:
```bash
aws s3 cp /path/to/local/file.txt s3://tnd-bucket-styra/path/in/bucket/
```

Download a file:
```bash
aws s3 cp s3://tnd-bucket-styra/path/in/bucket/file.txt /path/to/local/
```

---

## 12) Best practices and notes

- Do NOT hardcode credentials in scripts. Prefer IAM roles (for EC2) or environment variables managed by secure systems.
- Use least-privilege policies. Grant only the permissions necessary (GetObject, PutObject, ListBucket, etc.).
- When mounting S3 to a filesystem, be aware of the limitations: S3 is an object store and is not fully POSIX-compliant.
- Consider lifecycle rules, encryption (SSE), and versioning (if needed).
- Backup policies and monitor audit logs (CloudTrail) for security.
- For production workloads, consider using a dedicated service or gateway (e.g., AWS Transfer or S3 File Gateway) if you need more robust POSIX semantics.

---

## 13) Troubleshooting

- Permission denied errors: verify IAM policy ARNs match the bucket and object ARNs (object ARNs include "/*").
- CORS issues: ensure AllowedOrigins matches the exact protocol + domain you are calling from.
- If mounting fails, verify network connectivity and credentials, and try accessing S3 via `aws s3 ls` to confirm CLI access.
