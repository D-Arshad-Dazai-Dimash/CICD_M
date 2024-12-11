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




























     
---

### 1. **Подготовка AWS EC2 инстансов**

1. **Создайте два EC2 инстанса**:
   - Тип инстанса: `t2.micro` (для тестов подойдет).
   - Операционная система: Ubuntu (рекомендуется 20.04 или 22.04).
   - Скачайте приватный ключ (PEM-файл).

2. **Настройте инстансы**:
   - Убедитесь, что порты **5000** и **22** открыты в **Security Groups**.
   - Зайдите на каждый инстанс через SSH:
     ```bash
     ssh -i "your-key.pem" ubuntu@<EC2_IP>
     ```
   - Установите Docker:
     ```bash
     sudo apt update
     sudo apt install -y docker.io
     sudo systemctl enable docker
     sudo systemctl start docker
     ```
   - Дайте текущему пользователю доступ к Docker:
     ```bash
     sudo usermod -aG docker $USER
     ```
     Перезапустите сессию SSH.

---

### 2. **Создайте Dockerfile**
В директории проекта создайте файл `Dockerfile`:
```dockerfile
FROM python:3.9-slim

WORKDIR /app

COPY requirements.txt requirements.txt
RUN pip install -r requirements.txt

COPY . .

CMD ["python", "app.py"]
```

---

### 3. **Настройка GitHub Secrets**

1. **Добавьте следующие секреты в GitHub Actions**:
   - `DOCKER_USERNAME` – ваш DockerHub логин.
   - `DOCKER_PASSWORD` – ваш DockerHub пароль.
   - `SSH_KEY` – содержимое PEM-файла для первого инстанса.
   - `SSH_SECOND` – содержимое PEM-файла для второго инстанса.

2. **Проверьте приватные ключи**:
   ```bash
   cat your-key.pem | pbcopy  # macOS
   cat your-key.pem | xclip -sel clip  # Linux
   ```

---

### 4. **Настройка Load Balancer**

1. В AWS Console откройте **EC2 -> Load Balancers**.
2. Создайте Load Balancer:
   - Тип: Application Load Balancer.
   - Укажите оба EC2 инстанса в **Target Group**.
   - Настройте Listener на порт 5000.
3. Запишите DNS-имя вашего Load Balancer для дальнейшего использования.

---

### 5. **GitHub Actions Workflow**

#### Объяснение шагов вашего YAML:
- **`build`**: Строит Docker-образ и пушит его на DockerHub.
- **`deploy-1` и `deploy-2`**: Деплоят образ на два EC2 инстанса через SSH.

Используем ваш YAML, слегка скорректированный:
```yaml
name: CI/CD Pipeline

on:
  push:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - name: Checkout code
      uses: actions/checkout@v3

    - name: Login to DockerHub
      uses: docker/login-action@v2
      with:
        username: ${{ secrets.DOCKER_USERNAME }}
        password: ${{ secrets.DOCKER_PASSWORD }}

    - name: Build Docker image
      run: docker build -t ${{ secrets.DOCKER_USERNAME }}/cicd_m:latest .

    - name: Push Docker image to DockerHub
      run: docker push ${{ secrets.DOCKER_USERNAME }}/cicd_m:latest

  deploy-1:
    runs-on: ubuntu-latest
    needs: build
    steps:
    - name: Deploy to EC2-1
      run: |
        mkdir -p ~/.ssh
        echo "${{ secrets.SSH_KEY }}" > ~/.ssh/cicd_m
        chmod 600 ~/.ssh/cicd_m
        ssh -o StrictHostKeyChecking=no -i ~/.ssh/cicd_m ubuntu@<EC2_1_IP> << EOF
        sudo docker stop cicd_m || true
        sudo docker rm cicd_m || true
        sudo docker pull ${{ secrets.DOCKER_USERNAME }}/cicd_m:latest
        sudo docker run -d --name cicd_m -p 5000:5000 ${{ secrets.DOCKER_USERNAME }}/cicd_m:latest
        EOF

  deploy-2:
    runs-on: ubuntu-latest
    needs: build
    steps:
    - name: Deploy to EC2-2
      run: |
        mkdir -p ~/.ssh
        echo "${{ secrets.SSH_SECOND }}" > ~/.ssh/cicd_m
        chmod 600 ~/.ssh/cicd_m
        ssh -o StrictHostKeyChecking=no -i ~/.ssh/cicd_m ubuntu@<EC2_2_IP> << EOF
        sudo docker stop cicd_m || true
        sudo docker rm cicd_m || true
        sudo docker pull ${{ secrets.DOCKER_USERNAME }}/cicd_m:latest
        sudo docker run -d --name cicd_m -p 5000:5000 ${{ secrets.DOCKER_USERNAME }}/cicd_m:latest
        EOF
```

