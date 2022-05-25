## AUTOMATE INFRASTRUCTURE WITH IAC USING TERRAFORM PART 1
In this project I would be building the same set up as previous projects with the power of Infrastructure as Code (IaC) usin Terraform

#### Prerequisites before you begin writing Terraform code
- Create an IAM user, name it terraform (ensure that the user has only programatic access to your AWS account) and grant this user AdministratorAccess permissions.
- Copy the secret access key and access key ID. Save them in a notepad temporarily.
- I would be using VScode as my editor 
- install required plugins for AWS and terraform 
- Add a profile to connect to AWS using the secret access key and access key ID


The secret recipe of a successful Terraform projects consists of:
- Your understanding of your goal (desired AWS infrastructure end state)
- Your knowledge of the IaC technology used (in this case – Terraform)
- Your ability to effectively use up to date Terraform documentation


#### VPC | Subnets | Security Groups

Let us create a directory structure

Open your Visual Studio Code and:
- Create a folder called PBL
- Create a file in the folder, name it main.tf

#### Provider and VPC resource section

- Add AWS as a provider, and a resource to create a VPC in the main.tf file.
- Provider block informs Terraform that we intend to build infrastructure within AWS.
- Resource block will create a VPC.

     
      provider "aws" {
        region = "eu-west-2"
        shared_credentials_file = "~/.aws/credentials"
        profile                  = "terraform-aws"
      }
      
      # Create VPC
      resource "aws_vpc" "main" {
          cidr_block                     = "172.16.0.0/16"
          enable_dns_support             = "true"
          enable_dns_hostnames           = "true"
          enable_classiclink             = "false"
          enable_classiclink_dns_support = "false"
      }
      
  - Run terraform plan    
  - Then, if you are happy with changes planned, execute terraform apply    
  
  
  #### Subnets resource section
According to our architectural design, we require 6 subnets:

- 2 public
- 2 private for webservers
- 2 private for data layer

Let us create the first 2 public subnets. Add below configuration to the main.tf file:

    # Create public subnets1
    resource "aws_subnet" "public1" {
    vpc_id                     = aws_vpc.main.id
    cidr_block                 = "172.16.0.0/24"
    map_public_ip_on_launch    = true
    availability_zone          = "eu-west-2a"

    }

    # Create public subnet2
    resource "aws_subnet" "public2" {
    vpc_id                     = aws_vpc.main.id
    cidr_block                 = "172.16.1.0/24"
    map_public_ip_on_launch    = true
    availability_zone          = "eu-west-2b"
    }
    
Run **terraform plan** and **terraform apply**

Now let us improve our code by refactoring it. First, destroy the current infrastructure. Run **terraform destroy**


#### Fixing The Problems By Code Refactoring

Fixing Hard Coded Values: We will introduce variables, and remove hard coding.

- Starting with the provider block, declare a variable named region, give it a default value, and update the provider section by referring to the declared variable.
   
      variable "region" {
        default = "eu-west-2"
      }

       provider "aws" {
        region = var.region
       }
- Do the same to cidr value in the vpc block, and all the other arguments.     
          
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
We will explore the use of Terraform’s Data Sources to fetch information outside of Terraform. In this case, from AWS

    # Get list of availability zones
        data "aws_availability_zones" "available" {
        state = "available"
        }
        
 To make use of this new data resource, we will need to introduce a count argument in the subnet block: Something like this.
 
      # Create public subnet1
    resource "aws_subnet" "public" { 
        count                   = 2
        vpc_id                  = aws_vpc.main.id
        cidr_block              = "172.16.1.0/24"
        map_public_ip_on_launch = true
        availability_zone       = data.aws_availability_zones.available.names[count.index]

    }

#### Let’s make cidr_block dynamic.
 We will introduce a function cidrsubnet() to make this happen. Its parameters are cidrsubnet(prefix, newbits, netnum)
   
    # Create public subnet1
    resource "aws_subnet" "public" { 
        count                   = 2
        vpc_id                  = aws_vpc.main.id
        cidr_block              = cidrsubnet(var.vpc_cidr, 4 , count.index)
        map_public_ip_on_launch = true
        availability_zone       = data.aws_availability_zones.available.names[count.index]

    }
    
#### The final problem to solve is removing hard coded count value.

Since data.aws_availability_zones.available.names returns a list like ["eu-central-1a", "eu-central-1b", "eu-central-1c"] we can pass it into a lenght function and get number of the AZs.
Like this length(["data.aws_availability_zones.available.names")

The length function will return number 3 to the count argument, but what we actually need is 2. do the following to fix this:

- Declare a variable to store the desired number of public subnets, and set the default value

      variable "preferred_number_of_public_subnets" {
         default = 2
        }
- Next, update the count argument with a condition. Terraform needs to check first if there is a desired number of subnets. Otherwise, use the data returned by the lenght function. See how that is presented below.

        # Create public subnets
            resource "aws_subnet" "public" {
              count  = var.preferred_number_of_public_subnets == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_public_subnets   
              vpc_id = aws_vpc.main.id
              cidr_block              = cidrsubnet(var.vpc_cidr, 4 , count.index)
              map_public_ip_on_launch = true
              availability_zone       = data.aws_availability_zones.available.names[count.index]

            }
Note: if preferred_number_of_public_subnets variable is set to null then **3 subnets** would be created

#### Introducing variables.tf & terraform.tfvars
Instead of havng a long lisf of variables in main.tf file, we can actually make our code a lot more readable and better structured by moving out some parts of the configuration content to other files.

- Create a new file and name it variables.tf
- Copy all the variable declarations into the new file.
- Create another file, name it terraform.tfvars
- Set values for each of the variables.

#### Main.tf
