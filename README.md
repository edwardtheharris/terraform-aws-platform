# Deploying GrayMeta Platform with Terraform

Available on the [Terraform Registry](https://registry.terraform.io/modules/graymeta/platform/aws)

* Decide what email address will be used as the `From:` address for email notifications generated by the platform. You must use a [Verified SES email address](http://docs.aws.amazon.com/ses/latest/DeveloperGuide/verify-email-addresses.html) for this address. Make sure you verify the address in the AWS region into which you will deploy the platform. Submit a request to [move out of the SES sandbox](http://docs.aws.amazon.com/ses/latest/DeveloperGuide/request-production-access.html).
* Pick a _platform instance id_ for this deployment of the GrayMeta platform. A short, descriptive name like `production`, `labs`, `test`, etc. that can be used to uniquely identify this deployment of the GrayMeta Platform within your environment. Record this as variable `platform_instance_id`
* Pick which AWS region you want to deploy into from the list below:
  * us-east-1
  * us-east-2
  * us-west-2
  * ap-southeast-2
  * eu-west-1
* Pick the hostname which will be used to access the platform (example: graymeta.example.com). Record this value as the `dns_name` variable.
* Procure a valid SSL certificate for the hostname chosen in the previous step. Self-signed certificates will NOT work. Upload the SSL certificate to Amazon Certificate Manager in the same region you will be deploying the platform into. After upload, record the ARN of the certificate as variable `ssl_certificate_arn`
* Create an S3 bucket to store thumbnails, transcoded video and audio preview files, and metadata files. Record the ARN of the s3 bucket as variable `file_storage_s3_bucket_arn`.
* Create an S3 bucket to store usage reports. Record the ARN of the s3 bucket as variable `usage_s3_bucket_arn`.
* Stand up a network. The `modules/network` module can be used to stand up all of the networking components. If you want to stand the network up yourself, please review the [Networking Requirements](README-networking.md)
* Decide the CIDR or CIDRs that will be allowed access to the platform. Record as comma delimited lists of CIDR blocks.
  * `platform_access_cidrs` - The list of CIDRs that will be allowed to access the web ports of the platform
  * `ssh_cidr_blocks` - The list of CIDRs that will be allowed SSH access to the servers. This is typically an admin or VPN subnet somewhere within your VPC.
* Fill in the rest of the variables, review the output of a `terraform plan`, then apply the changes.
* Create a CNAME from your `dns_name` to the value of the `GrayMetaPlatformEndpoint` output. This needs to be publicly resolvable.
* Load `https://dns_name` where _dns\_name_ is the name you chose above. The default username is `admin@graymeta.com`. The password is set to the instance ID of one of the Services nodes of the platform. These are tagged with the name `GrayMetaPlatform-${platform_instance_id}-Services` in the EC2 console. There should be at least 2 nodes running. Try the instance ID of both. After logging in for the first time, change the password of the `admin@graymeta.com` account. Create other accounts as necessary.

## Example

```

locals {
    platform_instance_id = "labs"
    key_name             = "somekey"
}

provider "aws" {
    region = "us-east-1"
}

module "servicesiam" {
    source = "github.com/graymeta/terraform-aws-platform//modules/servicesiam?ref=v0.0.20"

    platform_instance_id = "${local.platform_instance_id}"
}

module "network" {
    source = "github.com/graymeta/terraform-aws-platform//modules/network?ref=v0.0.20"

    platform_instance_id = "${local.platform_instance_id}"
    region               = "us-east-1"
    az1                  = "us-east-1a"
    az2                  = "us-east-1b"
}

module "platform" {
    source = "github.com/graymeta/terraform-aws-platform?ref=v0.0.20"

    platform_instance_id       = "${local.platform_instance_id}"
    customer                   = "mycompanyname"
    region                     = "us-east-1"
    key_name                   = "${local.key_name}"
    platform_access_cidrs      = "0.0.0.0/0"
    file_storage_s3_bucket_arn = "arn:aws:s3:::cfn-file-api"
    usage_s3_bucket_arn        = "arn:aws:s3:::cfn-usage-api"
    dns_name                   = "foo.cust.graymeta.com"
    ssl_certificate_arn        = "arn:aws:acm:us-east-1:913397769129:certificate/507e54c3-51a4-45b3-ae21-9cb4647bb671"
    notifications_from_addr    = "noreply@example.com"
    services_iam_role_name     = "${module.servicesiam.services_iam_role_name}"

    # RDS Configuration
    db_username          = "mydbuser"
    db_password          = "mydbpassword"
    db_instance_size     = "db.t2.small"
    db_allocated_storage = "100"
    db_storage_encrypted = false    # optional
    db_kms_key_id        = ""       # optional

    # ECS Cluster Configuration
    ecs_instance_type    = "c4.large"
    ecs_max_cluster_size = 2
    ecs_min_cluster_size = 1
    ecs_volume_size      = 1000

    # Services Cluster Configuration
    services_instance_type    = "m4.large"
    services_max_cluster_size = 4
    services_min_cluster_size = 2
    services_iam_role_name    = "${module.servicesiam.services_iam_role_name}"

    # Encryption Tokens - 32 character alpha numberic strings
    client_secret_fe       = "012345678901234567890123456789ab"
    client_secret_internal = "012345678901234567890123456789ab"
    jwt_key                = "012345678901234567890123456789ab"
    encryption_key         = "012345678901234567890123456789ab"

    # Elasticache Configuration
    elasticache_instance_type_services = "cache.m4.large"
    elasticache_instance_type_facebox  = "cache.m4.large"

    # ElasticSearch Configuration
    elasticsearch_volume_size = "100"
    elasticsearch_instance_type = "m4.large.elasticsearch"
    elasticsearch_dedicated_master_type = "m4.large.elasticsearch"
    elasticsearch_dedicated_master_count = "3"
    elasticsearch_instance_count = "2"

    services_nat_ip           = "${module.network.services_nat_ip}/32"
    ecs_nat_ip                = "${module.network.ecs_nat_ip}/32"
    ecs_subnet_id             = "${module.network.ecs_subnet_id}"
    public_subnet_id_1        = "${module.network.public_subnet_id_1}"
    public_subnet_id_2        = "${module.network.public_subnet_id_2}"
    rds_subnet_id_1           = "${module.network.rds_subnet_id_1}"
    rds_subnet_id_2           = "${module.network.rds_subnet_id_2}"
    services_subnet_id_1      = "${module.network.services_subnet_id_1}"
    services_subnet_id_2      = "${module.network.services_subnet_id_2}"
    elasticsearch_subnet_id_1 = "${module.network.elasticsearch_subnet_id_1}"
    elasticsearch_subnet_id_2 = "${module.network.elasticsearch_subnet_id_2}"
    ssh_cidr_blocks           = "10.0.0.0/24,10.0.1.0/24"

    # Optional Error Reporting Configurations
    rollbar_token = ""

    # Facebox API Key
    facebox_key                      = ""

    # Box (Box.com)
    box_client_id                    = ""
    box_client_secret                = ""

    # Google maps (for plotting geocoded results on a map in the UI
    google_maps_key                  = ""
}

output "GrayMetaPlatformEndpoint" {
    value = "${module.platform.GrayMetaPlatformEndpoint}"
}
```