---

### 6. **Проверка**
1. После пуша в `main` ветку:
   - Пайплайн строит Docker-образ.
   - Пушит образ в DockerHub.
   - Деплоит его на два EC2 инстанса.
2. **Тестирование**:
   - Откройте DNS-имя Load Balancer: `http://<LOAD_BALANCER_DNS>:5000`.

---











При создании Application Load Balancer (ALB) в AWS, нужно настроить несколько важных параметров. Вот детальная инструкция:

---

### 1. **Тип Load Balancer**
1. Перейдите в **AWS Management Console -> EC2 -> Load Balancers -> Create Load Balancer**.
2. Выберите **Application Load Balancer (ALB)**.
3. Заполните основные параметры:
   - **Name**: Укажите имя ALB (например, `my-alb`).
   - **Scheme**: 
     - Выберите **Internet-facing**, если вы хотите, чтобы ALB был доступен извне.
     - Выберите **Internal**, если ALB используется только внутри вашей сети AWS.
   - **IP Address Type**: Оставьте значение **IPv4**.

---

### 2. **Настройка Listener**
1. Добавьте **Listener**:
   - **Protocol**: Выберите `HTTP`.
   - **Port**: Укажите порт `5000` (или другой порт вашего приложения).
2. Нажмите **Next**.

---

### 3. **Availability Zones**
1. Выберите **VPC**, в которой находятся ваши EC2 инстансы.
2. Отметьте **Availability Zones** (например, `us-east-1a`, `us-east-1b`) и связанные **Subnets**.
   - ALB должен находиться в той же VPC, где расположены ваши EC2 инстансы.

---

### 4. **Настройка Security Groups**
1. Создайте новый или выберите существующий **Security Group**.
2. Убедитесь, что Security Group позволяет входящие соединения:
   - Протокол: **HTTP**.
   - Порт: **5000**.
   - Источник: `0.0.0.0/0` (или ограничьте доступ по вашему IP).

---

### 5. **Настройка Target Groups**
1. Создайте **Target Group**:
   - **Name**: Укажите имя (например, `my-target-group`).
   - **Target Type**: Выберите `Instance`.
   - **Protocol**: `HTTP`.
   - **Port**: `5000`.
   - **VPC**: Укажите ту же VPC, что и у ALB.
2. **Health Checks**:
   - **Protocol**: `HTTP`.
   - **Path**: Укажите путь для проверки состояния (например, `/health`, если ваш сервис возвращает статус здоровья на этом пути).
   - **Healthy threshold**: 2 (минимум успешных проверок для работы).
   - **Interval**: 30 секунд (по умолчанию).
3. Нажмите **Next**.

---

### 6. **Добавление EC2 в Target Group**
1. После создания **Target Group** выберите ваши два EC2 инстанса.
2. Нажмите **Register Targets**.

---

### 7. **Проверка и запуск**
1. Просмотрите всю информацию.
2. Нажмите **Create Load Balancer**.

---

### 8. **Тестирование ALB**
1. После успешного создания, вы получите DNS-имя ALB, например:
   ```
   my-alb-1234567890.us-east-1.elb.amazonaws.com
   ```
