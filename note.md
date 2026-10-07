# Automate EC2 Provisioning and Deployment

This note continues from **point 2: Prepare AWS**. It assumes the project has been copied to its own folder, initialized as a Git repository, and connected to the remote repository. Do not push until the repository settings and workflow configuration are ready.

The existing `.github/workflows/deploy.yml` builds a Docker image, transfers it to EC2 over SSH, and runs it. It currently deploys to the fixed `EC2_INSTANCE_PUBLIC_IP` repository variable. The steps below explain how to find or create an instance tagged `Frontend` and pass its IP to the existing deployment job instead.

## 2. Prepare AWS

1. Choose the AWS region where the instance will run. Use the same region in the workflow settings.
2. In that region, identify or create an EC2 AMI. The instance launched from it must have Docker installed and running before deployment. You can bake Docker into the AMI or install it from a tested startup script.
3. Create or select an EC2 key pair. The instance uses the key pair's public key; GitHub Actions will use the matching private key for SSH.
4. Choose a subnet in the intended VPC. Make sure it assigns a public IPv4 address to launched instances, or otherwise provides an address reachable by the GitHub runner.
5. Choose a security group in that same VPC. Allow inbound TCP port `22` (SSH) from the workflow runner, and allow inbound TCP port `80` (or your chosen app port) from the intended web users. Do not open SSH to `0.0.0.0/0` unless this is a temporary, controlled exercise. GitHub-hosted runner egress IPs can change; a self-hosted runner with a stable egress IP can make the SSH rule narrower.
6. Create an IAM identity for GitHub Actions with only the EC2 permissions it needs. It needs to describe instances, launch instances, and create the required instance tags. If using the existing access-key configuration, store the access key and secret key as GitHub Actions secrets. Prefer configuring GitHub Actions OIDC with an IAM role instead of long-lived access keys when you are ready to set that up.
7. Confirm that the AMI, key pair, subnet, and security group are all in the selected region, and that the subnet and security group belong to the same VPC. `VPC_ID` is useful for checking your configuration; `run-instances` uses the subnet and security group IDs to select the network.

## 3. Configure GitHub Actions values

Open the GitHub repository, then go to **Settings → Secrets and variables → Actions**. Create these **repository variables**:

| Variable | Example or purpose |
| --- | --- |
| `AWS_REGION` | Region containing the AMI and network, such as `us-east-1` |
| `TAG_NAME` | `Frontend` |
| `INSTANCE_TYPE` | `t3.micro` |
| `AMI_ID` | The ID of the prepared AMI |
| `KEY_PAIR_NAME` | EC2 key pair name, not the private key |
| `VPC_ID` | VPC ID; use it to verify the subnet and security group are correct |
| `SUBNET_ID` | Subnet in which to launch the instance |
| `SECGRP_ID` | Security group ID for the instance |
| `APP_PORT` | `80`, or another allowed host port |
| `EC2_SSH_USER` | AMI's SSH user, for example `ec2-user` or `ubuntu` |

Create these **repository secrets** if using the current workflow's access-key authentication:

| Secret | Purpose |
| --- | --- |
| `AWS_ACCESS_KEY_ID` | IAM access key used by the workflow |
| `AWS_SECRET_ACCESS_KEY` | Matching IAM secret key |
| `SSH_KEY` | Private key for the EC2 key pair; paste the full private key contents |

Never put credentials or a private key in a repository variable, workflow YAML, or committed file. If the workflow uses OIDC instead, configure the AWS credentials step for the IAM role and remove the access-key secrets after verifying OIDC works.

## 4. Add the provisioning job

Edit `.github/workflows/deploy.yml`. Add a `provision` job under `jobs:` alongside `build` and `deploy`. Configure AWS credentials in this job: each GitHub Actions job runs on a separate runner and does not inherit another job's AWS credentials.

The job should:

1. Search for an instance tagged `Name=Frontend` in `pending` or `running` state.
2. Reuse the matching instance if found. This simple approach assumes there is at most one active instance with that tag.
3. If none is found, launch one with the configured AMI, type, key pair, subnet, security group, and tag.
4. Wait for the instance to reach `running`, then retrieve its public IP.
5. Fail if the IP is empty or `None`.
6. Publish the IP as a job output so the deployment job can consume it.

Use this as a starting point for the job (keep the indentation under `jobs:`):

