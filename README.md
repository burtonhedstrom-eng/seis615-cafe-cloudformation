# SEIS 615 — CloudFormation cafe-app

Two CloudFormation templates that rebuild the highly available **cafe-app**
from the "Elasticity and availability" assignment in **us-east-1**
(AZs `us-east-1a` and `us-east-1b`). Every resource is tagged
`Project=cafe-cfn`.

## What the two stacks build

### `lab-network.yaml` — network stack (deploy first)

- VPC `10.0.0.0/16` with DNS support and hostnames.
- Two public subnets (`10.0.0.0/24`, `10.0.1.0/24`) with auto-assign public
  IPv4, and two private subnets (`10.0.2.0/23`, `10.0.4.0/23`), one of each
  per AZ.
- Internet gateway and a public route table (`0.0.0.0/0` → IGW) for both
  public subnets.
- One NAT gateway with its own Elastic IP in each public subnet, and one
  private route table per AZ sending `0.0.0.0/0` to the NAT gateway in the
  same AZ.
- Exports consumed by the application stack: `-VPCID`, `-PublicSubnet1ID`,
  `-PublicSubnet2ID`, `-PrivateSubnet1ID`, `-PrivateSubnet2ID`, and
  `-SubnetID` (kept pointing at public subnet 1 for backward compatibility).

### `lab-application.yaml` — application stack (deploy second)

Imports the network exports with `Fn::ImportValue` and the
`NetworkStackName` parameter, then builds:

- **Security groups**: bastion (SSH 22 from `MyIP` only), ALB (HTTP 80 from
  anywhere), app (HTTP 80 from the ALB SG, SSH 22 from the bastion SG), and
  DB (MariaDB 3306 from the app SG only).
