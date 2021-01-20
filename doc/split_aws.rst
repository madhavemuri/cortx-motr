Split AWS S3
================

hctl bootstrap --mkfs motr-ldr2-2ios.yaml

dd if=/dev/urandom of=512M bs=1M count=512

aws s3 ls

aws s3 cp 512M s3://bkt1

aws s3api create-multipart-upload  --bucket bkt1 --key 512M

man split

split 512M -n 2

aws s3api upload-part  --bucket bkt1 --key 512M --part-number 1 --body xaa --upload-id "0d4dc4e8-672a-43e7-a6e4-290386275a31"

aws s3api upload-part  --bucket bkt1 --key 512M --part-number 2 --body xab --upload-id "0d4dc4e8-672a-43e7-a6e4-290386275a31"

aws s3api list-parts --bucket bkt1 --key 512M --upload-id "0d4dc4e8-672a-43e7-a6e4-290386275a31"
