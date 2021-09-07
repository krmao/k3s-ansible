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
    > ansible-playbook reset.yml -i inventory/my-cluster/hosts.ini -u root -b -vv
2. 安装master/node
    > ansible-playbook site.yml -i inventory/my-cluster/hosts.ini -u root -b -vv
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
    > [参考 使用 kubectl 从外部访问集群](https://docs.rancher.cn/docs/k3s/cluster-access/_index)
    1. ansible 控制机 macos 安装 k3s/helm/kubectl 命令
        ```shell
        HOMEBREW_NO_AUTO_UPDATE=1 \
        brew install k3d helm@3 kubectl
        ```
    2. 拷贝 k3s 集群 master 下面的 /etc/rancher/k3s/k3s.yaml 到 ansible 控制机
        ```shell
        scp root@122.112.245.157:/etc/rancher/k3s/k3s.yaml ~/ && sed -i 's/127.0.0.1/122.112.245.157/g' ~/k3s.yaml && cat ~/k3s.yaml
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
        vi ~/.bash_profile
            KUBECONFIG=~/k3s.yaml
            PATH=$PATH:$KUBECONFIG
            export KUBECONFIG
            alias k=kubectl
            export PATH
        source ~/.bash_profile

        k get pod -n kube-system
        k get nodes -o wide --all-namespaces
       ```
5. [Kubernetes Dashboard](https://rancher.com/docs/k3s/latest/en/installation/kube-dashboard/)
    ```shell
    # 在 ansible 控制机执行以下命令
   
    # Deploying the Kubernetes Dashboard
    GITHUB_URL=https://github.com/kubernetes/dashboard/releases
    VERSION_KUBE_DASHBOARD=$(curl -w '%{url_effective}' -I -L -s -S ${GITHUB_URL}/latest -o /dev/null | sed -e 's|.*/||')
    k create -f https://raw.githubusercontent.com/kubernetes/dashboard/${VERSION_KUBE_DASHBOARD}/aio/deploy/recommended.yaml
    
    # Dashboard RBAC Configuration
    # cd ./dashboard
    dashboard.admin-user.yml
        apiVersion: v1
        kind: ServiceAccount
        metadata:
        name: admin-user
        namespace: kubernetes-dashboard
     
    dashboard.admin-user-role.yml
        apiVersion: rbac.authorization.k8s.io/v1
        kind: ClusterRoleBinding
        metadata:
        name: admin-user
        roleRef:
        apiGroup: rbac.authorization.k8s.io
        kind: ClusterRole
        name: cluster-admin
        subjects:
        - kind: ServiceAccount
          name: admin-user
          namespace: kubernetes-dashboard
   
    k create -f dashboard.admin-user.yml -f dashboard.admin-user-role.yml
   
    # Obtain the Bearer Token
    k -n kubernetes-dashboard describe secret admin-user-token | grep '^token'
   
    # Local Access to the Dashboard
    k proxy
   
    # The Dashboard is now accessible at:
    http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/
    Sign In with the admin-user Bearer Token
    ```
6. 安装 K8S IDE [lens](https://k8slens.dev/)
    ```shell
    # 添加集群 -> add cat k3s.yaml -> 固定集群到菜单
    # 右击集群 -> Lens Metrics -> Enable bundled Prometheus metrics stack -> apply
    ```
7. [Containerd 国内加速源](https://blog.csdn.net/xs20691718/article/details/106515605)
    ```shell
    ssh k0
    cd /var/lib/rancher/k3s/agent/etc/containerd/
    sudo crictl info | grep mirror
    cp config.toml config.toml.tmpl
    
    vi config.toml.tmpl
        # 在 config.toml.tmpl 文件中添加
        [plugins.cri.registry.mirrors]
          [plugins.cri.registry.mirrors."docker.io"]
            endpoint = ["https://docker.mirrors.ustc.edu.cn"]
    
    systemctl restart k3s
    sudo crictl info | grep mirror
    ```
8. [helm install traefik pod 失败](https://github.com/k3s-io/k3s/issues/1332)
    > 注意 systemctl restart k3s 重启服务后依然会报错, 可直接使用步骤9的方法禁用该默认组件
    ```shell
    ssh k0
    mv /var/lib/rancher/k3s/server/manifests/traefik.yaml /tmp/traefik.yaml
    kubectl -n kube-system delete helmcharts.helm.cattle.io traefik
    kubectl delete -n kube-system service traefik-dashboard
    kubectl delete -n kube-system ingress traefik-dashboard
    mv /tmp/traefik.yaml /var/lib/rancher/k3s/server/manifests/traefik.yaml
    ```
9. [k3s no deploy default traefik](https://github.com/k3s-io/k3s/issues/1160)
    ```yaml
    # all.yml
    extra_server_args: "--no-deploy traefik"
   
    # reinstall
    ansible-playbook reset.yml -i inventory/my-cluster/hosts.ini -u root -b -v
    ansible-playbook site.yml -i inventory/my-cluster/hosts.ini -u root -b -v
   
    # detect and will find no about traefik
    ssh k0
    kubectl get pod -n kube-system -o wide
    # NAME                                      READY   STATUS    RESTARTS   AGE     IP          NODE         NOMINATED NODE   READINESS GATES
    # metrics-server-6d684c7b5-99ccw            1/1     Running   2          5m20s   10.42.0.3   krmao-hw-1   <none>           <none>
    # local-path-provisioner-58fb86bdfd-4vcz5   1/1     Running   2          5m20s   10.42.0.2   krmao-hw-1   <none>           <none>
    # coredns-6c6bb68b64-tp2g9                  1/1     Running   0          5m20s   10.42.1.2   krmao-hw-0   <none>           <none>
    
    # Unable to connect to the server: x509: certificate signed by unknown authority
    # 重新执行步骤 4.4.2 即 k3s.yaml 文件中包含新的证书, 重新安装 k3s 集群后不更新本地证书就有可能报错
    # 届时 lens 软件也需要
    ```
10. [k3s install traefik by self](https://doc.traefik.io/traefik/getting-started/install-traefik/)
    ```shell
    helm repo add traefik https://helm.traefik.io/traefik
    helm repo update
    
    # install in default namespace
    helm install traefik traefik/traefik
    # helm uninstall traefik --namespace=default
    
    # install in a dedicated namespace
    kubectl create ns traefik-v2
    # Install in the namespace "traefik-v2"
    helm install traefik traefik/traefik --namespace=traefik-v2
    
    # uninstall with namespace
    # helm uninstall traefik --namespace traefik-v2
    
    # https://community.traefik.io/t/traefik-pod-not-found-when-using-kubectl-port-forward-for-dashboard-access/9895
    kubectl port-forward --namespace traefik-v2 $(kubectl get pods --selector "app.kubernetes.io/name=traefik" --output=name --namespace traefik-v2) 9000:9000
    Accessible with the url: http://127.0.0.1:9000/dashboard/
    ```
11. lens with kube-state-metrics
    ```shell
    kubectl create ns lens-metrics
    
    helm install kube-state-metrics bitnami/kube-state-metrics --namespace=lens-metrics
    helm install node-exporter      bitnami/node-exporter      --namespace=lens-metrics
    helm install prometheus         bitnami/prometheus         --namespace=lens-metrics
    helm install prometheus         bitnami/prometheus-operator         --namespace=lens-metrics
    
    kubectl port-forward --namespace lens-metrics svc/kube-state-metrics 8080:8080
    Accessible with the url: http://127.0.0.1:8080/
    
    # 一键重装 start
    # 重置失败尝试重启后再试 ssh k0 -> reboot, ssh k1 -> reboot
    # ansible-playbook reset.yml -i inventory/my-cluster/hosts.ini -u root -b -v
    # ansible-playbook site.yml -i inventory/my-cluster/hosts.ini -u root -b -v && scp root@122.112.245.157:/etc/rancher/k3s/k3s.yaml ~/ && sed -i '.bak' 's/127.0.0.1/122.112.245.157/g' ~/k3s.yaml && cat ~/k3s.yaml && k get pod -o wide --all-namespaces
    # 一键重装 end
    
    helm uninstall kube-state-metrics --namespace lens-metrics
    helm uninstall node-exporter      --namespace lens-metrics
    
    # k get pod -o wide --all-namespaces
    k delete -n lens-metrics pod prometheus-0  --force --grace-period=0 # 删除后自动重装
    ```
12. [PROMETHEUS](http://localhost:61762/classic/graph)
### 参考
* [DOCKER HUB](https://registry.hub.docker.com/search?q=&type=image)
* [阿里云镜像仓库](https://cr.console.aliyun.com/cn-qingdao/instance/dashboard)
* [ANSIBLE 官方文档](https://docs.ansible.com/ansible/latest/index.html)
* [ANSIBLE 中文文档1](http://www.ansible.com.cn/index.html)
* [ANSIBLE 中文文档2](https://www.w3cschool.cn/automate_with_ansible/automate_with_ansible-db6727oq.html)
* [K3S 官方中文文档](https://docs.rancher.cn/k3s/)
* [K3S 安装参考](https://www.yinnote.com/k3s-instal/)
* [K3S Helm 包管理工具](https://docs.rancher.cn/docs/k3s/helm/_index/)
* [K3S 使用 kubectl 从外部访问集群](https://docs.rancher.cn/docs/k3s/cluster-access/_index)
* [K8S 官方中文文档](https://kubernetes.io/zh/docs/home/)
* IDEA 可以安装 Ansible 插件
* 实现跨云集群互联
  ```yaml
  #all.yml
  extra_server_args: "--node-external-ip {{ ansible_ssh_host }}" # 每一台 master 使用外网 ip 地址注册和互联, 实现跨云集群互联
  extra_agent_args: "--node-external-ip {{ ansible_ssh_host }}"  # 每一台 node   使用外网 ip 地址注册和互联, 实现跨云集群互联
  ```