- **Bastion host** (Amazon Linux 2023) in public subnet 1, for SSH checks.
- **Seven `/cafe/*` Secrets Manager secrets**: `showServerInfo`, `timeZone`,
  `currency`, `dbName`, `dbUser`, `dbUrl` (set to the RDS endpoint), and
  `dbPassword` — **generated** by Secrets Manager as a plain string
  (excluding `/ @ " ' \` and spaces). RDS reads the password through a
  `{{resolve:secretsmanager:...}}` dynamic reference, so no password is ever
  a parameter or stored in the repo.
- **RDS for MariaDB** `10.6.25`, `db.t3.micro`, gp2 20 GiB, Multi-AZ (via the
  `DBMultiAZ` condition), not publicly accessible, in a DB subnet group that
  spans both private subnets. `DeletionPolicy` and `UpdateReplacePolicy` are
  `Delete` (no leftover snapshot).
- **IAM role + instance profile** (no fixed names, so `CAPABILITY_IAM` is
  enough) granting the app instances read access to only the seven `/cafe/*`
  secrets, plus `AmazonSSMManagedInstanceCore`.
- **Launch template** (Amazon Linux 2023 from SSM, app SG, instance profile,
  detailed monitoring, no subnet). At boot it runs `cfn-init` against the
  template metadata to install `httpd`, `php`, `php-mysqlnd`, `mariadb105`
  and `git`, deploy the cafe-app (`cafe.zip` into `/var/www/html`, the AWS
  PHP SDK into `/var/www/html/cafe`, `chmod -R +r`, `date.timezone` set to
  `America/New_York`), patch `serverInfo.php` (public-ipv4 → local-ipv4 and
  placement/instance-id → instance-id), and load `cafe_db` into RDS. The load
  is idempotent and safe when two instances boot together: it runs in one
  `mysql` session that holds a `GET_LOCK` named lock and skips the load when
  `cafe_db.product` already exists. Boot steps are logged to
  `/var/log/cafe-bootstrap.log`; the DB password is never logged. The
  instance then sends `cfn-signal` to the Auto Scaling group.
- **Target group** (HTTP 80, health check `HealthCheckPath`, success 200),
  **internet-facing Application Load Balancer** across both public subnets,
  and an **HTTP:80 listener**.
- **Auto Scaling group** across both private subnets with ELB health checks,
  a `CreationPolicy` that waits for one signal per desired instance, an
  `AutoScalingRollingUpdate` update policy, and a **target-tracking scaling
  policy** on `ASGAverageCPUUtilization`.

## Parameters

### `lab-network.yaml`

| Parameter | Default | Purpose |
|-----------|---------|---------|
| `VpcCidr` | `10.0.0.0/16` | CIDR block for the VPC. |
| `PublicSubnet1Cidr` | `10.0.0.0/24` | Public subnet 1 (AZ index 0). |
| `PublicSubnet2Cidr` | `10.0.1.0/24` | Public subnet 2 (AZ index 1). |
| `PrivateSubnet1Cidr` | `10.0.2.0/23` | Private subnet 1 (AZ index 0). |
| `PrivateSubnet2Cidr` | `10.0.4.0/23` | Private subnet 2 (AZ index 1). |

### `lab-application.yaml`

| Parameter | Default | Purpose |
|-----------|---------|---------|
| `NetworkStackName` | `lab-network` | Name of the network stack whose exports are imported. |
| `MyIP` | *(none, required)* | Your public IP as a `/32` CIDR, the only source allowed to SSH to the bastion. |
| `KeyName` | `cafe-cfn-key` | Existing EC2 key pair for SSH. |
| `LatestAmiId` | `/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64` | Amazon Linux 2023 AMI resolved from SSM. |
| `InstanceType` | `t3.micro` | Instance type for the bastion and app instances. |
| `AsgMinSize` | `2` | Minimum app instances. |
| `AsgMaxSize` | `4` | Maximum app instances. |
| `AsgDesiredCapacity` | `2` | Desired app instances at launch. |
| `CpuTargetValue` | `50` | Target average CPU utilisation (%) for the scaling policy. |
| `DBInstanceClass` | `db.t3.micro` | RDS instance class. |
| `DBAllocatedStorage` | `20` | RDS storage in GiB. |
| `DBEngineVersion` | `10.6.25` | MariaDB engine version. |
| `DBMultiAZ` | `true` | Whether RDS is Multi-AZ (drives the `IsDBMultiAZ` condition). |
| `DBMasterUsername` | `admin` | RDS master username (also stored as `/cafe/dbUser`). |
| `AppRepoUrl` | `https://github.com/rclc/cafe-app.git` | Git repository the cafe-app is cloned from at boot. |
| `HealthCheckPath` | `/cafe/` | Path the target group health check requests. |

The RDS master password is **not** a parameter — Secrets Manager generates it.

## Outputs

### `lab-network.yaml`

| Output | Description |
|--------|-------------|
| `VPC` | VPC ID (exported as `${StackName}-VPCID`). |
| `PublicSubnet1` | Public subnet 1 ID (exported as `${StackName}-SubnetID`). |
| `PublicSubnet1ID` / `PublicSubnet2ID` | Public subnet IDs. |
| `PrivateSubnet1ID` / `PrivateSubnet2ID` | Private subnet IDs. |

### `lab-application.yaml`

| Output | Description |
|--------|-------------|
| `URL` | cafe-app URL on the load balancer (`http://<alb-dns>/cafe/`). |
| `BastionPublicIP` | Public IP of the bastion host. |
| `DBEndpoint` | RDS instance endpoint address. |
| `AutoScalingGroupName` | Name of the Auto Scaling group. |
| `TargetGroupArn` | ARN of the target group. |

## Deploy

Deploy the **network stack first**, then the application stack. Replace
`<your-ip>` with your own address.

```bash
aws cloudformation deploy \
  --region us-east-1 \
  --stack-name lab-network \
  --template-file lab-network.yaml

aws cloudformation deploy \
  --region us-east-1 \
  --stack-name lab-application \
  --template-file lab-application.yaml \
  --capabilities CAPABILITY_IAM \
  --parameter-overrides \
      NetworkStackName=lab-network \
      MyIP=<your-ip>/32
```

The application stack needs `--capabilities CAPABILITY_IAM` because it
creates an IAM role and instance profile (with generated names).

## Delete

Delete the **application stack first** (it imports the network exports), then
the network stack.

```bash
aws cloudformation delete-stack --region us-east-1 --stack-name lab-application
aws cloudformation wait stack-delete-complete --region us-east-1 --stack-name lab-application

aws cloudformation delete-stack --region us-east-1 --stack-name lab-network
aws cloudformation wait stack-delete-complete --region us-east-1 --stack-name lab-network
```

RDS uses `DeletionPolicy: Delete`, so no final snapshot is left behind.

## Validate

```bash
aws cloudformation validate-template --region us-east-1 --template-body file://lab-network.yaml
aws cloudformation validate-template --region us-east-1 --template-body file://lab-application.yaml
uvx cfn-lint lab-network.yaml lab-application.yaml
```
