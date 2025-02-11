
# Vagrantfile and Scripts Developer Environment 

## Kubernetes, Vagrant로 로컬 환경 구성

* github : https://github.com/Great-Stone/vagrant-k8s(opens new window)

Virtualbox를 활용하여 로컬 환경에서 Kubernetes(K8s) 환경을 쉽게 만들고 부실수(?) 있는 환경을 구성하기위해 Vagrant를 활용
Intel Mac (Catalina / Big Sur)에서 테스트
Windows는 테스트 필요

#### 실행 전 필요 소프트웨어

Virtualbox : https://www.virtualbox.org(opens new window)
Vagrant : https://www.vagrantup.com(opens new window)

#### Virtualbox 네트워크 구성

K8s vm이 사용하기 위한 네트워크를 추가하여 구성
기존 네트워크를 사용하고 싶다면 vagrantfile의 *.vm.network 부분의 ip에 수정 필요
vagrantfile 구성시 해당 ip 설정 부분을 상단의 NETWORK_SUB 부분에 정의함
네트워크 구성 설정에 따라 클러스터 간 통신이 안될 수 있음에 주의
vagrantfile의 START_IP를 활용하여 마스터 노드 및 워커 노드의 ip를 부여하는 방식으로 구성되었으나 변경 가능

```bash
#구성 설명
#디렉토리/파일 구조
#실행 전
.
├── 1.18
│   ├── files
│   │   └── pv.yaml
│   ├── scripts
│   │   ├── kube.sh
│   │   └── pv.sh
│   └── vagrantfile
├── 1.19
<반복>
```

* 버전별로 디렉토리가 분류되어있음
* 1.18~1.20 은 ubuntu 18.04 LTS 기반
* 1.21~1.23 은 utuntu 20.04 LTS 기반
* vagrantfile
* vagrant 실행 정의
* version2 사용
* kube.sh
* vagrantfile에서 provision으로 호출
* K8s 설치에 필요한 패키지 설치 및 실행
* pv.sh, pv.yaml
* vagrantfile에서 provision으로 호출
* pv.sh는 K8s master 노드 구성 후 디렉토리 생성 후 pv.yaml을 통해 pv 구성

```bash
#실행 후
├── 1.18
│   ├── .kube
│   ├── .vagrant
│   ├── files
│   │   └── pv.yaml
│   ├── join.sh
│   ├── k8s-pv
│   │   ├── pv01
│   │   ├── pv02
│   │   └── pv03
│   ├── scripts
│   │   ├── kube.sh
│   │   └── pv.sh
│   └── vagrantfile
├── 1.19
<반복>

```


* .kube 디렉토리 : kubernetes credential 및 접속 정보 생성
* .vagrant 디렉토리 : vagrant 실행 후 vm 정보 업데이트
* join.sh : 워커노드의 클러스터 조인을 위한 스크립트
* k8s-pv 디렉토리 : pv를 위한 디렉토리 구성

```bash
#vagrantfile - variable

IMAGE_NAME = "bento/ubuntu-20.04"

K8S_MINOR_VERSION = "21"
NETWORK_SUB = "192.168.60."
START_IP = 130
POD_CIDR = "10.#{K8S_MINOR_VERSION}.0.0/16"

cluster = {
  "master" => { :cpus => 2, :mem => 2048 },
  "node" => { :cpus => 1, :mem => 1024 }
}

NODE_COUNT = 1

VM_GROUP_NAME = "k8s-1.#{K8S_MINOR_VERSION}"
DOCKER_VER = "5:20.10.12~3-0~ubuntu-focal"
KUBE_VER = "1.#{K8S_MINOR_VERSION}.8-00"
```

	
| Variable name | value |
| :------------ |:---------------:|
|IMAGE_NAME	|vagrant에서 사용할 기본 이미지로 vagrant cloud 참조|
|K8S_MINOR_VERSION	|K8s 설치의 마이너 버전 지정|
|NETWORK_SUB|	Virtualbox network ip|
|START_IP|	각 K8s 클러스터의 master에 할당되며 워커노드는 +1 씩 증가|
|POD_CIDR|	kubeadm init 의 --pod-network-cidr 에 지정되는 CIDR|
|cluster={}	|클러스터 리소스를 정의한 오브젝트 형태의 변수|
|NODE_COUNT	|워커 노드 개수v
|VM_GROUP_NAME|	Virtualbox에 등록될 그룹 이름|
|DOCKER_VER	|docker-ce 버전|
|KUBE_VER	K8s| 관련 패키지 버전|


