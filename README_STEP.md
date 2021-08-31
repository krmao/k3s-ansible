### 1. ssh 免密钥登录
```shell
# 服务器 k0
ssh-keygen -t rsa -C '767709667@qq.com' -f ~/.ssh/rsa_krmao_hw_1
ssh-copy-id -i ~/.ssh/rsa_krmao_hw_1.pub  -p 22 root@122.112.245.157
ssh -vvv -i ~/.ssh/rsa_krmao_hw_1 -p 22 root@122.112.245.157 # -vvv 登录打印调试信息
ssh -i ~/.ssh/rsa_krmao_hw_1 -p 22 root@122.112.245.157 # 登录

# 服务器 k1, 使用同一个 key 直接 copy
ssh-copy-id -i ~/.ssh/rsa_krmao_hw_1.pub  -p 22 root@119.3.86.212
ssh -i ~/.ssh/rsa_krmao_hw_1 -p 22 root@119.3.86.212 # 登录

# 使用 config快捷登录 k0/k1
cat ~/.ssh/config
#-----------------
Host k0
    HostName 122.112.245.157
    User root
    Port 22
    IdentityFile ~/.ssh/rsa_krmao_hw_1

Host k1
    HostName 119.3.86.212
    User root
    Port 22
    IdentityFile ~/.ssh/rsa_krmao_hw_1
#-----------------
ssh k0
exit
ssh k1
exit
```

### 2. install ansible on mac
1. 通过 pip 方式安装
    ```shell
    # 最好用 python3
    sudo chown -R $USER /Library/Python/2.7
    sudo curl https://bootstrap.pypa.io/pip/2.7/get-pip.py | python
    pip --version
    pip install -U ansible
    ansible --version
    
    # ERROR: Cannot uninstall 'pyparsing'. It is a distutils installed project and thus we cannot accurately determine which files belong to it which would lead to only a partial uninstall.
    # sudo pip install -I pyparsing==2.2.0
    # pip install -U ansible
    ```
2. 通过 brew 方式安装
    ```shell
    #region Mac更新完系统后使用homebrew报错:homebrew-core is a shallow clone.
    cd /usr/local/Homebrew/Library/Taps/homebrew
    rm -rf homebrew-core
    brew upgrade
    #endregion
    brew install ansible
    ansible --version
    ```
3. 通过 source 方式安装
    ```shell
    git clone https://github.com/ansible/ansible.git
    cd ansible
    sudo python setup.py install
    ansible --version
    ```

### 3. ansible examples
1. 配置 ansible hosts 以及免密登录
   > sudo mkdir /etc/ansible && sudo vi /etc/ansible/hosts
   ```shell
   [all:vars] # all:vars 全局变量
   ansible_ssh_user=root
   ansible_ssh_port=22
   ansible_connection=ssh
   ansible_ssh_private_key_file=~/.ssh/rsa_krmao_hw_1
   
   [master:vars] # *:vars 块变量
   ansible_python_interpreter=/usr/bin/python2
   
   [node:vars]
   ansible_python_interpreter=/usr/bin/python2
   
   [master] # 组
   k0 ansible_ssh_host=122.112.245.157
   
   [node] # 组
   k1 ansible_ssh_host=119.3.86.212
   ```
2. ansible 相关命令
   * ping all hosts
      > ansible all -m ping # -u root
   
   * command 模块 (不指定-m参数时, 使用的就是 command 模块)
      > ansible k0  -a 'pwd'
      > 
      > ansible all  -a "echo hello"
     
   * shell 模块
      > ansible k0  -a "ps -fe |grep sa_q" -m shell   
     
   * script 模块
      > ansible k0 -m script -a ~/workspace/k3s-ansible/ansible-learn/test.sh
     
   * playbook 模块
      > ansible-playbook ~/workspace/k3s-ansible/ansible-learn/test.yml # -u root

### 4. [k3s-ansible](https://www.cnblogs.com/k8ops/p/12943766.html#2102995432)
> [国内镜像地址下载](https://www.cnblogs.com/k3s2019/p/14339547.html)
1. 清理环境
    > ansible-playbook reset.yml -i inventory/my-cluster/hosts.ini -u ops_root -b -vv
2. 安装master/node
    > ansible-playbook site.yml -i inventory/my-cluster/hosts.ini -u ops_root -b -vv
3.  登录 master 检查 kubectl 命令是否成功安装
    > ssh k0
    ```shell
    # kubectl get pod -n kube-system
    NAME                                      READY   STATUS      RESTARTS   AGE
    local-path-provisioner-58fb86bdfd-df99p   1/1     Running   0          2m10s
    metrics-server-6d684c7b5-vm82v            1/1     Running   0          2m10s
    coredns-6c6bb68b64-7z75d                  1/1     Running   0          2m10s
    helm-install-traefik-bck6m                0/1     Error     4          2m10s
    
    # kubectl get nodes -o wide
    NAME         STATUS   ROLES    AGE   VERSION        INTERNAL-IP     EXTERNAL-IP       OS-IMAGE                KERNEL-VERSION                CONTAINER-RUNTIME
    krmao-hw-1   Ready    <none>   85s   v1.17.5+k3s1   192.168.0.57    119.3.86.212      CentOS Linux 7 (Core)   3.10.0-1160.15.2.el7.x86_64   containerd://1.3.3-k3s2
    krmao-hw-0   Ready    master   97s   v1.17.5+k3s1   192.168.0.169   122.112.245.157   CentOS Linux 7 (Core)   3.10.0-1160.15.2.el7.x86_64   containerd://1.3.3-k3s2
    ```
4. [配置 Ansible 控制机 MacOS 从外部访问 K3s 集群](https://habd.as/post/kubernetes-macos-k3s-k3d-rancher/)
    1. ansible 控制机 macos 安装 k3s/helm/kubectl 命令
        ```shell
        HOMEBREW_NO_AUTO_UPDATE=1 \
        brew install k3d helm@3 kubectl
        ```
    2. 拷贝 k3s 集群 master 下面的 /etc/rancher/k3s/k3s.yaml 到 ansible 控制机
        ```shell
        scp root@122.112.245.157:/etc/rancher/k3s/k3s.yaml ~/
        ```
        1. 修改 k3s.yaml server ip 指向 k3s 集群 master external ip
            ```shell
            vi ~/k3s.yaml
               server: https://127.0.0.1:6443
               server: https://122.112.245.157:6443
            wq
            ```
        2. 控制机从外部访问 k3s集群
            ```shell
            helm --kubeconfig ~/k3s.yaml ls --all-namespaces
            kubectl --kubeconfig ~/k3s.yaml get nodes -o wide --all-namespaces
            ```
    3. 设置 kubeconfig 环境变量, 配置指令别名用以简写指令
       ```shell
        export KUBECONFIG=~/k3s.yaml
        alias k=kubectl
        k get pod -n kube-system
        k get nodes -o wide --all-namespaces
       ```
### 参考
* [安装参考](https://www.yinnote.com/k3s-instal/)
* /etc/systemd/system/k3s.service
  > ./roles/k3s/master/templates/k3s.service.j2
* 实现跨云集群互联
  ```yaml
  #all.yml
  extra_server_args: "--node-external-ip {{ ansible_ssh_host }}" # 每一台 master 使用外网 ip 地址注册和互联, 实现跨云集群互联
  extra_agent_args: "--node-external-ip {{ ansible_ssh_host }}"  # 每一台 node   使用外网 ip 地址注册和互联, 实现跨云集群互联
  ```