2. Перейдите по этому адресу в браузере:
   ```
   http://my-alb-1234567890.us-east-1.elb.amazonaws.com:5000
   ```
   Вы должны увидеть ваш работающий сервис.

---

### Полезные советы
1. **Health Checks**: Если инстансы не отображаются как "Healthy", проверьте:
   - Установлен ли Docker и запущен ли ваш контейнер на EC2.
   - Доступен ли путь `/health` или выбранный путь проверки.
   - Соответствует ли порт 5000 настройкам Target Group.

2. **Debugging**: Если ALB не работает:
   - Проверьте Security Groups для ALB и EC2.
   - Убедитесь, что инстансы зарегистрированы в Target Group.











Создание и настройка **Security Group (SG)** в AWS – это ключевой шаг для обеспечения безопасности и доступности вашего ALB и EC2 инстансов. Вот пошаговое руководство с подробным описанием полей:

---

### 1. **Где настроить Security Group**
1. В AWS Console перейдите в **EC2 -> Security Groups**.
2. Нажмите **Create Security Group**.

---

### 2. **Основные поля при создании SG**
1. **Security group name**:
   - Укажите уникальное имя, чтобы было понятно, для чего SG используется (например, `alb-sg` или `ec2-sg`).
2. **Description**:
   - Кратко опишите назначение группы (например, `Security Group for ALB` или `Security Group for EC2`).
3. **VPC**:
   - Выберите VPC, в которой расположены ваши EC2 и ALB.

---

### 3. **Настройка правил входящего трафика (Inbound Rules)**

Входящие правила определяют, какие соединения разрешены для ALB и EC2 инстансов.

#### **Для ALB**
Создайте SG для ALB, если он публичный (Internet-facing):
1. **Разрешить HTTP трафик для порта Load Balancer**:
   - **Type**: HTTP.
   - **Protocol**: TCP.
   - **Port Range**: `80` (или `5000`, если используется нестандартный порт).
   - **Source**: `0.0.0.0/0` для доступа из любой точки интернета. *(Внимание: это открывает ALB для всех, используйте только для тестов)*.

2. **Если используется HTTPS**:
   - Добавьте правило:
     - **Type**: HTTPS.
     - **Protocol**: TCP.
     - **Port Range**: `443`.
     - **Source**: `0.0.0.0/0`.

---

#### **Для EC2 инстансов**
Создайте отдельный SG для EC2 инстансов, на которые ALB будет перенаправлять трафик:
1. **Разрешить HTTP трафик от ALB**:
   - **Type**: HTTP.
   - **Protocol**: TCP.
   - **Port Range**: `5000` (или порт вашего приложения).
   - **Source**: SG ID Security Group ALB (например, `sg-0123456789abcdef`).

2. **Разрешить SSH доступ для администрирования**:
   - **Type**: SSH.
   - **Protocol**: TCP.
   - **Port Range**: `22`.
   - **Source**:
     - Ваш IP-адрес: выберите **My IP** (например, `203.0.113.0/32`).
     - Это ограничивает SSH доступ только с вашего компьютера.

---

### 4. **Настройка правил исходящего трафика (Outbound Rules)**
Обычно AWS по умолчанию разрешает весь исходящий трафик. Если нужно, настройте:

1. Разрешить весь исходящий трафик:
   - **Type**: All traffic.
   - **Protocol**: All.
   - **Port Range**: All.
   - **Destination**: `0.0.0.0/0`.

2. Если хотите ограничить исходящий трафик:
   - Например, разрешить только HTTP и HTTPS:
     - **Type**: HTTP/HTTPS.
     - **Protocol**: TCP.
     - **Port Range**: `80` и `443`.
     - **Destination**: `0.0.0.0/0`.

---

