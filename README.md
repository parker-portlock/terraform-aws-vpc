# Terraform - VPC Infrastructure Provisioning
This deployment creates two VPCs (one in each region), with inside and outside subnets in each VPC. This also deploys EC2 instances inside of the VPCs.

Some concepts forked from https://github.com/kung-foo/multiregion-terraform

#### Changelog
* 2019-07-17 - Converted to support Terraform 0.12

## Concept
The goal of this code is to define a multi-regional, redundant, and resilient network architecture in AWS as the standard for new product implementations going forward. Ideally, the an infrastructure team will create and maintain the VPC portion of the overall AWS infrastructure to maintain design and security standards. With that being said, the goal of the infrastructure team should be to allow other members of the engineering teams to operate in these VPCs with ease.

## Design
These design specifications should allow for some flexibility for different products to reside in AWS while maintaining a certain level of organization:

* VPCs created per region
* Private and public subnets created for each availability-zone
* Products are segmented by subnets
* VPC sharing can be allowed to use one VPC per region for diffrent accounts if needed. (not in current code)

## Implementation
This code is deployable out-of-the-box. Key things to change are the following:

* The desired regions used for the VPCs
* The CIDRSs for the VPCs
* The size of the subnets in each VPC

### Region Configuration

Found in `/main.tf`, create a module folder and call it with:
```
module "vpc-us-west-2" {
  source = "./vpc-us-west-2"
  region = "us-west-2"
}
```

### VPC CIDR Configuration

In the `module.tf` file for the region, modify:
```
resource "aws_vpc" "main-eu-west" {
    cidr_block              = "10.26.0.0/16"
    ...
    }
}
```

### Subnet Configuration

Modify the subnet size by adjusting the number of network bits in the configuration using `cidrsubnet`:
```
resource "aws_subnet" "public" {
  count                   = length(data.aws_availability_zones.available.names)
  vpc_id                  = aws_vpc.main-us-west.id
  cidr_block              = cidrsubnet(aws_vpc.main-us-west.cidr_block, 6, count.index)
  map_public_ip_on_launch = true
  availability_zone       = element(data.aws_availability_zones.available.names, count.index)

  tags = {
    Name = "cdn-${element(data.aws_availability_zones.available.names, count.index)}-public"
  }
}
```

### VPC Peering

This is accomplished by creating modules for both initiating and accepting connections. The target regions and IDs can be grabbed from the outputs generated by the other modules.
```
module "peering-us-east-to-us-west-init" {
  source         = "./pcx-us-east-to-us-west-init"
  request_region = module.vpc-us-west-2.region
  request_vpc_id = module.vpc-us-west-2.vpc_id
  accept_region  = module.vpc-us-east-1.region
  accept_vpc_id  = module.vpc-us-east-1.vpc_id
}

module "peering-us-east-to-us-west-accept" {
  source        = "./pcx-us-east-to-us-west-accept"
  pcx_id        = module.peering-us-east-to-us-west-init.pcx_id
  accept_region = module.vpc-us-east-1.region
}
```

### VPC Endpoint Connectivity

An example of how to implement a VPC endpoint can be found in the us-west-2 `module.tf` file:
```
#create S3 VPC endpoint
resource "aws_vpc_endpoint" "s3" {
  vpc_id       = aws_vpc.main-us-west.id
  service_name = "com.amazonaws.us-west-2.s3"
}

#VPC endpoint route table association
resource "aws_vpc_endpoint_route_table_association" "s3" {
  count           = length(data.aws_availability_zones.available.names)
  route_table_id  = element(aws_route_table.private.*.id, count.index)
  vpc_endpoint_id = aws_vpc_endpoint.s3.id
}
```