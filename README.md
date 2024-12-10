Below are detailed, step-by-step instructions for configuring a load balancer in front of your EC2 instance that is being continuously deployed via a GitHub Actions CI/CD pipeline. The instructions assume you have already:

- Set up an AWS account.
- Created and configured an EC2 instance that is running your application.
- Successfully configured GitHub Actions to deploy code to that EC2 instance (as per your existing pipeline).

**High-Level Overview:**  
1. **Prepare Your EC2 Instance:** Ensure your EC2 instance’s security group and application are correctly set up to receive traffic from a load balancer.  
2. **Create a Target Group:** Define a target group that points to your EC2 instance.  
3. **Create an Elastic Load Balancer (ELB/ALB):** Create and configure an Application Load Balancer (ALB) and attach the target group.  
4. **Update Security Groups and Inbound Rules:** Ensure that the load balancer can forward traffic to your EC2 instance.  
5. **Adjust GitHub Actions Pipeline (If Needed):** Typically, the pipeline remains largely the same, but you may want to use the load balancer endpoint in your tests or notifications.  
6. **Test and Validate:** Confirm the load balancer is correctly routing traffic to your instance.

---

### Prerequisites

- **AWS CLI Installed (optional):** Useful for verifying configurations, but not mandatory.
- **AWS IAM User/Role with Proper Permissions:** You need permissions to create load balancers, target groups, and edit security groups.
- **Your Application Listening on a Known Port:** For example, a Node.js app listening on port 3000 or a web server on port 80/443.

---

### Step-by-Step Instructions

#### Step 1: Confirm Your EC2 Instance Setup

1. **Check Application Port:**  
   Make sure you know which port your application is running on (e.g., port 80 for HTTP, or 8080 if custom).

2. **Check Security Group:**  
   The security group for your EC2 instance should allow inbound traffic on the application’s port from your load balancer.  
   - For now, it can be open to the internet if you’ve been testing directly. Later, you’ll restrict it to only accept traffic from the load balancer.

#### Step 2: Create a Target Group

1. **Navigate to the EC2 Console:**  
   Go to **Services > EC2** in the AWS Management Console.
   
2. **Target Groups Section:**  
   On the left menu, under “Load Balancing,” click on “Target Groups.”

3. **Create Target Group:**  
   - Click **Create target group**.
   - Choose “Instances” as the target type.
   - Give it a descriptive name, e.g., `my-app-tg`.
   - Select a protocol (generally HTTP) and port that matches your application’s listening port.
   - Click **Next**.

4. **Register Targets (Your EC2 Instances):**  
   - On the next screen, select your EC2 instance from the list of available instances.
   - Click **Include as pending below** and then **Create target group**.
   
   After creation, verify that the health checks (default is a simple HTTP 200 check on the specified port/path) are correct. If your app runs on `/`, the default health check path `/` should be fine.

#### Step 3: Create an Application Load Balancer (ALB)

1. **Go to Load Balancers Page:**  
   Under “Load Balancing” in the EC2 console, select “Load Balancers.”

2. **Create Load Balancer:**  
   - Click **Create Load Balancer**.
   - Choose **Application Load Balancer**.
   - Enter a name, e.g., `my-app-alb`.
   - Select **internet-facing** if you want your app accessible to the public internet.
   - Choose the appropriate VPC and subnets (at least two subnets in different Availability Zones are required for redundancy).

3. **Security Groups for the ALB:**  
   - Assign a security group to the ALB that allows inbound traffic on the port you’ll serve. For HTTP, that’s port 80; for HTTPS, port 443.
   - By default, inbound on port 80 is allowed. If not, add a rule to allow inbound TCP:80 from the internet (0.0.0.0/0) or your desired IP ranges.

4. **Listeners and Target Group Association:**  
   - By default, you can add a listener on port 80 (HTTP). You may add HTTPS later if you have SSL certificates.
   - Under “Default actions,” choose “Forward to” and select the target group you created (`my-app-tg`).