```yaml
  provision:
    runs-on: ubuntu-latest
    outputs:
      public_ip: ${{ steps.address.outputs.public_ip }}
    steps:
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ vars.AWS_REGION }}

      - name: Find or launch instance
        id: instance
        env:
          TAG_NAME: ${{ vars.TAG_NAME }}
          AMI_ID: ${{ vars.AMI_ID }}
          INSTANCE_TYPE: ${{ vars.INSTANCE_TYPE }}
          KEY_PAIR_NAME: ${{ vars.KEY_PAIR_NAME }}
          SUBNET_ID: ${{ vars.SUBNET_ID }}
          SECGRP_ID: ${{ vars.SECGRP_ID }}
        run: |
          set -euo pipefail
          instance_id="$(aws ec2 describe-instances \
            --filters "Name=tag:Name,Values=${TAG_NAME}" \
                      "Name=instance-state-name,Values=pending,running" \
            --query 'Reservations[].Instances[].InstanceId | [0]' \
            --output text)"

          if [[ -z "$instance_id" || "$instance_id" == "None" ]]; then
            instance_id="$(aws ec2 run-instances \
              --image-id "$AMI_ID" \
              --instance-type "$INSTANCE_TYPE" \
              --key-name "$KEY_PAIR_NAME" \
              --subnet-id "$SUBNET_ID" \
              --security-group-ids "$SECGRP_ID" \
              --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=${TAG_NAME}}]" \
              --query 'Instances[0].InstanceId' \
              --output text)"
          fi

          [[ "$instance_id" == i-* ]] || { echo "Could not find or launch an EC2 instance." >&2; exit 1; }
          echo "instance_id=$instance_id" >> "$GITHUB_OUTPUT"

      - name: Wait for instance and get public IP
        id: address
        env:
          INSTANCE_ID: ${{ steps.instance.outputs.instance_id }}
        run: |
          set -euo pipefail
          aws ec2 wait instance-running --instance-ids "$INSTANCE_ID"
          public_ip="$(aws ec2 describe-instances \
            --instance-ids "$INSTANCE_ID" \
            --query 'Reservations[0].Instances[0].PublicIpAddress' \
            --output text)"
          [[ -n "$public_ip" && "$public_ip" != "None" ]] || {
            echo "The instance has no public IP. Check the subnet's public IP assignment and routing." >&2
            exit 1
          }
          echo "public_ip=$public_ip" >> "$GITHUB_OUTPUT"
```

The `aws ec2 wait instance-running` command waits for the EC2 state to become `running`; it does not mean SSH or Docker is ready. A stopped instance is not reused by the query above, so this version would launch another instance if only a stopped tagged instance exists. Decide whether to start stopped instances or terminate them as a separate, deliberate improvement.

## 5. Send the discovered IP to deployment

In the existing `deploy` job, make the job depend on both the build and provisioning jobs:

```yaml
  deploy:
    needs: [build, provision]
```

Replace each `${{ vars.EC2_INSTANCE_PUBLIC_IP }}` reference used by the deploy job with:

```yaml
${{ needs.provision.outputs.public_ip }}
```

For example, set the environment for the existing **Validate deployment settings**, **Verify target instance**, and **Prepare verified SSH connection** steps to use the job output. Keep the existing artifact download, SCP, and remote Docker run steps. Remove the `EC2_INSTANCE_PUBLIC_IP` repository variable once the updated workflow no longer uses it.

The existing deployment job maps the EC2 host port in `APP_PORT` to container port `80`. The security group must allow inbound web traffic on that host port.

## 6. Wait for SSH and fail on timeout

In the existing `Prepare verified SSH connection` step, create the SSH key/configuration as it already does. Then add a readiness step immediately after it and before **Copy image to EC2**. It should use the same SSH config and retry for a fixed period, such as 30 attempts with a 10-second pause (about five minutes):

```yaml
      - name: Wait for SSH
        run: |
          set -euo pipefail
          for attempt in {1..30}; do
            if ssh -F "$RUNNER_TEMP/ec2_ssh_config" \
              -o ConnectTimeout=5 ec2-deploy true; then
              echo "SSH is ready."
              exit 0
            fi
            if [[ "$attempt" -lt 30 ]]; then
              sleep 10
            fi
          done
          echo "SSH did not become available before the timeout." >&2
          exit 1
```

If this step fails, the deployment stops. Check the instance state, public IP, route table, security-group SSH source, key pair/private key, and `EC2_SSH_USER`. If SSH works but Docker commands fail, check that Docker is installed, running, and usable by the SSH user. The current workflow disables SSH host-key checking; this is convenient for an exercise, but pinning and verifying the host key is safer for a production deployment.

## 7. Build, test, and push

Before pushing, run the project's local checks from the repository root:

```sh
npm ci
npm run build
docker compose up --build
```

Open `http://localhost:8080` and confirm the app loads. Stop the local Compose service with `Ctrl+C` when finished.

Review `.github/workflows/deploy.yml` and confirm that:

- The provisioning job uses the correct AWS region and variables.
- The deployment job has `needs: [build, provision]` and uses the provisioned IP output everywhere it previously used the fixed IP variable.
- The repository has the needed variables and secrets configured.
- The IAM permissions, subnet, security group, AMI, key pair, and SSH username are correct.
- No credentials or private keys are committed.

When the configuration is ready, push the branch to GitHub. The existing workflow runs on pushes to `main` and can also be started manually from the repository's **Actions** tab. Follow the run through build, provisioning, SSH readiness, and deployment. Then open `http://<instance-public-ip>/` to verify the deployed app.

An instance's public IPv4 address may change after a stop/start. Looking up the instance by tag on each run avoids keeping a stale address in GitHub settings. If a stable address is required, associate an Elastic IP and account for its cost and lifecycle.