### 5. **Привязка Security Group**
1. **К ALB**:
   - Перейдите в **EC2 -> Load Balancers**.
   - Найдите ваш ALB, нажмите **Edit security groups**.
   - Выберите созданную SG для ALB (например, `alb-sg`).

2. **К EC2**:
   - Перейдите в **EC2 -> Instances**.
   - Найдите ваши инстансы, нажмите **Actions -> Security -> Change security groups**.
   - Добавьте SG для EC2 (например, `ec2-sg`).

---

### Пример правил Security Group

#### **ALB Security Group**
| Type  | Protocol | Port Range | Source      | Description                |
|-------|----------|------------|-------------|----------------------------|
| HTTP  | TCP      | 80         | 0.0.0.0/0   | Allow HTTP from anywhere   |
| HTTPS | TCP      | 443        | 0.0.0.0/0   | Allow HTTPS from anywhere  |

#### **EC2 Security Group**
| Type  | Protocol | Port Range | Source      | Description                         |
|-------|----------|------------|-------------|-------------------------------------|
| HTTP  | TCP      | 5000       | sg-ALB-ID   | Allow HTTP traffic from ALB         |
| SSH   | TCP      | 22         | Your IP     | Allow SSH from your specific IP     |

---

### Полезные советы
1. **Минимизация рисков**:
   - Избегайте использования `0.0.0.0/0` для SSH. Это открывает доступ всему интернету.
   - Если нужно обеспечить безопасность, используйте VPN или ограничьте доступ по IP.

2. **Проверка соединений**:
   - Используйте команды `curl` или браузер для тестирования ALB.
   - Проверьте доступность EC2 внутри VPC.








Вот детальное и объединённое руководство для настройки CI/CD пайплайна с использованием **Docker**, **GitHub Actions**, **AWS EC2**, **Load Balancer** и **Security Groups**. Мы разберем все шаги от начала до конца.

---

## Шаг 1. Создание EC2 Инстансов

1. **Создайте 2 EC2 инстанса**:
   - **Тип инстанса**: `t2.micro` (бесплатный уровень, если тестируете).
   - **ОС**: Ubuntu 20.04 или 22.04.
   - **VPC**: Оставьте значение по умолчанию.
   - **Скачайте приватный ключ** (PEM-файл).

2. **Настройте EC2**:
   - Подключитесь к каждому инстансу через SSH:
     ```bash
     ssh -i "your-key.pem" ubuntu@<EC2_IP>
     ```
   - Установите Docker:
     ```bash
     sudo apt update
     sudo apt install -y docker.io
     sudo systemctl enable docker
     sudo systemctl start docker
     sudo usermod -aG docker $USER
     ```
   - Перезапустите SSH-сессию для активации изменений.

---

## Шаг 2. Создание Dockerfile

В корне вашего проекта создайте файл `Dockerfile`:

```dockerfile
FROM python:3.9-slim

WORKDIR /app

COPY requirements.txt requirements.txt
RUN pip install -r requirements.txt

COPY . .

CMD ["python", "app.py"]
```

**Пример `requirements.txt`:**
```
flask
```

---

## Шаг 3. Создание Application Load Balancer (ALB)

1. Перейдите в AWS Management Console → **EC2 → Load Balancers** → **Create Load Balancer**.
2. Выберите **Application Load Balancer (ALB)**.
3. Настройте параметры:
   - **Name**: `my-alb`.
   - **Scheme**: `Internet-facing` (если нужен публичный доступ).
   - **IP Address Type**: `IPv4`.
4. **Listeners**:
   - Добавьте `HTTP` Listener на порт `80` (или порт вашего приложения, например, `5000`).

5. **Availability Zones**:
   - Выберите VPC, в которой находятся ваши EC2.
   - Укажите Subnets для доступных зон (например, `us-east-1a` и `us-east-1b`).

6. **Создайте Target Group**:
   - **Name**: `my-target-group`.
   - **Target Type**: `Instance`.
   - **Protocol**: `HTTP`.
   - **Port**: `5000`.
   - **Health Checks**:
     - **Path**: `/health` (или путь вашего health-check).
     - **Interval**: 30 секунд.

