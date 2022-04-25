##
AUTOMATE INFRASTRUCTURE WITH IAC USING TERRAFORM
##

- Create an S3 bucket to store Terraform state file. After creating it, run the following command to check it.

![](https://github.com/akinolafusi/DAREY-PBL-PROJECTS3/blob/838856b21113a8c89648d31f84b53229b1d9e3ea/P16/bucket.PNG)
```
import boto3
s3 = boto3.resource('s3')
for bucket in s3.buckets.all():
    print(bucket.name)
```

###
VPC | SUBNETS | SECURITY GROUPS
###

- Provider and VPC resource section of the configuration. 
```
provider "aws" {
  region = "eu-central-1"
}

# Create VPC
resource "aws_vpc" "main" {
  cidr_block                     = "172.16.0.0/16"
  enable_dns_support             = "true"
  enable_dns_hostnames           = "true"
  enable_classiclink             = "false"
  enable_classiclink_dns_support = "false"
}
```
and run
```
terraform init
```
![](https://github.com/akinolafusi/DAREY-PBL-PROJECTS3/blob/838856b21113a8c89648d31f84b53229b1d9e3ea/P16/init.PNG)

- Then run
```
terraform plan
```

- Subnets resource section
According to our architectural design, we require 6 subnets:

- 2 public
- 2 private for webservers
- 2 private for data layer

```
# Create public subnets1
    resource "aws_subnet" "public1" {
    vpc_id                     = aws_vpc.main.id
    cidr_block                 = "172.16.0.0/24"
    map_public_ip_on_launch    = true
    availability_zone          = "eu-central-1a"

}

# Create public subnet2
    resource "aws_subnet" "public2" {
    vpc_id                     = aws_vpc.main.id
    cidr_block                 = "172.16.1.0/24"
    map_public_ip_on_launch    = true
    availability_zone          = "eu-central-1b"
}
```

- Run terraform apply to provision the resources.
```
terraform apply
```
![](https://github.com/akinolafusi/DAREY-PBL-PROJECTS3/blob/838856b21113a8c89648d31f84b53229b1d9e3ea/P16/terraform%20apply.PNG)

- VPC Created

![](https://github.com/akinolafusi/DAREY-PBL-PROJECTS3/blob/838856b21113a8c89648d31f84b53229b1d9e3ea/P16/terraform%20apply.PNG)

- Run terraform destroy to delete the provisioned resources
```
terraform destroy
```
![](https://github.com/akinolafusi/DAREY-PBL-PROJECTS3/blob/838856b21113a8c89648d31f84b53229b1d9e3ea/P16/destroy1.PNG)

##

CODE REFACTORING
##

- Fixing The Problems By Code Refactoring
- Introduce variables to remove hard coding. Starting with the region variable.
```
 variable "region" {
        default = "eu-central-1"
    }

    provider "aws" {
        region = var.region
    }
```
- same thing to cidr and vpc
```
variable "region" {
        default = "eu-central-1"
    }

    variable "vpc_cidr" {
        default = "172.16.0.0/16"
    }

    variable "enable_dns_support" {
        default = "true"
    }

    variable "enable_dns_hostnames" {
        default ="true" 
    }

    variable "enable_classiclink" {
        default = "false"
    }

    variable "enable_classiclink_dns_support" {
        default = "false"
    }

    provider "aws" {
    region = var.region
    }

    # Create VPC
    resource "aws_vpc" "main" {
    cidr_block                     = var.vpc_cidr
    enable_dns_support             = var.enable_dns_support 
    enable_dns_hostnames           = var.enable_dns_support
    enable_classiclink             = var.enable_classiclink
    enable_classiclink_dns_support = var.enable_classiclink

    }
```

- Fixing the availability zones using "data module"
```
# Get list of availability zones
        data "aws_availability_zones" "available" {
        state = "available"
        }
```

- Introduce the count argument in the subnet block.
```
  # Create public subnet1
    resource "aws_subnet" "public" { 
        count                   = 2
        vpc_id                  = aws_vpc.main.id
        cidr_block              = "172.16.1.0/24"
        map_public_ip_on_launch = true
        availability_zone       = data.aws_availability_zones.available.names[count.index]

    }
```

- make the cidr_block dynamic
```
# Create public subnet1
    resource "aws_subnet" "public" { 
        count                   = 2
        vpc_id                  = aws_vpc.main.id
        cidr_block              = cidrsubnet(var.vpc_cidr, 4 , count.index)
        map_public_ip_on_launch = true
        availability_zone       = data.aws_availability_zones.available.names[count.index]

    }
```
- Remove the hard coded count value

```
# Create public subnets
resource "aws_subnet" "public" {
  count  = var.preferred_number_of_public_subnets == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_public_subnets   
  vpc_id = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 4 , count.index)
  map_public_ip_on_launch = true
  availability_zone       = data.aws_availability_zones.available.names[count.index]

}

```
- Final look of the configuration
```
# Get list of availability zones
data "aws_availability_zones" "available" {
state = "available"
}

variable "region" {
      default = "eu-central-1"
}

variable "vpc_cidr" {
    default = "172.16.0.0/16"
}

variable "enable_dns_support" {
    default = "true"
}

variable "enable_dns_hostnames" {
    default ="true" 
}

variable "enable_classiclink" {
    default = "false"
}

variable "enable_classiclink_dns_support" {
    default = "false"
}

  variable "preferred_number_of_public_subnets" {
      default = 2
}

provider "aws" {
  region = var.region
}

# Create VPC
resource "aws_vpc" "main" {
  cidr_block                     = var.vpc_cidr
  enable_dns_support             = var.enable_dns_support 
  enable_dns_hostnames           = var.enable_dns_support
  enable_classiclink             = var.enable_classiclink
  enable_classiclink_dns_support = var.enable_classiclink

}

# Create public subnets
resource "aws_subnet" "public" {
  count  = var.preferred_number_of_public_subnets == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_public_subnets   
  vpc_id = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 4 , count.index)
  map_public_ip_on_launch = true
  availability_zone       = data.aws_availability_zones.available.names[count.index]

}
```

- Note: You should try changing the value of preferred_number_of_public_subnets variable to null and notice how many subnets get created.

#
Introducing variables.tf & terraform.tfvars
#

- Create a new file and name it variables.tf. Move all the variable configuration in main.tf to variables.tf.
![](https://github.com/akinolafusi/DAREY-PBL-PROJECTS3/blob/838856b21113a8c89648d31f84b53229b1d9e3ea/P16/vars.PNG)

```
variable "region" {
      default = "eu-central-1"
}

variable "vpc_cidr" {
    default = "172.16.0.0/16"
}

variable "enable_dns_support" {
    default = "true"
}

variable "enable_dns_hostnames" {
    default ="true" 
}

variable "enable_classiclink" {
    default = "false"
}

variable "enable_classiclink_dns_support" {
    default = "false"
}

  variable "preferred_number_of_public_subnets" {
      default = null
}
```

- Create another file, name it terraform.tfvars and Set values for each of the variables.

![](https://github.com/akinolafusi/DAREY-PBL-PROJECTS3/blob/838856b21113a8c89648d31f84b53229b1d9e3ea/P16/tfvars.PNG)

```

region = "eu-central-1"

vpc_cidr = "172.16.0.0/16" 

enable_dns_support = "true" 

enable_dns_hostnames = "true"  

enable_classiclink = "false" 

enable_classiclink_dns_support = "false" 

preferred_number_of_public_subnets = 2
````
- Current File Structure
```
 PBL
    ├── main.tf
    ├── terraform.tfstate
    ├── terraform.tfstate.backup
    ├── terraform.tfvars
    └── variables.tf
```

- Run terraform plan and ensure everything works..