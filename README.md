To securely use your SSH private key in GitHub Actions, you need to add it as a **GitHub Secret** and then access it within your workflow. Here's a step-by-step guide to set up and use `SSH_KEY`:

---

### **Step 1: Generate an SSH Key (if you don't have one)**
1. Open a terminal on your local machine.
2. Run the following command to generate an SSH key pair:
   ```bash
   ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
   ```
   - Press `Enter` to save the key to the default location (`~/.ssh/id_rsa`).
   - Provide a passphrase if desired (optional, for added security).

3. Locate the private key (`id_rsa`) and public key (`id_rsa.pub`) in the `.ssh` directory:
   ```bash
   ls ~/.ssh/
   ```
4. Copy the **public key** (`id_rsa.pub`) to your remote server:
   ```bash
   ssh-copy-id -i ~/.ssh/id_rsa.pub username@your-server-ip
   ```
   - This adds the public key to the `~/.ssh/authorized_keys` file on your server.

---

### **Step 2: Add the Private Key as a GitHub Secret**
1. Go to your GitHub repository.
2. Click on **Settings** → **Secrets and variables** → **Actions**.
3. Click **New repository secret**.
4. Add a new secret:
   - **Name**: `SSH_KEY`
   - **Value**: Paste the contents of your private key (`id_rsa`):
     ```bash
     cat ~/.ssh/id_rsa
     ```
   - Make sure not to include extra spaces or newlines.

---

### **Step 3: Use the Secret in Your Workflow**
In your GitHub Actions workflow, access the `SSH_KEY` secret and set up the SSH key.

#### Example:
```yaml
- name: Setup SSH Key
  run: |
    mkdir -p ~/.ssh
    echo "${{ secrets.SSH_KEY }}" > ~/.ssh/id_rsa
    chmod 600 ~/.ssh/id_rsa
```

---

### **Step 4: Add the Host to Known Hosts**
To prevent GitHub Actions from prompting for host verification, add your server to the `known_hosts` file. You can use `ssh-keyscan` to fetch the server's public key and add it.

#### Example:
```yaml
- name: Add Host to Known Hosts
  run: |
    ssh-keyscan -H your-server-ip >> ~/.ssh/known_hosts
```

---

### **Complete Workflow Example**
Here’s how your deployment job might look after integrating the SSH key:

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout code
      uses: actions/checkout@v3

    - name: Setup SSH Key
      run: |
        mkdir -p ~/.ssh
        echo "${{ secrets.SSH_KEY }}" > ~/.ssh/id_rsa
        chmod 600 ~/.ssh/id_rsa

    - name: Add Host to Known Hosts
      run: |
        ssh-keyscan -H your-server-ip >> ~/.ssh/known_hosts

    - name: Deploy to Server
      run: |
        ssh -o StrictHostKeyChecking=no username@your-server-ip << 'EOF'
          sudo docker stop your_container_name || true
          sudo docker rm your_container_name || true
          sudo docker pull your_docker_image:latest
          sudo docker run -d --name your_container_name -p 5000:5000 your_docker_image:latest
        EOF
```

---

### **Step 5: Test the Workflow**
1. Push changes to your repository.
2. Ensure that the GitHub Actions workflow runs successfully.
3. Verify that the application is deployed on your server.

---

### **Important Notes**
- **Never commit your private SSH key** to the repository.
- If you use a passphrase for your SSH key, you may need an additional secret for the passphrase and update your workflow to handle it.
- Use environment variables and GitHub Secrets to ensure sensitive data is securely managed.














on:
  push:
    branches:
      - main


jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - name: checkout
      uses: actions/checkout@v3

    - name: login to dockerhub
      uses: docker/login-action@v2
      with:
        username: ${{ secrets.DOCKER_USERNAME }}
        password: ${{ secrets.DOCKER_PASSWORD }}

    - name: build image
      run: |
        docker build -t ${{ secrets.DOCKER_USERNAME }}/cicd_m:latest .
    - name: push to dockerhub
      run: |
        docker push ${{ secrets.DOCKER_USERNAME }}/cicd_m:latest
  
  deploy-1:
    runs-on: ubuntu-latest
    needs: build 
    steps:
    - name: deploy
      run: |
        mkdir -p ~/.ssh
        echo "${{ secrets.SSH_KEY }}" > ~/.ssh/cicd_m
        chmod 600 ~/.ssh/cicd_m
        ssh -o StrictHostKeyChecking=no -i ~/.ssh/cicd_m ubuntu@54.227.63.56 << '+'
        
        sudo docker stop cicd_m || true
        sudo docker rm cicd_m || true
        sudo docker pull ${{ secrets.DOCKER_USERNAME }}/cicd_m:latest
        sudo docker run -d --name cicd_m -p 5000:5000 ${{ secrets.DOCKER_USERNAME }}/cicd_m:latest
        +

  deploy-2:
    runs-on: ubuntu-latest
    needs: build 
    steps:
    - name: deploy
      run: |
        mkdir -p ~/.ssh
        echo "${{ secrets.SSH_SECOND }}" > ~/.ssh/cicd_m
        chmod 600 ~/.ssh/cicd_m
        ssh -o StrictHostKeyChecking=no -i ~/.ssh/cicd_m ubuntu@100.27.246.28 << '+'
        
        sudo docker stop cicd_m || true
        sudo docker rm cicd_m || true
        sudo docker pull ${{ secrets.DOCKER_USERNAME }}/cicd_m:latest
        sudo docker run -d --name cicd_m -p 5000:5000 ${{ secrets.DOCKER_USERNAME }}/cicd_m:latest
        +
  
    - name: successful completion
      run: echo "Deployment to EC2 complete."