```bash
#vagrantfile - vm

Vagrant.configure("2") do |config|
  config.vm.box = IMAGE_NAME

  config.vm.define "master", primary: true do |master|
    master.vm.box = IMAGE_NAME
    master.vm.network "private_network", ip: "#{NETWORK_SUB}#{START_IP}"
    master.vm.hostname = "master"
    master.vm.provision "kube", type: "shell", privileged: true, path: "scripts/kube.sh", env: {
      "docker_ver" => "#{DOCKER_VER}",
      "k8s_ver" => "#{KUBE_VER}"
    }
    master.vm.provision "0", type: "shell", preserve_order: true, privileged: true, inline: <<-SHELL
      OUTPUT_FILE=/vagrant/join.sh
      rm -rf /vagrant/join.sh
      rm -rf /vagrant/.kube
      sudo kubeadm init --apiserver-advertise-address=#{NETWORK_SUB}#{START_IP} --pod-network-cidr=#{POD_CIDR}
      sudo kubeadm token create --print-join-command > /vagrant/join.sh
      chmod +x $OUTPUT_FILE
      mkdir -p $HOME/.kube
      sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
      sudo chown $(id -u):$(id -g) $HOME/.kube/config
      cp -R $HOME/.kube /vagrant/.kube
      #kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
      kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
      kubectl completion bash >/etc/bash_completion.d/kubect
      echo 'alias k=kubectl' >>~/.bashrc
    SHELL

    master.vm.provision "file", preserve_order: true, source: "files", destination: "/tmp"
    master.vm.provision "3", type: "shell", preserve_order: true, privileged: true, path: "scripts/pv.sh"

    master.vm.provider "virtualbox" do |v|
      <생략>
    end # end provider
  end

  (1..NODE_COUNT).each do |i|
    config.vm.define "node-#{i}" do |node|
      node.vm.box = IMAGE_NAME
      node.vm.network "private_network", ip: "#{NETWORK_SUB}#{i + START_IP}"
      node.vm.hostname = "node-#{i}"
        <생략>

      node.vm.provider "virtualbox" do |v|
        <생략>
      end # end provider
    end
  end
end
```

* vm 생성을 위한 정의는 크게 master와 node-#로 구분됨
	- master
		+ master의 경우 1개만을 생성하도록 지정
  		+ .vm.provision을 통해 스크립트를 실행하거나 파일을 복사하여 프로비저닝

* master.vm.provision "0"에서 kubeconfig파일과 워커노드 조인을 위한 토큰 커맨드를 join.sh파일로 생성
	- node-#
 		+ 변수 NODE_COUNT 에서 지정된 개수에 따라 반복 수행
 		+ 앞서 master 프로비저닝시 생성된 join.sh를 이용하여 클러스터에 조인

```
#실행 및 확인
#vagrant cli
cli doc : https://www.vagrantup.com/docs/cli(opens new window)
```yaml


* vagrant up : 프로비저닝을 실행하며, 이미 프로비저닝 된 경우 다시 프로비저닝하지 않고 vm만 기동
* vagrant up vm-name : config.vm.define 에 선언된 아이디에 해당하는 vm만 기동 e.g. : vagrant up master
* vagrant halt : vm 정지
* vagrant halt vm-name : config.vm.define 에 선언된 아이디에 해당하는 vm만 정지 e.g. : vagrant halt node-1
* vagrant status : vm 상태 확인
* vagrant ssh vm-name : config.vm.define 에 선언된 아이디에 해당하는 vm에 ssh 접속 e.g. : vagrant ssh master
* vagrant destroy : 프로비저닝된 vm 삭제

```bash
#Check kubeconfig
master노드 프로비저닝 과정에서 .kube/config를 생성하므로, 해당 kubeconfig를 사용하여 호스트 환경에서 kubectl 사용 가능


~/vagrant-k8s/1.23> kubectl --kubeconfig=./.kube/config get nodes
NAME     STATUS   ROLES                  AGE   VERSION
master   Ready    control-plane,master   68m   v1.23.1
node-1   Ready    <none>                 63m   v1.23.1
node-2   Ready    <none>                 59m   v1.23.1
node-3   Ready    <none>                 54m   v1.23.1
#패키지 버전 확인 방법

#apt-cache policy <packagename>
apt-cache policy kubelet | grep 1.2
  Candidate: 1.23.1-00
     1.23.1-00 500
     1.23.0-00 500
     1.22.5-00 500
     1.22.4-00 500
     1.22.3-00 500
     1.22.2-00 500
     1.22.1-00 500
     1.22.0-00 500
     1.21.8-00 500
     1.21.7-00 500
     ...
```

