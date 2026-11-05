# Minecraft on AWS (SAM)

Launches a `t3a.medium` Ubuntu (amd64) instance from a launch template in its own VPC. The instance has a
20 GB encrypted gp3 root volume and an IAM instance profile with SSM access. User data installs
Docker and runs `pufferpanel/pufferpanel:latest` in a container. An Elastic IP is attached. The instance
is stopped at 00:00 UTC and started at 08:00 UTC every day.

Resources:
- `ServerVpc`, `PublicSubnet`, `InternetGateway`, `PublicRouteTable`: a `192.168.160.0/24` VPC with one
  public subnet and a default route to the internet.
- `ServerLaunchTemplate` (`AWS::EC2::LaunchTemplate`): instance type, AMI, key pair, instance
  profile, security group, root volume, instance tag `minecraft-pufferpanel-docker`, and user data.
- `ServerInstanceAmd` (`AWS::EC2::Instance`): references the launch template's latest version.
- `ServerEip` (`AWS::EC2::EIP`): the public address.
- `StopEC2Schedule` / `StartEC2Schedule` (`AWS::Scheduler::Schedule`): stop and start the instance daily
  through `SchedulerExecutionRole`.

Security group: every ingress rule (19132 and 19133 UDP + TCP, 5657, 22, 8080, ICMP echo) is restricted
to the `MyIpCidr` parameter.

`MyIpCidr` has no default. Pass your public IP in CIDR form at deploy time (e.g. `203.0.113.10/32`).

## Deployment

- AWS profile `minecraft`
- Region: `us-east-1`. The `AwsRegion` parameter defaults to this and must match the `--region` used for deploy; the stack fails if they differ.
- Stack name: `pufferpanel-on-ec2-docker`
- Key pair: `minecraft-keypair` (private key saved locally as `minecraft-keypair.pem`, read-only, git-ignored)

Create the key pair (run once, before deploying):

```sh
aws ec2 create-key-pair --key-name minecraft-keypair --key-type rsa --key-format pem \
  --region us-east-1 --profile minecraft --query KeyMaterial --output text > minecraft-keypair.pem
chmod 400 minecraft-keypair.pem
```

```sh
sam validate --lint
sam deploy --profile minecraft --stack-name pufferpanel-on-ec2-docker --region us-east-1 \
  --resolve-s3 --capabilities CAPABILITY_IAM \
  --parameter-overrides MyIpCidr=<your-public-ip>/32
```

Get the public address from the stack's `PublicIp` output. It stays the same unless the stack is deleted and recreated.

## Container setup

User data runs these commands on first boot:

```sh
docker volume create pufferpanel-config
docker volume create pufferpanel-servers
docker create --name pufferpanel \
  -p 8080:8080 -p 5657:5657 \
  -v pufferpanel-config:/etc/pufferpanel \
  -v pufferpanel-servers:/var/lib/pufferpanel:z \
  -v /var/run/docker.sock:/var/run/docker.sock \
  --restart=on-failure \
  pufferpanel/pufferpanel:latest
docker start pufferpanel
```

The Docker socket is mounted so the panel can start game server containers.

## After deploy

No user is created automatically. Connect with SSH and create an admin user inside the container:

```sh
ssh -i keypair ubuntu@<public-ip>
sudo docker exec -it pufferpanel /pufferpanel/pufferpanel user add
```

Then open the PufferPanel URL. Setup runs in the background after boot, so allow a few minutes.
Progress: `/var/log/cloud-init-output.log`. Container logs: `sudo docker logs pufferpanel`.

SSM also works: `aws ssm start-session --target <instance-id> --profile minecraft --region us-east-1`.

## Notes

- Default AMI is Ubuntu 26.04 LTS, resolved from an SSM parameter (`UbuntuAmiId`).
- If your public IP changes, redeploy with the new `MyIpCidr` value.
- User data runs only on first boot. Changing it creates a new launch template version, and the
  instance may be replaced, which deletes its root volume and data and changes the Elastic IP.
- Changing the instance type's architecture or the AMI replaces the instance and its root
  volume, so data on it is lost. Changing architecture also requires renaming the instance
  resource, because CloudFormation can't change the type in place on the old architecture.
- The instance is off between 00:00 and 08:00 UTC by design. Server data persists across stop/start, since only the
  root volume is tied to the instance lifecycle.
- Creates its own VPC; it does not use the default VPC.

# Deployment Command
sam deploy --profile minecraft --stack-name pufferpanel-on-ec2-docker --region us-east-1 --resolve-s3 --capabilities CAPABILITY_IAM --no-confirm-changeset --parameter-overrides MyIpCidr=<your ip> AwsRegion=us-east-1

#ssh to ec2:
chmod 400 "minecraft-keypair.pem"
- check the connect comand on aws console:
ssh -i "minecraft-keypair.pem" ubuntu@ip

# Add user to pufferpanel
sudo docker exec -it pufferpanel /pufferpanel/bin/pufferpanel user add

# access to pufferpanel cli
sudo docker exec -it pufferpanel /bin/sh
