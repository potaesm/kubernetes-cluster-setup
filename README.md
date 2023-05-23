# Kubernetes Cluster Setup

- [How to install Minikube in Google Cloud Vm Instance](https://medium.com/google-cloud/how-to-install-minikube-in-google-cloud-vm-instance-b5ea57cb2204)
- [Accessing a remote minikube from a local computer](https://faun.pub/accessing-a-remote-minikube-from-a-local-computer-fd6180dd66dd#:~:text=You%20can't%20access%20minikube,forward%20them%20to%20kube%2Dapiserver.)
- [Setup Your Own Kubernetes Cluster with K3s](https://itnext.io/setup-your-own-kubernetes-cluster-with-k3s-b527bf48e36a)
- [3 ways to connect to your K3s Kubernetes Cluster](https://headworq.org/en/how-to-connect-to-kubernetes/)
- [NestJS Kubernetes Deployment](https://huseyinnurbaki.medium.com/nestjs-kubernetes-deployment-part-2-deployment-dad327dee631)

1. Install Docker
```bash
sudo apt update
sudo apt install docker.io
docker --version
sudo usermod -aG docker $USER && newgrp docker
```

2. Install Minikube dependencies
```bash
sudo apt-get update && sudo apt-get install -y apt-transport-https gnupg2 curl
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
sudo apt-get update && sudo apt-get install -y docker-ce docker-ce-cli containerd.io
sudo curl -Lo /usr/local/bin/kubectl https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl && sudo chmod +x /usr/local/bin/kubectl
sudo apt-get update && sudo apt-get install -y qemu-kvm libvirt-daemon-system libvirt-clients bridge-utils virt-manager
```

3. Install Minikube
```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
minikube start --driver=docker --memory 6144 --cpus 2
```

4. Install Nginx
```bash
sudo apt update
sudo apt install nginx
# Get minikube server address
cat ~/.kube/config | grep server:
minikube ip
# Generate password for minikube
sudo apt-get install apache2-utils -y
sudo htpasswd -c /etc/nginx/.htpasswd minikube
# Update Nginx config file
sudo nano /etc/nginx/nginx.conf
# Test Nginx config file
sudo nginx -t
# Restart Nginx
sudo systemctl restart nginx
# Enable Minikube Nginx add-on
minikube addons enable ingress
minikube tunnel
```
- Nginx Config
```conf
events {
    worker_connections 4096;
}
http {
    default_type application/octet-stream;
    # kubernetes api
    server {
        listen 8443;
        listen [::]:8443;
        auth_basic_user_file /etc/nginx/.htpasswd;
        location / {
            proxy_pass https://<MINIKUBE_IP_ADDRESS>:8443;
            proxy_ssl_certificate /home/<USER_NAME>/.minikube/profiles/minikube/client.crt;
            proxy_ssl_certificate_key /home/<USER_NAME>/.minikube/profiles/minikube/client.key;
        }
    }
    server {
        listen 80;
        listen [::]:80;
        # backend
        location /my-backend-app-one {
            # proxy_pass http://<MINIKUBE_IP_ADDRESS>.nip.io/my-backend-app-one;
            proxy_pass http://<MINIKUBE_IP_ADDRESS>/my-backend-app-one;
        }
        location /my-backend-app-two {
            # proxy_pass http://<MINIKUBE_IP_ADDRESS>.nip.io/my-backend-app-two;
            proxy_pass http://<MINIKUBE_IP_ADDRESS>/my-backend-app-two;
        }
        # frontend
        location ~ \.css {
            add_header  Content-Type    text/css;
        }
        location / {
            include /etc/nginx/mime.types;
            sendfile on;
            proxy_pass http://<MINIKUBE_IP_ADDRESS>/;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header Host $host;
            proxy_cache_bypass $http_upgrade;
        }
    }
    ssl_session_cache   shared:SSL:10m;
    ssl_session_timeout 10m;
    server {
        listen 443 ssl;
        listen [::]:443 ssl;
        server_name         subdomain.domain.topleveldomain;
        keepalive_timeout   70;
        ssl_certificate     /home/<USER_NAME>/cert.pem;
        ssl_certificate_key /home/<USER_NAME>/privkey.pem;
        ssl_protocols       TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
        ssl_ciphers         HIGH:!aNULL:!MD5;
    }
}
```

5. Config firewall
```bash
# Ubuntu Firewall
sudo apt install ufw
sudo ufw allow 8443/tcp
sudo ufw enable
sudo ufw status
# GCloud Firewall Rules
# gcloud compute firewall-rules delete kubernetes-api
gcloud compute firewall-rules create kubernetes-api --allow=tcp:8443 --direction=ingress --enable-logging --description="Allow incoming traffic on Kubernetes API"
# Check ports usage
sudo apt install net-tools
sudo netstat -ntlp
```

6. Use Kubeconfig
```yaml
apiVersion: v1
clusters:
  - cluster:
      server: http://minikube:<PASSWORD>@<REMOTE_IP_ADDRESS>:8443
    name: minikube-cluster
contexts:
  - context:
      cluster: minikube-cluster
    name: minikube-context
current-context: minikube-context
kind: Config
```
```bash
kubectl --kubeconfig=/path/to/config.yaml get nodes
export KUBECONFIG=${KUBECONFIG:-~/.kube/config}:/path/to/config.yaml
```