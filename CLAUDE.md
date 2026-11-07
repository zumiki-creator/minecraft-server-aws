# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## AWS

Use only the AWS CLI profile `minecraft`. Region is `us-east-1`.

## What this repo is

A single standalone CloudFormation template, `pufferpanel-cloudformation.yaml`, that launches one EC2
instance (`t3a.small` or `t3a.medium`, chosen by the `Memory` parameter, Ubuntu amd64) running PufferPanel
in Docker to host a Minecraft server. There is no build step, no application code, and no test suite.
The template is the only artifact. Verification is linting/validation plus a deploy.

## Commands

```sh
# Validate (the only offline check available)
aws cloudformation validate-template --template-body file://pufferpanel-cloudformation.yaml --profile minecraft --region us-east-1

```

On the instance:
- Bootstrap progress: `sudo cat /var/log/cloud-init-output.log`
- PufferPanel container: `sudo docker logs pufferpanel`, `sudo docker exec -it pufferpanel /bin/sh`
- Extra admin: `sudo docker exec -it pufferpanel /pufferpanel/bin/pufferpanel user add`

## Architecture

- **Network**: one VPC (`192.168.160.0/24`) with two `/25` subnets in the same AZ. `PublicSubnet` has an
  internet gateway route and auto-assigns public IPs. `PrivateSubnet` has no default route and hosts only the
  EFS mount targets.
- **Security groups**: `ServerSecurityGroup` opens Minecraft ports (Bedrock 19132, Java 25565; UDP and TCP) to
  `MinecraftUsersCidr`. SSH, PufferPanel UI (8080), SFTP (5657) and ICMP are restricted to `MyIpCidr`.
  `EfsSecurityGroup` allows NFS (2049) only from `ServerSecurityGroup`.
- **Compute**: `ServerLaunchTemplate` holds the instance config (SSM-resolved AMI, encrypted gp3 root volume,
  SSM instance role, optional key pair, user data). `ServerInstanceAmd` launches from the template's
  `LatestVersionNumber`, so a template change creates a new version and can replace the instance.
- **Persistent data**: two EFS file systems, `PufferPanelServersFileSystem` (mounted at
  `/mnt/pufferpanel-servers`, backs `/var/lib/pufferpanel`) and `PufferPanelConfigFileSystem` (mounted at
  `/mnt/pufferpanel-config`, backs `/etc/pufferpanel`). Game and panel data survive instance replacement.
  The `ServerInstanceAmd` resource `DependsOn` both mount targets.
- **Bootstrap**: the `UserData` block in the launch template (inline `Fn::Sub`, not a separate file) installs
  `docker.io` and `nfs-common`, mounts both EFS volumes via `/etc/fstab`, runs `pufferpanel/pufferpanel:latest`
  with the Docker socket mounted (so the panel can start game-server containers), and creates the first admin
  with `pufferpanel user add`. It runs only on first boot of an instance.
- **Schedule**: `StopEC2Schedule` (00:00 UTC) and `StartEC2Schedule` (08:00 UTC) stop and start the instance
  through `SchedulerExecutionRole`. Both are `State: DISABLED` in the template, so the daily stop/start does not
  run until they are enabled.
- **Backups**: AWS Backup plan `DailyBackupPlan` takes daily snapshots of the instance at midnight America/New_York
  into `BackupVault`, retained 3 days. Selection covers only the instance (root volume), not the EFS file systems.
- **Stable address**: `ServerEip` is attached to the instance. It changes only if the stack is deleted and recreated.

## Constraints when editing

- Changing user data or the launch template creates a new version, which can replace the instance. The root
  volume is lost, but EFS data persists. The Elastic IP and instance ID change on replacement.
- Changing `Memory` or the AMI replaces the instance.
- `AwsRegion` must equal the `--region` used for deploy. The `RegionMatchesStackRegion` rule enforces this.
- `UbuntuAmiId` defaults to the Ubuntu 26.04 SSM parameter. The README was written for 24.04 and is out of date;
  confirm the target release before changing it.
- The `AdminPassword` value is interpolated into the user-data script. Do not add `set -x` or other tracing to
  the script, because it would write the password to cloud-init logs. The template comment already warns about this.
- Parameter `AllowedPattern`s are strict (CIDR format, admin password character set). Keep them consistent with the
  documentation when changing them.