7. **Добавьте EC2 в Target Group**:
   - Выберите оба EC2 инстанса → Нажмите **Register Targets**.

8. Нажмите **Create Load Balancer**.

---

## Шаг 4. Создание Security Groups

### 1. Security Group для ALB
- Перейдите в **EC2 → Security Groups** → **Create Security Group**.
- Заполните параметры:
  - **Name**: `alb-sg`.
  - **Description**: `Security Group for ALB`.
  - **VPC**: Выберите VPC ваших EC2.
- **Inbound Rules**:
  - Разрешить HTTP:
    - **Type**: HTTP.
    - **Port Range**: `80`.
    - **Source**: `0.0.0.0/0`.
- **Outbound Rules**:
  - Оставьте значение **All traffic**.

### 2. Security Group для EC2
- Создайте второй Security Group:
  - **Name**: `ec2-sg`.
  - **Description**: `Security Group for EC2`.
  - **VPC**: Та же VPC.
- **Inbound Rules**:
  - Разрешить HTTP от ALB:
    - **Type**: HTTP.
    - **Port Range**: `5000`.
    - **Source**: SG ID ALB (например, `sg-0123456789abcdef`).
  - Разрешить SSH для администрирования:
    - **Type**: SSH.
    - **Port Range**: `22`.
    - **Source**: `My IP`.
- **Outbound Rules**:
  - Оставьте значение **All traffic**.

---

## Шаг 5. Настройка GitHub Actions

### 1. Настройте Secrets:
В вашем репозитории GitHub добавьте Secrets:
- `DOCKER_USERNAME` – ваш DockerHub логин.
- `DOCKER_PASSWORD` – ваш DockerHub пароль.
- `SSH_KEY` – приватный ключ PEM для первого EC2.
- `SSH_SECOND` – приватный ключ PEM для второго EC2.

### 2. Создайте workflow файл `.github/workflows/main.yml`:

```yaml
name: CI/CD Pipeline

on:
  push:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - name: Checkout code
      uses: actions/checkout@v3

    - name: Login to DockerHub
      uses: docker/login-action@v2
      with:
        username: ${{ secrets.DOCKER_USERNAME }}
        password: ${{ secrets.DOCKER_PASSWORD }}

    - name: Build Docker image
      run: docker build -t ${{ secrets.DOCKER_USERNAME }}/midterm:latest .

    - name: Push Docker image to DockerHub
      run: docker push ${{ secrets.DOCKER_USERNAME }}/midterm:latest

  deploy-1:
    runs-on: ubuntu-latest
    needs: build
    steps:
    - name: Deploy to EC2-1
      run: |
        mkdir -p ~/.ssh
        echo "${{ secrets.SSH_KEY }}" > ~/.ssh/midterm
        chmod 600 ~/.ssh/midterm
        ssh -o StrictHostKeyChecking=no -i ~/.ssh/midterm ubuntu@<EC2_1_IP> << EOF
        sudo docker stop midterm || true
        sudo docker rm midterm || true
        sudo docker pull ${{ secrets.DOCKER_USERNAME }}/midterm:latest
        sudo docker run -d --name midterm -p 5000:5000 ${{ secrets.DOCKER_USERNAME }}/midterm:latest
        EOF

  deploy-2:
    runs-on: ubuntu-latest
    needs: build
    steps:
    - name: Deploy to EC2-2
      run: |
        mkdir -p ~/.ssh
        echo "${{ secrets.SSH_SECOND }}" > ~/.ssh/midterm
        chmod 600 ~/.ssh/midterm
        ssh -o StrictHostKeyChecking=no -i ~/.ssh/midterm ubuntu@<EC2_2_IP> << EOF
        sudo docker stop midterm || true
        sudo docker rm midterm || true
        sudo docker pull ${{ secrets.DOCKER_USERNAME }}/midterm:latest
        sudo docker run -d --name midterm -p 5000:5000 ${{ secrets.DOCKER_USERNAME }}/midterm:latest
        EOF
```

