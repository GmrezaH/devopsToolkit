gitlab backup s3 

```yml
apiVersion: ceph.rook.io/v1
kind: CephObjectStoreUser
metadata:
  name: gitlab-backup-user
  namespace: rook-ceph
spec:
  store: ceph-objectstore
  displayName: gitlab-backup-user
  quotas:
    maxBuckets: 1
    maxSize: 10G
    maxObjects: 100
---
apiVersion: objectbucket.io/v1alpha1
kind: ObjectBucketClaim
metadata:
  name: gitlab-backup
  namespace: rook-ceph
spec:
  bucketName: gitlab-backup
  # generateBucketName: gitlab-backup
  storageClassName: ceph-bucket
  additionalConfig:
    maxObjects: "100"
    maxSize: "10G"
```

bucket policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Statement1",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam:::user/gitlab-backup-user"
      },
      "Action": [
        "s3:PutObject"
      ],
      "Resource": [
        "arn:aws:s3:::gitlab-backup/*",
        "arn:aws:s3:::gitlab-backup"
      ]
    }
  ]
}
```

bucket lifecycle

```json

{
  "LifecycleConfiguration": {
    "Rule": [
      {
        "ID": "AbortIncompleteMultipartUploads",
        "Prefix": null,
        "Status": "Enabled",
        "AbortIncompleteMultipartUpload": {
          "DaysAfterInitiation": "1"
        }
      },
      {
        "ID": "ExpireAfter30Days",
        "Prefix": null,
        "Status": "Enabled",
        "Expiration": {
          "Days": "30"
        }
      }
    ]
  }
}
```

test s3 bucket

```bash
cat <<EOF > ~/.aws/credentials
[default]
aws_access_key_id = 9U5GZTI0SZQ86JQ4
aws_secret_access_key = ihdVz3fXlMmRxhJBZ3WOURWNHxLCPIos1Ez

[gitlab-backup]
aws_access_key_id = P2XF4PMPG3HGAGETI
aws_secret_access_key = dvJ9AimIjRQz95kxildz976tSf6uHjjHB2q
EOF

export S3_HOST=http://objectstore.ceph.testbed.moh
export BUCKET_NAME=gitlab-backup

echo "helloworld" > /tmp/testbucketest
s5cmd --profile gitlab-backup  --endpoint-url $S3_HOST cp /tmp/testbucketest s3://$BUCKET_NAME
s5cmd --profile default  --endpoint-url $S3_HOST ls s3://$BUCKET_NAME
s5cmd --profile default  --endpoint-url $S3_HOST cat s3://$BUCKET_NAME/testbucketest
```

enable 2 factor auth on gmail and then create an app password


create vault variables:

```
cat <<EOF > roles/gitlab/defaults/main/vault.yml
gitlab_root_password: XRkx6jZ5UlSFwt3WRZrMm

gitlab_smtp_username: 'gmrezah.biz@gmail.com'
gitlab_smtp_password: 'xrgc'

s3_gitlab_backup_access_key: P2XF4PMPG3HGAG
s3_gitlab_backup_secret_key: dvJkxildz976tSjHB2qVR14E

gitlab_backup_encryption_pass: QWEqwe
EOF

ansible-vault encrypt roles/gitlab/defaults/main/vault.yml
```


Test smtp 
gitlab-rails console
Notify.test_email('rezagh1377@gmail.com', 'Message Subject', 'Message Body').deliver_now

