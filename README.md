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






###**Добавление Auto Scaling и Load Balancer к вашему проекту с использованием AWS включает в себя несколько шагов. Мы будем настраивать инфраструктуру, чтобы поддерживать Auto Scaling и Load Balancer для вашего Docker-сервиса, который уже развёрнут с помощью GitHub Actions.**

### **Шаг 1. Настройка AWS EC2 Instances**

1. **Создайте AMI (Amazon Machine Image):**
   - Войдите на ваш EC2 сервер, где работает `cicd-m-container`.
   - Настройте сервер с установленным Docker и настройте ваш контейнер.
   - После этого создайте **AMI** (снимок) текущего сервера EC2. Это нужно для создания новых экземпляров в Auto Scaling группе.

2. **Настройте роль для EC2:**
   - Создайте IAM роль с доступом к Amazon EC2 и Load Balancer.
   - Присоедините эту роль к вашим экземплярам EC2.

---

### **Шаг 2. Настройка Auto Scaling**

1. В AWS Management Console:
   - Перейдите в раздел **EC2 > Auto Scaling Groups**.
   - Создайте новую Auto Scaling группу.
     - Выберите созданную ранее AMI.
     - Укажите количество начальных экземпляров (например, `2`).
     - Настройте минимальное (`1`), максимальное (`4`) и желаемое (`2`) количество экземпляров.
   - Добавьте **Health Check** для Load Balancer.

2. Укажите Target Scaling Policy:
   - Укажите, чтобы новые экземпляры создавались при превышении, например, 70% использования CPU.

---

### **Шаг 3. Настройка Load Balancer**

1. Перейдите в **EC2 > Load Balancers**.
   - Создайте Application Load Balancer (ALB).
   - Укажите Target Group, которая включает экземпляры Auto Scaling группы.
   - Настройте маршрут HTTP-трафика к порту `80`.

---

### **Шаг 4. Обновление CD Pipeline для Auto Scaling**

Добавьте шаги для обновления всех экземпляров Auto Scaling группы при развёртывании нового образа Docker.

```yaml
name: CD Pipeline

on:
  workflow_run:
    workflows: ["CI Pipeline"]
    types:
      - completed

jobs:
  deploy:
    runs-on: self-hosted

    steps:
    - name: Pull Docker image
      run: sudo docker pull dimash26/cicd_m:latest
    - name: Update EC2 Instances
      run: |
        INSTANCE_IDS=$(aws autoscaling describe-auto-scaling-instances --query 'AutoScalingInstances[].InstanceId' --output text)
        for ID in $INSTANCE_IDS; do
          aws ec2 reboot-instances --instance-ids $ID
        done
    - name: Delete Old Docker Container
      run: sudo docker rm -f cicd-m-container || true
    - name: Run New Docker Container
      run: sudo docker run -d -p 80:3000 --name cicd-m-container dimash26/cicd_m
```

---

### **Шаг 5. Настройка Health Checks для Load Balancer**

1. В разделе **Target Groups**, настройте Health Check URL, например `/health`:
   - Убедитесь, что ваше приложение возвращает код `200` для этого URL.

---

### **Шаг 6. Тестирование и Проверка**

1. Убедитесь, что Load Balancer правильно распределяет трафик между экземплярами.
2. Проверьте, что Auto Scaling добавляет новые экземпляры при нагрузке.

---

### **Шаг 7. Настройка для Автоматического Обновления Auto Scaling**

Для автоматического обновления Auto Scaling группы при новой версии контейнера, используйте **Lifecycle Hooks**:
- Создайте Lifecycle Hook, который завершает старые экземпляры только после успешного запуска новых.

---

После выполнения этих шагов ваш сервис будет поддерживать Auto Scaling и Load Balancer, обеспечивая отказоустойчивость и масштабируемость.


###**Создание AMI (Amazon Machine Image) в AWS позволяет создать шаблон экземпляра EC2, который включает операционную систему, конфигурации и установленные приложения. Вот пошаговое руководство:**

---

### **Шаг 1: Подготовьте ваш EC2 экземпляр**

1. **Настройте EC2 экземпляр:**
   - Убедитесь, что на вашем экземпляре EC2 установлено всё необходимое:
     - Docker.
     - Настроенный Docker-контейнер (если требуется).
     - Все зависимости, нужные для работы вашего приложения.

2. **Проверьте конфигурацию:**
   - Убедитесь, что ваш сервис работает корректно (например, контейнер Docker запущен, а сайт "Hello World" доступен).

---

### **Шаг 2: Создание AMI**

1. **Перейдите в AWS Management Console:**
   - Войдите в консоль AWS.
   - Перейдите в раздел **EC2**.

2. **Выберите ваш экземпляр EC2:**
   - Найдите экземпляр EC2, который вы хотите использовать для создания AMI.

3. **Создайте образ:**
   - Выберите экземпляр, нажмите на кнопку **Actions (Действия)**.
   - Перейдите в **Image and templates (Изображение и шаблоны)** → **Create Image (Создать образ)**.

4. **Заполните параметры:**
   - В поле **Image Name (Имя образа)** укажите понятное имя, например, `MyWebApp-Docker-AMI`.
   - Укажите описание, чтобы знать, для чего этот образ предназначен.
   - Вы можете оставить дополнительные параметры по умолчанию.

5. **Создайте AMI:**
   - Нажмите **Create Image (Создать образ)**.

---

### **Шаг 3: Мониторинг создания AMI**

1. Перейдите в раздел **AMIs** в панели **Images** (в меню слева).
2. Найдите ваш образ и убедитесь, что его статус изменился на **Available** (Доступен).

---

### **Шаг 4: Использование AMI**

1. **Создание новых экземпляров EC2:**
   - Выберите ваш AMI, нажмите **Launch Instance**.
   - Создайте новый экземпляр, который будет точно таким же, как исходный.

2. **Настройка Auto Scaling:**
   - Используйте ваш AMI как базовый образ при создании Auto Scaling группы (см. предыдущие шаги).

---

### **Полезные советы**

1. **Регулярно обновляйте AMI:**
   - Если ваш проект активно развивается, создавайте новые версии AMI, чтобы отражать изменения в конфигурации.

2. **Храните только нужные AMI:**
   - Удаляйте старые образы, чтобы избежать лишних затрат.

3. **Используйте скрипты запуска:**
   - Для более гибкой настройки вы можете использовать **User Data** для выполнения дополнительных команд при запуске экземпляра из AMI.

После создания AMI вы можете использовать её для Auto Scaling и Load Balancer, чтобы легко разворачивать ваши приложения.
