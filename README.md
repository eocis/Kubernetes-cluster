## Compose Kubernetes Cluster

> ### UTM Image Links(for Mac M1)
- Master Node: 
- Worker Node 1:
- Worker Node 2:

> ### 구성환경

- Server: Laptop - Macbook Air
- OS: OSX 12.0.1(21A559) - Monterey
- Spec
    - CPU: M1(ARM)
    - RAM: 16G
    - SSD: 256G
- Hypervisor: UTM
- Kubernetes:
    - Container Runtime: Docker
    - CNI: Flannel

<br>

- Cluster 구성(3 VM)
    - <b>Master Node</b>(Ubuntu Server 20.04 LTS - arm64)
        - RAM: 4G
        - HDD: 40G
        - IP: [ 192.168.64.10/24, 10.244.0.1/16 ]
        - User: master
        - Password: master
        <br></br>
    - <b>Worker Node</b>(Ubuntu Server 20.04 LTS - arm64)
        - RAM: 2G
        - HDD: 30G
        - IP: [ 192.168.64.11/24, 10.244.0.2/16 ]
        - User: worker1
        - Password: worker1
        <br></br>
    - <b>Worker Node</b>(Ubuntu Server 20.04 LTS - arm64)
        - RAM: 2G
        - HDD: 30G
        - IP: [ 192.168.64.12/24, 10.244.0.3/16 ]
        - User: worker2
        - Password: worker2
        <br></br>

> ### 설치 구현

1. Download Ubuntu Server Image
2. Create UTM VM
    - 시스템: ARM64(aarch64))
    - 드라이브: Import Drive - ISO Attach
3. VM Run
4. Installation ubuntu
    - don't change mirror
    - install openssh server
5. Change Ubuntu DHCP IP(DHCP - static)
    - sudo vim /etc/netplan/00-installer-config.yaml
    ```yaml
    # This is the network config written by 'subiquity'
    network:
    ethernets:
        enp0s9:
        dhcp4: true
        addresses:
                - [ 192.168.64.10/24, 10.244.0.1/16 ] # VM Network & Kubernetes Network
    version: 2
    ```
6. Edit apt repository 
    - sudo vim /etc/apt/source.list
    - :%s/deb/deb [arch=arm64]

```sh
# 7. apt update
$ sudo apt update && apt upgrade -y

# 8. Install Container Runtime(Docker)
$ apt-transport-https ca-certificates curl software-properties-common gnupg2

# 9. Add Docker GPG Key
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

# 10. Add Docker apt repository
$ sudo add-apt-repository \
"deb https://download.docker.com/linux/ubuntu \
$(lsb_release -cs) \
stable"

# 11. Install Docker CE
$ sudo apt update && sudo apt install containerd.io docker-ce docker-ce-cli

# 12. Set Container Runtime(Set cgroup Driver)
$ sudo mkdir -p /etc/systemd/system/docker.service.d
$ sudo su -
cat >> /etc/docker/daemon.json
{
"exec-opts": ["native.cgroupdriver=systemd"],
"log-driver": "json-file",
"log-opts": {
    "max-size": "100m"
},
"storage-driver": "overlay2"
}

# 13. Restart Docker
$ sudo systemctl daemon-reload
$ sudo systemctl restart docker
```

14. Common Setting(Master, Worker)
    ```sh
    $ sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
    $ echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list # Add kubernetes apt repository
    $ sudo apt install -y kubelet kubeadm kubectl
    $ sudo apt-mark hold kubelet kubeadm kubectl    # Hold Version
    ```

15. Set Master Node
    ```sh
    $ ip a  # check ip
    $ sudo kubeadm init --apiserver-advertise-address=<Master Node IP> --pod-network-cidr=<10.244.0.0/16># Copy "kubeadm join ~~~~"
    $ mkdir -p $HOME/.kube
    $ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    $ sudo chown $(id -u):$(id -g) $HOME/.kube/config
    ```

16. Set Cluster Network(Cluster Networking Interfaces, CNI)
    ```sh
    $ curl -O https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml # Default Pod network: 10.244.0.0/16
    $ kubectl apply -f kube-flannel.yml
    ```

17. Set Worker Nodes
    ```sh
    $ sudo kubeadm init
    $ scp master@192.168.64.10:.kube/config ~/.kube/config  # get config file
    $ kubeadm join 192.168.64.10:6443 --token nznopm.t0yxwct0rfv1fw9r --discovery-token-ca-cert-hash sha256:80976183c88405785848c2182a18a8e0acccdb7552152514c721400cf8eecc9d # Copied Chapter 15 command(아래 cluster join항목 참조)
    ```

18. Working Test
    ```sh
    # All Nodes
    $ kubectl get nods
    $ kubectl get all -A
    $ kubectl apply -f ./test_app/app.yaml  # replicaset 2
    $ kubectl describe <pod> -n kube-example | grep '10.244.0'
    ```
---

- Cluster Join(Worker Node)
    ```sh
    ## Master Node ##
    # 발행된 Token 확인
    $ kubeadm token list

    # 토큰이 없을 시
    $ kubeadm token create

    # 토큰 Hash값 확인
    $ openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'

    ## Worker Node ##
    # Join
    $ kubeadm join <kubernetes api server:PORT> --token <Token value> --discovery-token-ca-cert-hash sha256:<Hash value>
    ```

---

<참고>
- Installation: https://kubernetes.io/ko/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
- Conatiner-Runtime: https://kubernetes.io/ko/docs/setup/production-environment/container-runtimes/
- cgroup-Driver: https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/configure-cgroup-driver/
- CNI: https://kubernetes.io/ko/docs/concepts/cluster-administration/networking/
- Flannel: https://github.com/flannel-io/flannel#flannel
