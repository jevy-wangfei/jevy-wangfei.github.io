---

title: Get Data from Google Storage

category: Gist
tags: [Google]
date: 2019-09-11
---

## Software
- Install gcloud with gsutil included
- 

## Login google cloud using json file
document: https://cloud.google.com/sdk/gcloud/reference/auth/activate-service-account

```bash
gcloud auth activate-service-account test@test.iam.gserviceaccount.com --key-file=./secret.json

```

## Copy data
Reference this document for details: https://cloud.google.com/storage/docs/gsutil/commands/cp

Copy bucket data to local
```bash
gsutil -m cp -r gs://my-bucket/data ./data
```

## rsync data
Reference this document for details: https://cloud.google.com/storage/docs/gsutil/commands/rsync

Sync local data from bucket
```bash
gsutil -m rsync -d -r gs://mybucket/data  ./data
```
