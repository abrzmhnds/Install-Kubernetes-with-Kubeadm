# Install Kubernetes with Kubeadm

## Preparation
- Docker
- Kubernetes Tools
- Calico
- Helm
- MetalLB
- Nginx Ingress Controller

### INSTALL DOCKER (Master and Worker)
```
sudo apt-get update
curl -fSsl https://get.docker.com/ | sh
```
```
cat <<EOF | sudo tee /etc/docker/daemon.json
{
"exec-opts": ["native.cgroupdriver=systemd"],
"log-driver": "json-file",
"log-opts": {
"max-size": "100m"
},
"storage-driver": "overlay2"
}
EOF 
```
```
sudo systemctl enable docker
sudo systemctl daemon-reload
sudo systemctl restart docker
```

### INSTALL KUBERNETES TOOLS (Master and Worker)
```
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
```
```
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```
```
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

### INISIALISASI CLUSTER (Master Node)
```
sudo swapoff -a
sudo kubeadm init --pod-network-cidr=190.168.0.0/16 --control-plane-endpoint=10.8.60.225
make sure that pod network did not crash with ip node
```
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
```
sudo kubeadm token list
sudo openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
```

### Troubleshoot
If get an error when kube init
```
sudo rm -rf /etc/cni/net.d 
sudo rm -f /etc/containerd/config.toml
```

### JOIN WORKER (Worker Node)
```
sudo kubeadm join 10.8.60.225:6443 --token osv9gs.cb7xuvlllqcmph4h --discovery-token-ca-cert-hash sha256:95b510bc95ad666fbb6def5620ba25b89385d72ce97f9ef16b4239b51df8b721
```

### INSTALL NETWORK
```
curl https://docs.projectcalico.org/manifests/calico.yaml -o
kubectl create -f calico.yaml
```
you need to update ip in custom-resources.yaml if pod-network-cidr not 192.168.0.0/16

### FINALISASI CLUSTER
```
kubectl label node handis-kube-w node-role.kubernetes.io/worker=worker
kubectl label node handis-kube-m node-role.kubernetes.io/master=master
kubectl label node handis-kube-m node-role.kubernetes.io/control-plane-
kubectl taint node master node-role.kubernetes.io/master:NoSchedule
```

### INSTALL HELM
```
sudo apt install apt-transport-https --yes
curl https://baltocdn.com/helm/signing.asc | sudo apt-key add -
echo "deb https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
```
```
sudo apt install -y helm
```
If theres no apt package helm
```
sudo snap install helm --classic
```

### INSTALL MetalLB for LoadBalancer
``` 
kubectl edit configmap -n kube-system kube-proxy
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: "ipvs"
ipvs:
  strictARP: true
```
```
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.7/config/manifests/metallb-native.yaml
```
```
sudo mkdir metallb
vim ~/metallb/ipaddress_pools.yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: development
  namespace: metallb-system
spec:
  addresses:
  - 10.8.60.230-10.8.60.232
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: pool1-advert
  namespace: metallb-system
spec:
  ipAddressPools:
  - develoment
```
```
kubectl apply -f  ~/metallb/ipaddress_pools.yaml
```

### DEPLOY APP WITH MetalLB
```
kubectl create deploy nginx-test --image nginx && kubectl expose deploy nginx-test --port 80 --type LoadBalancer
```
```
$ kubectl get svc
NAME         TYPE           CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
kubernetes   ClusterIP      10.96.0.1     <none>        443/TCP        11d
nginx-test   LoadBalancer   10.98.57.87   10.8.60.231   80:30083/TCP   78m
```
access external ip on web browser
in case the service unreachable you must configure ssh tunneling and proxy
setup ssh tunneling on mobaxterm
setup proxy in firefox

### INSTALL NGINX INGRESS CONTROLLER
```
helm repo add nginx-stable https://helm.nginx.com/stable
helm repo update
helm install ingress nginx-stable/nginx-ingress
kubectl get svc ingress-nginx-ingress
```

### DEPLOY APP WITH INGRESS CONTROLLER
```
kubectl create deployment nginx-ingress --image nginx
kubectl create deployment apache-ingress --image httpd
kubectl expose deployment nginx-ingress --port 80 --type NodePort
kubectl expose deployment apache-ingress --port 80 --type LoadBalancer
```

### CREATE INGRESS FOR TWO PREVIOUS SERVICES
```
vim ~/ingress/nginx-apache.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-handis
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: nginx.handis.id
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nginx-ingress
                port:
                  number: 80
    - host: apache.handis.id 
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: apache-ingress
                port:
                  number: 80
```
```
kubectl apply -f ~/ingress/nginx-apache.yaml
```
### DEFINE DOMAIN IN /etc/hosts
```
vim /etc/hosts
your-loadbalancer-ipaddress nginx.handis.id
your-loadbalancer-ipaddress apache.handis.id
```