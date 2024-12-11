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