5. **Review and Create:**  
   - Click **Create load balancer**.
   - Wait until the ALB’s status is “active” (this can take a few minutes).

#### Step 4: Configure EC2 Security Group to Accept Traffic Only From the ALB

1. **Identify the ALB’s Security Group or CIDR:**  
   Each ALB is assigned a set of IP addresses. The best practice is to allow inbound traffic from the ALB’s security group rather than from any IP.  
   - Go to the ALB details page and note the security group assigned to it.

2. **Modify the EC2 Instance’s Security Group:**  
   - Navigate to the EC2 console, and go to “Instances.”
   - Select your instance and click on the Security tab.
   - Click the security group to modify it.
   - Remove the rule that allows general inbound traffic on the application port.
   - Add a new inbound rule that allows inbound traffic from the ALB’s security group on the application port (e.g., port 80).
   
   This ensures that only traffic routed through the ALB reaches your instance.

#### Step 5: Update Your GitHub Actions Pipeline (If Necessary)

In most CI/CD pipelines, the deployment step is typically something like an SSH command, a script to `scp` files, or running a configuration management tool (Ansible, Terraform, etc.) to update the code on the EC2 instance. The existence of the load balancer doesn’t necessarily change your deployment steps, as you’re still deploying to the EC2 instance directly.

**Optional Adjustments:**

- If you perform post-deployment tests (like integration tests) in your pipeline, you may now want to point those tests to the load balancer’s DNS name instead of the EC2’s public IP. This tests the real production environment end-to-end.
- Update environment variables in your workflow to store the ALB’s DNS name if needed.  
  - For example, under your repository’s “Settings > Secrets and variables > Actions,” you might add `LOAD_BALANCER_DNS` and set it to the ALB’s DNS name.  
  - Use this variable in your tests or notifications.

Your `.github/workflows/ci-cd.yml` might look like this (example snippet):

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v2

      - name: Deploy to EC2
        run: |
          ssh -i ${{ secrets.EC2_KEY }} ec2-user@${{ secrets.EC2_PUBLIC_IP }} "cd /var/www/app && git pull && npm install && pm2 restart all"

      - name: Test via ALB
        run: |
          curl -I http://${{ secrets.LOAD_BALANCER_DNS }}
```

This way, after deployment, you’re testing your live environment.

#### Step 6: Testing the Setup

1. **Check ALB Health Checks:**  
   - Go to the Target Group page in the AWS console.
   - Ensure the instance shows as “healthy.” If not, troubleshoot by checking security groups, and whether the application responds to the health check path.

2. **Access via ALB DNS Name:**  
   - The ALB provides a DNS name (e.g., `my-app-alb-1234567890.us-east-1.elb.amazonaws.com`).
   - Put this DNS name into your browser (or use `curl`) to see if your application responds.
   - If it works, congratulations—you’re now serving your application behind a load balancer!

3. **(Optional) Set a Custom Domain and SSL:**  
   - Create or update a CNAME record in your DNS provider to point your domain (e.g., `app.example.com`) to the ALB’s DNS name.
   - Configure HTTPS using AWS Certificate Manager (ACM) and update the ALB’s listener to use the certificate.

---

### Additional Considerations

- **Scalability:** With a load balancer, you can later add auto-scaling groups to launch multiple EC2 instances behind the ALB for higher availability and scalability.
- **Monitoring and Logging:** Use AWS CloudWatch and AWS ELB access logs to monitor the health and performance of your application.
- **Cost Management:** ALBs have a cost associated. Make sure to budget accordingly.

---

**Conclusion:**  
By following the above steps, you’ll have your CI/CD pipeline continuing to deploy updates directly to the EC2 instance, but now all end-user traffic will be routed through a properly configured load balancer. This provides better fault tolerance, easier scaling, and a single stable endpoint for accessing your application.
