## Данный проект описывает как добавить backend приложение и postgres образы из DockerHub в kubernets узел

Kubernetes был создан с помощью minikube на удаленном сервере ubuntu 22.04.

### Необходимые компоненты:
- minikube
- kubectl
- docker
- nginx

### Для того чтобы запустить minicube требуются следующие команды:  
Подробнее: https://kubernetes.io/ru/docs/tasks/tools/install-minikube/
- Установить kubectl
```bash
curl -LO https://dl.k8s.io/release/`curl -LS https://dl.k8s.io/release/stable.txt`/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl

# Убедиться в установке
kubectl version --client
```
- Установить Docker
```bash
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update

sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

sudo groupadd docker
sudo usermod -aG docker $USER
newgrp docker

sudo systemctl enable docker.service
sudo systemctl enable containerd.service

# Убедиться в установке
docker --version
```
- Установить minikube
```bash
curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
chmod +x minikube
sudo mv minikube /usr/local/bin/

# Убедиться в установке
minikube version
```
- Запустить minikube
```bash
minikube start
```
### Теперь после установки minikube нужно применить configmaps, manifests и services через kubectl
```bash
# Сначала для postgres
kubectl apply -f config_maps/postgres-config_map.yml
kubectl apply -f manifests/postgres-manifest.yml
kubectl apply -f services/postgres-service.yml

# Теперь для backend-образа
kubectl apply -f config_maps/backend-config_map.yml
kubectl apply -f manifests/backend-manifest.yml
kubectl apply -f services/backend-service.yml

# Убедимся что все поды имеют статус RUNING
kubectl get pods
```

Благодаря сервисам backend-service.yml и postgres-service.yml у нас есть доступ к контейнерам по minikube ip и через порты 30000 и 31000.  

- Содержимое backend-service.yml:
```yml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  selector:
    app: backend-pod
  ports:
    - protocol: TCP
      port: 8000
      targetPort: 8000
      nodePort: 30000
  type: NodePort
```
- Содержимое postgres-service.yml:
```yml
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
spec:
  selector:
    app: postgres
  ports:
    - protocol: TCP
      port: 5432
      targetPort: 5432
      nodePort: 31000
  type: NodePort
```
### Далее для доступа к проекту необходимо установить nginx и перенаправить контейнер:
```
sudo apt update
sudo apt install nginx
sudo systemctl enable nginx

# Убедиться в установке
sudo systemctl status nginx
```
- После заменить nginx.conf по пути /etc/nginx/  
```bash
user <username>;
worker_processes auto;

error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    server {
        listen 8000;
        server_name _;

        location / {
            proxy_pass <minikube_ip>:30000;
            proxy_set_header Host $host;
        }
    }
}
```
Узнать ip можно командой:
```bash
minikube ip
```
- Применить изменения
```bash
# Убедиться что нет ошибок 
sudo nginx -t

sudo systemctl reload nginx
```
