# Minecraft Server on AWS (CloudFormation)

A single CloudFormation template that launches an Ubuntu EC2 instance running
[PufferPanel](https://www.pufferpanel.com/) in Docker, so you can host Minecraft (Java and Bedrock) servers
from a web panel. Server and panel data live on EFS, so they survive instance replacement.

Architecture diagram: [`Documentation/Minecraft Server.drawio`](Documentation/Minecraft%20Server.drawio)
(open it with [diagrams.net](https://app.diagrams.net)).

## What gets created

| Resource | Purpose |
| --- | --- |
| `ServerVpc`, `PublicSubnet`, `PrivateSubnet` | `192.168.160.0/24` VPC with a public `/25` (internet route) and a private `/25` (no internet route, EFS only) |
| `InternetGateway`, route tables | Internet access for the public subnet |
| `ServerSecurityGroup` | Minecraft ports, PufferPanel, SFTP, SSH and ping, each restricted to the CIDRs you pass in |
| `ServerLaunchTemplate` | Instance type, Ubuntu AMI, encrypted 10 GB gp3 root volume, key pair, IAM role, user data |
| `ServerInstanceAmd` | The EC2 instance, launched from the template's latest version |
| `ServerEip` | Elastic IP, the stable public address |
| `PufferPanelServersFileSystem`, `PufferPanelConfigFileSystem` | Encrypted EFS volumes for game server data and PufferPanel config |
| `StopEC2Schedule`, `StartEC2Schedule` | EventBridge Scheduler jobs that stop (00:00 UTC) and start (08:00 UTC) the instance |
| `BackupVault`, `DailyBackupPlan`, `DailyBackupSelection` | Daily instance backups at midnight America/New_York, kept 3 days |

Ports:
- Minecraft Bedrock 19132 and Java 25565 (UDP and TCP): open to `MinecraftUsersCidr`.
- SSH 22, PufferPanel web UI 8080, SFTP 5657 and ICMP echo: open only to `MyIpCidr`.

## Parameters

| Parameter | Default | Notes |
| --- | --- | --- |
| `AwsRegion` | `us-east-1` | Must match the `--region` used for deploy. A stack rule enforces this. |
| `MyIpCidr` | none (required) | Your public IP in CIDR form, e.g. `203.0.113.10/32` |
| `MinecraftUsersCidr` | none (required) | Players' CIDR, e.g. `0.0.0.0/0` for anyone |
| `Memory` | `2GB` | `2GB` uses `t3a.small`, `4GB` uses `t3a.medium`. Changing it replaces the instance. |
| `UbuntuAmiId` | Ubuntu 26.04 LTS (SSM parameter) | Resolves the amd64 AMI. Change the version in the path to use another release. |
| `KeyName` | empty | Optional EC2 key pair. Leave empty to use SSM Session Manager only. |

## Prerequisites

- An AWS CLI profile named `minecraft` with permission to create the resources above
- Optional: an EC2 key pair for SSH. Create one once:

```sh
aws ec2 create-key-pair --key-name minecraft-keypair --key-type rsa --key-format pem \
  --region us-east-1 --profile minecraft --query KeyMaterial --output text > minecraft-keypair.pem
chmod 400 minecraft-keypair.pem
```

Pass `KeyName=minecraft-keypair` at deploy time to use it. Keep `*.pem` files out of git (already in `.gitignore`).

## Deploy

Validate the template:

```sh
aws cloudformation validate-template --template-body file://pufferpanel-cloudformation.yaml \
  --profile minecraft --region us-east-1
```

Get the address from the stack outputs (`PublicIp`, `PufferPanelUrl`, `InstanceId`):

```sh
aws cloudformation describe-stacks --stack-name pufferpanel-on-ec2-docker \
  --profile minecraft --region us-east-1 --query 'Stacks[0].Outputs'
```

The public address stays the same unless the stack is deleted and recreated. If your public IP changes,
redeploy with the new `MyIpCidr`.

## After deploy

Setup runs in the background on first boot. Allow a few minutes, then open `PufferPanelUrl`. No admin account
is created by the stack, so create the first user on the instance (see below) before logging in.

To check progress or connect:

```sh
# SSH (requires KeyName)
ssh -i minecraft-keypair.pem ubuntu@<public-ip>

# Or SSM Session Manager (works without a key pair)
aws ssm start-session --target <instance-id> --profile minecraft --region us-east-1

# Bootstrap log
sudo cat /var/log/cloud-init-output.log

# PufferPanel container
sudo docker logs pufferpanel
sudo docker exec -it pufferpanel /bin/sh

# Create a PufferPanel admin user (the stack no longer creates one)
sudo docker exec -it pufferpanel /pufferpanel/bin/pufferpanel user add --name <name> --email <email> --password <password> --admin
```

## Operating notes

- **Daily stop/start**: the instance is off between 00:00 and 08:00 UTC by design. Both schedules are created
  in the `DISABLED` state, so enable `StopEC2Schedule` and `StartEC2Schedule` in EventBridge Scheduler if you
  want the automatic shutdown.
- **Data persistence**: game servers and PufferPanel config are on EFS, so they persist across stop/start and
  instance replacement. The EC2 root volume does not hold any server data.
- **Backups**: `DailyBackupPlan` backs up the EC2 instance (root volume) only. The EFS file systems are not in
  the backup selection.
- **Instance replacement**: changing user data, the launch template, `Memory` or the AMI creates a new
  instance. The root volume is lost, and the Elastic IP and instance ID change. EFS data is kept.
- **Region**: `AwsRegion` must match the deploy region.
- **Ubuntu release**: the default AMI is Ubuntu 26.04 LTS. Earlier versions of this README referred to 24.04.

## Repository layout

- `pufferpanel-cloudformation.yaml`: the template (the only deployable artifact)
- `Documentation/`: architecture diagram (`Minecraft Server.drawio`)
- `CLAUDE.md`: guidance for Claude Code working in this repo
- `LICENSE`: project license