## 쿠버네티스 version

k8s version 1.22.

## 설치 준비 사항

1. Vagrant, VirtualBox setup
2. 8 Gig + RAM workstation as the Vms use 3 vCPUS and 4+ GB RAM

## For MAC Users

Latest version of Virtualbox for Mac/Linux can cause issues because you have to create/edit the /etc/vbox/networks.conf file and add:
<pre>* 0.0.0.0/0 ::/0</pre>

So that the host only networks can be in any range, not just 192.168.56.0/21 as described here:
https://discuss.hashicorp.com/t/vagrant-2-2-18-osx-11-6-cannot-create-private-network/30984/23
 
## vagrant 실행

To provision the cluster, execute the following commands.

```shell
git clone https://github.com/scriptcamp/vagrant-kubeadm-kubernetes.git
cd vagrant-kubeadm-kubernetes
vagrant up
```

## Kubeconfig file 설정.

```shell
cd vagrant-kubeadm-kubernetes
cd configs
export KUBECONFIG=$(pwd)/config
```

or you can copy the config file to .kube directory.

```shell
cp config ~/.kube/
```

## Kubernetes Dashboard URL

```shell
http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/#/overview?namespace=kubernetes-dashboard
```

## Kubernetes login token

Vagrant up will create the admin user token and saves in the configs directory.

```shell
cd vagrant-kubeadm-kubernetes
cd configs
cat token
```

## To shutdown the cluster, 

```shell
vagrant halt
```

## To restart the cluster,

```shell
vagrant up
```

## To destroy the cluster, 

```shell
vagrant destroy -f
```

## Centos & HA based Setup

If you want Centos based setup, please refer https://github.com/marthanda93/VAAS
  
## 주요 추가 사항
### Disk 추가
PS C:\Users\Administrator> Get-ChildItem Env:
PS C:\Users\Administrator> [Environment]::SetEnvironmentVariable('VAGRANT_EXPERIMENTAL','disks','User')

      node.vm.disk :disk, size: "30GB", name: "extra_storage1"
PS C:\Users\Administrator> vagrant reload

### helm chart 
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add influxdata https://helm.influxdata.com
helm repo add grafana https://grafana.github.io/helm-charts
helm install kafka bitnami/kafka --namespace kafka --create-namespace

### root Disk space resize
ls /sys/class/scsi_device/
echo 1 > /sys/class/scsi_device/2\:0\:0\:0/device/rescan
cfdisk
parted

lvextend -l +100%FREE /dev/mapper/ubuntu--vg-ubuntu--lv
resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv

### default StorageClass
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml
kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

### code server
curl -fsSL https://code-server.dev/install.sh | sh
sudo systemctl enable --now code-server@$USER

### maven
As of 2021/10/25, maven 3.6.x is not supporting Java 17
Create vi ~/.mavenrc file and give any version lower than 17.

wget https://dlcdn.apache.org/maven/maven-3/3.8.4/binaries/apache-maven-3.8.4-bin.tar.gz
sudo mkdir -p /usr/local/apache-maven
sudo mv apache-maven-3.8.4-bin.tar.gz /usr/local/apache-maven
cd /usr/local/apache-maven
sudo tar -xzvf apache-maven-3.8.4-bin.tar.gz

Edit ~/.profile

export M2_HOME=/usr/local/apache-maven/apache-maven-3.8.4
export M2=$M2_HOME/bin
export MAVEN_OPTS="-Xms256m -Xmx512m"
export PATH=$M2:$PAT

### download kubectl for windows 10
https://storage.googleapis.com/kubernetes-release/release/v1.23.0/bin/windows/amd64/kubectl.exe

### date
date -d  '2022-01-01T03:00:00Z' +"%s%N"
date --date='@1643550000'


# Setup for Micro Service Architeure   
## VS code for Vagrant
VS code 설치 : https://code.visualstudio.com/download 

#### VS code Extension(확장프로그램) 설치

