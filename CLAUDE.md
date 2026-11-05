# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

# AWS
use only AWS profile: minecraft

## What this repo is

An AWS SAM/CloudFormation stack that launches one EC2 instance (`t3a.medium`, Ubuntu amd64) running
PufferPanel in Docker, for hosting a Minecraft server. There is no application code, no build step,
and no test suite. The "code" is `template.yaml` (infrastructure) and `user-data.yaml` (first-boot
bootstrap, pulled into the launch template via `AWS::Include`).

## Commands

```sh
# Validate and lint the template (the only verification available)
sam validate --lint

# Deploy (profile minecraft, us-east-1)
sam deploy --profile minecraft --stack-name pufferpanel-on-ec2-docker --region us-east-1 \
  --resolve-s3 --capabilities CAPABILITY_IAM \
  --parameter-overrides MyIpCidr=<your-public-ip>/32

# Connect to the instance
ssh -i keypair ubuntu@<public-ip>
aws ssm start-session --target <instance-id> --profile minecraft --region us-east-1

# Create the first PufferPanel admin (run on the instance)
sudo docker exec -it pufferpanel /pufferpanel/pufferpanel user add
```

Debugging on the instance: `/var/log/cloud-init-output.log` (bootstrap progress) and
`sudo docker logs pufferpanel`.

## Architecture

- **Network**: a dedicated VPC (`192.168.160.0/24`) with one public subnet, an internet gateway, and
  a default route. The README says it uses the default VPC; the template does not. Trust the template.
- **Security group** (`ServerSecurityGroup`): Minecraft Bedrock UDP/TCP on 19132 and 19133 and PufferPanel
  SFTP on 5657 are open to `0.0.0.0/0`. SSH (22) and the PufferPanel web UI (8080) are restricted to the
  `MyIpCidr` parameter. ICMP echo is open.
- **Compute**: `ServerLaunchTemplate` holds the instance config (AMI from an SSM parameter, encrypted 20 GB
  gp3 root volume, IAM instance profile with `AmazonSSMManagedInstanceCore`, key pair, user data).
  `ServerInstanceAmd` is launched from the template's `LatestVersionNumber`, so a template change creates a new version.
- **Bootstrap**: `user-data.yaml` installs `docker.io`, creates the `pufferpanel-config` and
  `pufferpanel-servers` volumes, and runs `pufferpanel/pufferpanel:latest` with the Docker socket mounted so
  PufferPanel can start game-server containers. It runs only on first boot of an instance.
- **Schedule**: EventBridge Scheduler (`StopEC2Schedule` at 00:00 UTC, `StartEC2Schedule` at 08:00 UTC)
  stops and starts the instance daily through `SchedulerExecutionRole`. The instance is off overnight by design.
- **Stable address**: `ServerEip` attaches an Elastic IP. It changes only if the stack is deleted and recreated.

## Constraints when editing

- Changing user data creates a new launch template version. The instance may be replaced, which deletes the
  root volume and all server data and changes the Elastic IP. Warn the user before doing this.
- Changing the instance architecture or the AMI replaces the instance. Architecture changes also require
  renaming the instance resource, because CloudFormation cannot change the type in place.
- `AWS::SSM::Parameter::Value` for the AMI pins Ubuntu 26.04 by default. The README says 24.04. Check which
  one the user wants before changing it.
- `MyIpCidr` has no default. Every deploy must pass `--parameter-overrides MyIpCidr=<ip>/32`, or SAM will prompt for it.

## Secrets

- The `keypair` private key lives in the repo root and is not git-ignored yet (there is no `.gitignore`). Do not commit it.