---

## Шаг 6. Проверка работы

1. **Пушьте код в `main`**:
   - Workflow автоматически соберет Docker-образ, зальет его в DockerHub и задеплоит на EC2.

2. **Тестирование ALB**:
   - Откройте в браузере:
     ```
     http://<ALB_DNS>:5000
     ```

3. **Проверка ошибок**:
   - Если инстансы не "Healthy", проверьте:
     - Запущен ли Docker-контейнер на EC2.
     - Правильный ли путь в Health Check.

---




---

### **1. Create `ci.yml` for CI**

This workflow will handle the build and push of the Docker image to DockerHub.

```yaml
name: CI Pipeline

on:
  push:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - name: Checkout code
      uses: actions/checkout@v3

    - name: Login to DockerHub
      uses: docker/login-action@v2
      with:
        username: ${{ secrets.DOCKER_USERNAME }}
        password: ${{ secrets.DOCKER_PASSWORD }}

    - name: Build Docker image
      run: docker build -t ${{ secrets.DOCKER_USERNAME }}/midterm:latest .

    - name: Push Docker image to DockerHub
      run: docker push ${{ secrets.DOCKER_USERNAME }}/midterm:latest
```

---

### **2. Create `cd.yml` for CD**

This workflow will deploy the Docker image to the EC2 instances. It will trigger only after the `ci.yml` workflow completes successfully.

#### Trigger Deployment Manually or Automatically
To trigger this workflow:
- Use the `workflow_run` event to start this workflow after `ci.yml`.
- Alternatively, use a manual dispatch (if desired).

Here’s the deployment workflow (`cd.yml`):

```yaml
name: CD Pipeline

on:
  workflow_run:
    workflows:
      - CI Pipeline
    types:
      - completed

jobs:
  deploy-1:
    runs-on: ubuntu-latest
    steps:
    - name: Deploy to EC2-1
      run: |
        mkdir -p ~/.ssh
        echo "${{ secrets.SSH_KEY }}" > ~/.ssh/midterm
        chmod 600 ~/.ssh/midterm
        ssh -o StrictHostKeyChecking=no -i ~/.ssh/midterm ubuntu@<EC2_1_IP> << EOF
        sudo docker stop midterm || true
        sudo docker rm midterm || true
        sudo docker pull ${{ secrets.DOCKER_USERNAME }}/midterm:latest
        sudo docker run -d --name midterm -p 5000:5000 ${{ secrets.DOCKER_USERNAME }}/midterm:latest
        EOF

  deploy-2:
    runs-on: ubuntu-latest
    needs: deploy-1
    steps:
    - name: Deploy to EC2-2
      run: |
        mkdir -p ~/.ssh
        echo "${{ secrets.SSH_SECOND }}" > ~/.ssh/midterm
        chmod 600 ~/.ssh/midterm
        ssh -o StrictHostKeyChecking=no -i ~/.ssh/midterm ubuntu@<EC2_2_IP> << EOF
        sudo docker stop midterm || true
        sudo docker rm midterm || true
        sudo docker pull ${{ secrets.DOCKER_USERNAME }}/midterm:latest
        sudo docker run -d --name midterm -p 5000:5000 ${{ secrets.DOCKER_USERNAME }}/midterm:latest
        EOF
```

---

### Key Points:
1. **Workflow Dependencies**:
   - `ci.yml` builds the image and pushes it to DockerHub.
   - `cd.yml` triggers after `ci.yml` completes using `workflow_run`.

2. **Separate Workflows**:
   - Each workflow has a specific role, keeping your CI/CD pipeline clean and modular.

3. **Manual Trigger (Optional)**:
   - Add a manual dispatch option for `cd.yml` if automatic deployment is not required:
     ```yaml
     on:
       workflow_dispatch:
     ```

