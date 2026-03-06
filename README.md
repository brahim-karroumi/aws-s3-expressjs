# AWS S3 Bucket — File Upload Guide

A step-by-step guide to implementing file uploads to an AWS S3 bucket using Node.js, Express, and Multer.

---

## Table of Contents

1. [Handling Multipart Form Data with Multer](#1-handling-multipart-form-data-with-multer)
2. [Setting Up Your S3 Bucket](#2-setting-up-your-s3-bucket)
3. [IAM — Creating a Policy and User](#3-iam--creating-a-policy-and-user)
4. [Connecting to S3 from Your App](#4-connecting-to-s3-from-your-app)
5. [Uploading Files to S3](#5-uploading-files-to-s3)
6. [Generating Unique Filenames](#6-generating-unique-filenames)
7. [Generating a Signed URL (with expiry)](#7-generating-a-signed-url-with-expiry)
8. [Public Bucket URL (no expiry)](#8-public-bucket-url-no-expiry)

---

## 1. Handling Multipart Form Data with Multer

Express cannot parse `multipart/form-data` by default. We use [Multer](https://www.npmjs.com/package/multer) to handle it.

**Install:**

```bash
npm install multer
```

**Setup:**

```js
import multer from "multer";

const storage = multer.memoryStorage();
const upload = multer({ storage: storage });
```

**Before (without Multer):**

```js
app.post("/posts", (req, res) => {
    try {
        const { title, description, image } = req.body;

        console.log({ title, description, image });
        res.status(200).json({ message: "Post created successfully" });
    } catch (error) {
        res.status(500).json({ message: "Error creating post", error: error.message });
    }
});
```

**After (with Multer):**

```js
app.post("/posts", upload.single("image"), (req, res) => {
    try {
        const { title, description } = req.body;
        const image = req.file;

        console.log({ title, description, image });
        res.status(200).json({ message: "Post created successfully" });
    } catch (error) {
        res.status(500).json({ message: "Error creating post", error: error.message });
    }
});
```

**The received `req.file` object looks like:**

```js
{
    fieldname: 'image',
    originalname: 'Rugal_cvsnk2.png',
    encoding: '7bit',
    mimetype: 'image/png',
    buffer: <Buffer 89 50 4e 47 ...>,
    size: 139923
}
```

> The most important field is `buffer` — it contains the raw image data: `imageData = req.file.buffer`

---

## 2. Setting Up Your S3 Bucket

1. Go to [https://aws.amazon.com](https://aws.amazon.com) and sign in.
2. Search for **S3** in the navbar under the **Products** tab and click it.
3. Click **Create bucket**.
4. Choose **General purpose**, give your bucket a name, and click **Create bucket**.
5. Navigate to **Properties**, copy the **region** and **bucket name**, then add them to your `.env` file:

```env
AWS_BUCKET_NAME=geeks-demo-bucket
AWS_REGION=eu-north-1
```

---

## 3. IAM — Creating a Policy and User

By default, only the bucket owner can access it. You need to create a dedicated IAM user to represent your application.

### Create a Policy

1. Search for **IAM** in the AWS console.
2. Navigate to **Policies** in the left sidebar and click **Create policy**.
3. Under **Select a service**, choose **S3**.
4. Under **Actions allowed**:
   - **Read:** `GetObject`
   - **Write:** `PutObject`, `DeleteObject`,`ListBucket`
5. Under **Resources**, choose **Specific** → **Add ARNs**:
   - **Resource bucket name:** your bucket name (e.g. `geeks-demo-bucket`)
   - **Resource object name:** select **Any object name**
   - Click **Add ARNs**
6. Click **Next**, give the policy a name (e.g. `geeks-policy`), then click **Create policy**.

### Create a User

1. Navigate to **Users** in the left sidebar and click **Create user**.
2. Give the user a name.
3. Check **Provide user access to the AWS Management Console** (optional).
4. Click **Next**, choose **Attach policies directly**, select the policy you just created, click **Next**, then **Create user**.

### Create an Access Key

1. In the top right of the user page, click **Create access key**.
2. Choose **Local code**, confirm the policy, then click **Next**.
3. Give a description tag value (e.g. `aws-tag-value`).
4. Copy the generated **Access key** and **Secret access key** into your `.env`:

```env
ACCESS_KEY=YOUR_AWS_ACCESS_KEY_ID
SECRET_ACCESS_KEY=YOUR_AWS_SECRET_ACCESS_KEY
```

---

## 4. Connecting to S3 from Your App

**Install the AWS SDK:**

```bash
npm install @aws-sdk/client-s3
```

> Reference: [https://www.npmjs.com/package/@aws-sdk/client-s3](https://www.npmjs.com/package/@aws-sdk/client-s3)

**Read env variables and configure the S3 client:**

```js
import { S3Client, PutObjectCommand } from "@aws-sdk/client-s3";

const awsBucketName = process.env.AWS_BUCKET_NAME;
const awsRegion = process.env.AWS_REGION;
const accessKey = process.env.ACCESS_KEY;
const secretAccessKey = process.env.SECRET_ACCESS_KEY;

const s3 = new S3Client({
    credentials: {
        accessKeyId: accessKey,
        secretAccessKey: secretAccessKey
    },
    region: awsRegion
});
```

---

## 5. Uploading Files to S3

Build the upload params and send the command:

```js
const params = {
    Bucket: awsBucketName,
    Key: req.file.originalname,
    Body: image.buffer,
    ContentType: req.file.mimetype
};

const command = new PutObjectCommand(params);
await s3.send(command);
```

You can verify the upload at:

```
https://eu-north-1.console.aws.amazon.com/s3/buckets/geeks-demo-bucket?region=eu-north-1&tab=objects
```

---

## 6. Generating Unique Filenames

If two files share the same name, the newer one will overwrite the existing one in S3. To avoid this, generate a random name for each upload.

```js
import crypto from "crypto";

const randomImageName = (bytes = 32) => crypto.randomBytes(bytes).toString("hex");
```

Then use it in your params:

```js
const params = {
    Bucket: awsBucketName,
    Key: randomImageName() + "-" + image.originalname,
    Body: image.buffer,
    ContentType: req.file.mimetype
};
```

---

## 7. Generating a Signed URL (with expiry)

A signed URL gives temporary access to a private S3 object.

**Install:**

```bash
npm install @aws-sdk/s3-request-presigner
```

> Reference: [https://www.npmjs.com/package/@aws-sdk/s3-request-presigner](https://www.npmjs.com/package/@aws-sdk/s3-request-presigner)

**Import:**

```js
import { S3Client, PutObjectCommand, GetObjectCommand } from "@aws-sdk/client-s3";
import { getSignedUrl } from "@aws-sdk/s3-request-presigner";
```

**Endpoint:**

```js
app.post("/posts", upload.single("image"), async (req, res) => {
    try {
        const { title, description } = req.body;
        const image = req.file;

        const params = {
            Bucket: awsBucketName,
            Key: randomImageName() + "-" + image.originalname,
            Body: image.buffer,
            ContentType: req.file.mimetype
        };

        const send_command = new PutObjectCommand(params);
        await s3.send(send_command);

        const get_command = new GetObjectCommand(params);
        const url = await getSignedUrl(s3, get_command, { expiresIn: 3600 });

        console.log({ title, description, image });
        res.status(200).json({ message: "Post created successfully", url });
    } catch (error) {
        console.log(error);
        res.status(500).json({ message: "Error creating post", error: error.message });
    }
});
```

---

## 8. Public Bucket URL (no expiry)

To serve images via a permanent public URL, you need to make the bucket publicly accessible.

### Step 1 — Make the Bucket Public

1. Go to your S3 bucket → **Permissions** tab.
2. Turn off **Block all public access**.
3. Add the following **Bucket Policy** (replace `geeks-demo-bucket` with your bucket name):

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PublicReadGetObject",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::geeks-demo-bucket/*"
        }
    ]
}
```

### Step 2 — Build the Permanent URL

```js
app.post("/posts", upload.single("image"), async (req, res) => {
    try {
        const { title, description } = req.body;
        const image = req.file;

        const params = {
            Bucket: awsBucketName,
            Key: randomImageName() + "-" + image.originalname,
            Body: image.buffer,
            ContentType: req.file.mimetype
        };

        const send_command = new PutObjectCommand(params);
        await s3.send(send_command);

        const url = `https://${awsBucketName}.s3.${awsRegion}.amazonaws.com/${params.Key}`;

        console.log({ title, description, image });
        res.status(200).json({ message: "Post created successfully", url });
    } catch (error) {
        console.log(error);
        res.status(500).json({ message: "Error creating post", error: error.message });
    }
});
```
# aws-s3-expressjs
# aws-s3-expressjs
# aws-s3-expressjs
# aws-s3-expressjs
# aws-s3-expressjs
