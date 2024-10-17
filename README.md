To execute a script on an EC2 instance without using AWS Systems Manager (SSM), you'll need to directly SSH into the instance and run the script. Below is an updated version of the GitHub Action workflow that uses SSH to run the script on the EC2 instance after it's launched, and then shuts it down.

### Requirements:
1. **An existing EC2 Key Pair** to SSH into the instance.
2. **Your SSH Private Key** stored as a GitHub secret.
3. **EC2 Security Group** should allow inbound SSH access (port 22) from GitHub Actions IP addresses.

### 1. **Store SSH Private Key in GitHub Secrets**
First, you need to add your private SSH key to GitHub secrets:
- Navigate to **Settings** > **Secrets and Variables** > **Actions** in your GitHub repository.
- Add the secret: `EC2_SSH_KEY`
- Add the secret: `AWS_ACCESS_KEY_ID`
- Add the secret: `AWS_SECRET_ACCESS_KEY`

### 2. **Create the GitHub Action Workflow**

Here's the GitHub Action workflow `.github/workflows/launch-ec2-instance.yml` that launches an EC2 instance, connects via SSH, runs a script, and then terminates the instance:

```yaml
name: Launch EC2, Run Script via SSH, and Terminate
on:
  workflow_dispatch:  # Allows manual triggering of the workflow

jobs:
  launch-run-terminate:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout repository
      uses: actions/checkout@v3

    - name: Set up AWS CLI
      uses: aws-actions/configure-aws-credentials@v2
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: 'us-east-2'  # Change this to your preferred region

    - name: Launch EC2 instance
      id: launch_ec2
      run: |
        INSTANCE_ID=$(aws ec2 run-instances \
          --image-id <AMI_ID> \
          --instance-type <INSTANCE_TYPE> \
          --subnet-id <SUBNET_GROUP> \
          --security-group-ids <SECURITY_GROUP_ID> \
          --key-name <KEY_PAIR_NAME> \
          --associate-public-ip-address
          --query 'Instances[0].InstanceId' \
          --output text)
        echo "INSTANCE_ID=$INSTANCE_ID" >> $GITHUB_ENV

    - name: Wait for EC2 instance to be in running state
      run: |
        aws ec2 wait instance-running --instance-ids ${{ env.INSTANCE_ID }}

    - name: Get EC2 instance public IP
      run: |
        PUBLIC_IP=$(aws ec2 describe-instances \
          --instance-ids ${{ env.INSTANCE_ID }} \
          --query 'Reservations[0].Instances[0].PublicIpAddress' \
          --output text)
        echo "PUBLIC_IP=$PUBLIC_IP" >> $GITHUB_ENV

    - name: Add SSH key to SSH agent
      uses: webfactory/ssh-agent@v0.7.0
      with:
        ssh-private-key: ${{ secrets.EC2_SSH_KEY }}

    - name: Wait for SSH availability
      run: |
        echo "Waiting for SSH to become available..."
        for i in {1..15}; do
          if ssh -o StrictHostKeyChecking=no -i ~/.ssh/id_rsa ubuntu@${{ env.PUBLIC_IP }} "echo SSH ready"; then
            break
          else
            echo "SSH not ready, retrying in 10 seconds..."
            sleep 10
          fi
        done

    - name: Run script on EC2 instance via SSH
      run: |
        ssh -o StrictHostKeyChecking=no -i ~/.ssh/id_rsa ubuntu@${{ env.PUBLIC_IP }} << 'EOF'
        # Your script goes here. Replace the echo with actual commands.
        echo "Running your script on the EC2 instance"
        # Example script:
        sudo apt update -y
        sudo apt-get update -y
        sudo apt-get install -y nginx
        sudo systemctl start nginx
        curl localhost:80
        EOF

    - name: Terminate EC2 instance
      run: |
        aws ec2 terminate-instances --instance-ids ${{ env.INSTANCE_ID }}
        aws ec2 wait instance-terminated --instance-ids ${{ env.INSTANCE_ID }}

```

### Breakdown of the Workflow:
1. **Launch EC2 Instance**: 
   - The EC2 instance is launched in a specific VPC subnet, and the security group and key pair are specified. You'll need to replace the placeholders (AMI ID, subnet, security group, and key name) with your actual values.

2. **Wait for SSH Availability**: 
   - The workflow waits for the instance's SSH port (22) to become available. This is done using a loop that attempts to SSH into the instance multiple times with a delay.

3. **Run Script on EC2 via SSH**: 
   - After SSH is available, the workflow SSHs into the instance using the key pair and runs the script. Replace the `echo` and sample commands in the `ssh` block with your actual script commands.

4. **Terminate EC2 Instance**: 
   - Once the script has completed, the instance is terminated.

### Notes:
- The SSH username (`ec2-user`) may vary depending on the AMI you use. For Amazon Linux, it's typically `ec2-user`, for Ubuntu it's `ubuntu`, etc.
- Ensure that your EC2 security group allows inbound SSH traffic on port 22 from GitHub Actions' IP addresses, or temporarily open it to all IPs (`0.0.0.0/0`) during testing (though this is not recommended for production).
- Replace the AMI_ID, INSTANCE_TYPE, SUBNET_GROUP, SECURITY_GROUP_ID and KEY_PAIR_NAME with valid values for your AWS environment.