- ssh-development 설치

- vagrant 경로에서 vagrant ssh 설정 값 복사

```shell
 vagrant ssh-config
```

- ssh-development 에서 ssh-target 경로(.../user/ssh/config) 설정해서 복사한 vagrant ssh 설정 값 붙여넣기

- Master, node1, node2 확인

<img width="892" alt="스크린샷 2022-05-20 오전 9 14 31" src="https://user-images.githubusercontent.com/24773549/169424150-1ec1052e-29c8-4ad6-ae17-bcfdac0a4f7c.png">

- Master 접속 확인

<img width="1379" alt="스크린샷 2022-05-20 오전 9 17 11" src="https://user-images.githubusercontent.com/24773549/169424328-68acf607-78be-410f-a634-592bb78323db.png">

#### Kafka 설치
helm(쿠버네티스 용 패키지 매니저)으로 설치해도 되지만 Kube 의존없이 수행하기 위해 master에 카프카 설치

- Open JDK 11 설치

```shell
sudo apt-get update 
sudo apt-get install openjdk-11-jdk 
```

- Kafka 설치 (version kafka_2.13-2.8.1) 

```shell
wget https://dlcdn.apache.org/kafka/2.8.1/kafka_2.13-2.8.1.tgz 
tar xvfz kafka_2.13-2.8.1.tgz 
```

# Kubernetes CronJob 생성

## 사전 프로그램 & 조치
SSL을 네트워크팀 담당자한테 할당 받아야함(port : 443 뚫려야함)

1. Vagrant 내 도커 설치 -(참고) https://dongle94.github.io/docker/docker-ubuntu-install/  
2. Vagrant 내 AWS CLI 설치
```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
```

```bash
sudo apt-get install unzip
```

```bash
unzip awscliv2.zip
```

```bash
sudo ./aws/install
```


### 우분투 타임존 설정
```bash
sudo dpkg-reconfigure tzdata
```


## 우분투 Repository Source List (AWS EC2 서버 기준 설정)

```bash
sudo cp /etc/apt/sources.list /etc/apt/sources_temp.list 
```

```bash
sudo vi /etc/apt/sources.list
```

```
## Note, this file is written by cloud-init on first boot of an instance
## modifications made here will not survive a re-bundle.
## if you wish to make changes you can:
## a.) add 'apt_preserve_sources_list: true' to /etc/cloud/cloud.cfg
##     or do the same in user-data
## b.) add sources in /etc/apt/sources.list.d
## c.) make changes to template file /etc/cloud/templates/sources.list.tmpl
 
# See http://help.ubuntu.com/community/UpgradeNotes for how to upgrade to
# newer versions of the distribution.
deb http://ap-northeast-2.ec2.archive.ubuntu.com/ubuntu/ focal main restricted
# deb-src http://ap-northeast-2.ec2.archive.ubuntu.com/ubuntu/ focal main restricted
 
## Major bug fix updates produced after the final release of the
## distribution.
deb http://ap-northeast-2.ec2.archive.ubuntu.com/ubuntu/ focal-updates main restricted
# deb-src http://ap-northeast-2.ec2.archive.ubuntu.com/ubuntu/ focal-updates main restricted
 
## N.B. software from this repository is ENTIRELY UNSUPPORTED by the Ubuntu
## team. Also, please note that software in universe WILL NOT receive any
## review or updates from the Ubuntu security team.
deb http://ap-northeast-2.ec2.archive.ubuntu.com/ubuntu/ focal universe
# deb-src http://ap-northeast-2.ec2.archive.ubuntu.com/ubuntu/ focal universe
deb http://ap-northeast-2.ec2.archive.ubuntu.com/ubuntu/ focal-updates universe
# deb-src http://ap-northeast-2.ec2.archive.ubuntu.com/ubuntu/ focal-updates universe
 
## N.B. software from this repository is ENTIRELY UNSUPPORTED by the Ubuntu
## team, and may not be under a free licence. Please satisfy yourself as to
## your rights to use the software. Also, please note that software in
## multiverse WILL NOT receive any review or updates from the Ubuntu
## security team.
deb http://ap-northeast-2.ec2.archive.ubuntu.com/ubuntu/ focal multiverse
# deb-src http://ap-northeast-2.ec2.archive.ubuntu.com/ubuntu/ focal multiverse
deb http://ap-northeast-2.ec2.archive.ubuntu.com/ubuntu/ focal-updates multiverse
# deb-src http://ap-northeast-2.ec2.archive.ubuntu.com/ubuntu/ focal-updates multiverse
 
## N.B. software from this repository may not have been tested as
## extensively as that contained in the main release, although it includes
## newer versions of some applications which may provide useful features.
## Also, please note that software in backports WILL NOT receive any review
## or updates from the Ubuntu security team.
deb http://ap-northeast-2.ec2.archive.ubuntu.com/ubuntu/ focal-backports main restricted universe multiverse
# deb-src http://ap-northeast-2.ec2.archive.ubuntu.com/ubuntu/ focal-backports main restricted universe multiverse
 
## Uncomment the following two lines to add software from Canonical's
## 'partner' repository.
## This software is not part of Ubuntu, but is offered by Canonical and the
## respective vendors as a service to Ubuntu users.
# deb http://archive.canonical.com/ubuntu focal partner
# deb-src http://archive.canonical.com/ubuntu focal partner
 
deb http://security.ubuntu.com/ubuntu focal-security main restricted
# deb-src http://security.ubuntu.com/ubuntu focal-security main restricted
deb http://security.ubuntu.com/ubuntu focal-security universe
# deb-src http://security.ubuntu.com/ubuntu focal-security universe
deb http://security.ubuntu.com/ubuntu focal-security multiverse
# deb-src http://security.ubuntu.com/ubuntu focal-security multiverse
```


