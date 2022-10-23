# Installing Apache airflow using helm chart on Windows 11 WSL2 / microk8s
## Intro
I needed a playground to quickly spin various data engineering platforms, including Apache Airflow and most of them had official helm charts to get up and running. It turned out, my personal laptop is running Windows 11 and has WSL on it, which, at the time of writing started to support systemd. I went through several resources and did some small additions on my own, which led to this quick guide.
This guide was composed using the following resources:
- https://learn.microsoft.com/en-us/windows/wsl/install
- https://devblogs.microsoft.com/commandline/systemd-support-is-now-available-in-wsl/
- https://github.com/microsoft/WSL/releases
- https://microk8s.io/docs/getting-started
- https://airflow.apache.org/docs/helm-chart/stable/index.html
## Requirements
- Windows 11 which is at release 22H2 at the time of writing. Older releases of Windows 10/11 may or may not be supported
- WSL version 2 with "Preview" release of 0.7.0 or higher. This installed automatically with wsl --install, but if you have previous versions of WSL, please uninstall them or make sure you are using "Preview release" from the store or WSL releases URL below.
- Sufficient RAM and fast CPU. 16Gb / Ryzen 7 4000 series on my ThinkBook G14ARE2 work fine.
## Installation steps
### Install wsl on clean install of Windows 11
```
wsl --install
wsl --version
```
Version should be higher than 0.67
You can otherwise download preview version of WSL from https://github.com/microsoft/WSL/releases
You can install it using ```add-appxpackage <package.msix>``` cmdlet in privileged Powershell

### Login to wsl shell and enable systemd
```
sudo tee -a /etc/wsl.conf << EOF
[boot]
systemd=true
EOF
```

### Shutdown and relogin to wsl from Windows command prompt
```
wsl --shutdown
wsl
```

### install microk8s

```
sudo snap install microk8s --classic
sudo usermod -a -G microk8s $USER
sudo chown -f -R $USER ~/.kube
newgrp microk8s
```
### microk8s kubectl and helm aliases with autocompletion
```
tee -a ~/.bashrc << EOF
alias kubectl=microk8s.kubectl
source <(microk8s.kubectl completion bash)
alias helm=microk8s.helm
source <(microk8s.helm completion bash)
EOF
source ~/.bashrc
```
### Check status afer 1 minute or so
```
microk8s status
```
### Enable optional services
```
microk8s enanble dashboard dns ha-cluster helm helm3 host-access hostpath-storage metrics-server storage         
```
### Wait for 2-3 mins and you can install airflow
```
microk8s helm repo add apache-airflow https://airflow.apache.org
microk8s helm upgrade --install airflow apache-airflow/airflow --namespace airflow --create-namespace
```

### expose deployment as node port
```
kubectl --namespace airflow expose deployment airflow-webserver --type=NodePort --name airflow-webserver-nodeport
```
Check the node port service:
```
kubectl --namespace airflow get service | grep NodePort
# airflow-webserver-nodeport    NodePort    10.152.183.103   <none>        8080:32700/TCP      3h43m
```
Note the port mapping 8080:<destination> and use WSL IP address and port number to access Airflow web UI.
You can obtain IP address of your WSL bridge:
```
ip addr | grep 172
# this assumes your IP address of WSL bridge on Windows is 172.x.y.z
```
e.g. http://172.21.152.92:32700/