## Docker hub 이미지 pull
참고: https://hub.docker.com/_/buildpack-deps

```bash
#docker.sock 소켓 권한 추가(Linux)
sudo chmod 666 /var/run/docker.sock
```

```bash
docker login
```

```bash
docker pull buildpack-deps
```


## AWS ECR에 이미지 등록 (buildpack-deps : curl을 위함)

```bash
aws configure
```

```bash
AWS Access Key ID [None]: XXXXX
AWS Secret Access Key [None]: XXXXX
Default region name [None]: {REGION}
Default output format [None]:  json
```

```bash
#ECR login
aws ecr get-login-password --region {REGION} --no-verify-ssl | docker login --username AWS --password-stdin {ACCOUNT}.dkr.ecr.{REGION}.amazonaws.com
```


### buildpack-deps ECR 생성
```bash
aws ecr create-repository --repository-name buildpack-deps --image-scanning-configuration scanOnPush=true --region {REGION} --no-verify-ssl
```


### buildpack-deps ECR에 로컬 이미지 Push
```bash
#ECR 로그인 정보 확인
cat ~/.docker/config.json 

#로컬 이미지 확인
docker images

#이미지 이름 변경
docker tag buildpack-deps:latest {ACCOUNT}.dkr.ecr.{REGION}.amazonaws.com/buildpack-deps:atest
docker tag buildpack-deps:latest {ACCOUNT}.dkr.ecr.{REGION}.amazonaws.com/buildpack-deps:0.0.1

#이미지 변경확인
docker images

#이미지 변경확인
docker push {ACCOUNT}.dkr.ecr.{REGION}.amazonaws.com/buildpack-deps:0.0.1
```


## AWS Bastion을 통해 Cronjob 생성

### predictCronjob.yaml 파일
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: batch-predict
spec:
  schedule: "*/1 * * * *" #분 시 일 월 년
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: batch-predict
            image: {ACCOUNT}.dkr.ecr.{REGION}.amazonaws.com/buildpack-deps:0.0.1
            imagePullPolicy: IfNotPresent
            command:
            - /bin/sh
            - -c
            - curl https://{GATEWAY_ID}.execute-api.{REGION}.amazonaws.com/Prod/predict
          restartPolicy: OnFailure
```


### predictCronjob.yaml 로 Kubernetes Cronjob 생성
```bash
kubectl apply -f predictCronjob.yaml
```

### Kubernetes Cronjob 생성 시간에 Pod 확인 
```bash
kubectl get pods
```

```
NAME                                    READY   STATUS      RESTARTS   AGE
batch-predict-27668591-vp57l   0/1     Completed   0          31s
```


### Cronjob Pod 내 실행로그 확인

```bash
kubectl logs batch-predict-27668591-vp57l
```

```
% Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100    36  100    36    0     0      5      0  0:00:07  0:00:06  0:00:01     9
{"message": “Predict”}
```




