# Deploy a Production Ready Kubernetes Cluster

![Kubernetes Logo](https://raw.githubusercontent.com/kubernetes-sigs/kubespray/master/docs/img/kubernetes-logo.png)

### 运行环境
* CentOS7.6+
* python3.8+

### 安装依赖
```ShellSession
# 拉取源码
git clone -b release-2.19-cn https://github.com/longtds/kubespray.git
cd kubespray
# 安装依赖包
pip3 install -r requirements.txt
```
### 在线安装
和原来的安装方式一样，已经把部分很难访问的地址修改为国内镜像源地址
```ShellSession
# Copy ``inventory/sample`` as ``inventory/mycluster``
cp -rfp inventory/sample inventory/mycluster

# Update Ansible inventory file with inventory builder
declare -a IPS=(10.10.1.3 10.10.1.4 10.10.1.5)
CONFIG_FILE=inventory/mycluster/hosts.yaml python3 contrib/inventory_builder/inventory.py ${IPS[@]}

# Review and change parameters under ``inventory/mycluster/group_vars``
cat inventory/mycluster/group_vars/all/all.yml
cat inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml

# Deploy Kubespray with Ansible Playbook - run the playbook as root
# The option `--become` is required, as for example writing SSL keys in /etc/,
# installing packages and interacting with various systemd daemons.
# Without --become the playbook will fail to run!
ansible-playbook -i inventory/mycluster/hosts.yaml  --become --become-user=root cluster.yml
```

### 离线安装
先通过ansible主机拉取所需软件和镜像，然后配置成cache模式离线部署
```ShellSession
# ansible主机和k8s节点操作系统及版本要一致，确保获取的软件依赖包可兼容
# 开启ansible主机yum cache，以便于打包依赖的rpm包
sed -i 's/keepcache=0/keepcache=1/g' /etc/yum.conf

# 在ansible主机上运行一遍安装过程，完成后集群所需的文件缓存到/tmp/kubespray_cache目录下
ansible-playbook -i inventory/local/hosts.yaml  --become --become-user=root cluster.yml

# 卸载ansible主机上的环境
ansible-playbook -i inventory/local/hosts.yaml  --become --become-user=root reset.yml

# 收集安装过程中系统需要安装的rpm包到/tmp/kubespray_cache/rpms目录下
mkdir -p /tmp/kubespray_cache/rpms
for i in `find /var/cache/yum/ -name *.rpm`; do cp $i /tmp/kubespray_cache/rpms; done 

# 拷贝inventory文件
cp -rfp inventory/sample inventory/mycluster

# 修改部署模式为强制cache模式
sed -i "s/download_force_cache: false/download_force_cache: true/g" inventory/mycluster/group_vars/all/all.yml
sed -i "s/download_run_once: true/download_run_once: false/g" inventory/mycluster/group_vars/all/all.yml

# 配置部署节点IP
declare -a IPS=(10.10.1.3 10.10.1.4 10.10.1.5)
CONFIG_FILE=inventory/mycluster/hosts.yaml python3 contrib/inventory_builder/inventory.py ${IPS[@]}

# 配置ansible主机免密登录所有节点IP
# 所有节点部署系统依赖的rpm包
ansible -i inventory/mycluster/hosts.yaml all -m copy -a "src=/tmp/kubespray_cache/rpms/ dest=/tmp/rpms/"
ansible -i inventory/mycluster/hosts.yaml all -m shell -a "yum localinstall -y /tmp/rpms/*.rpm"

# 开始部署
ansible-playbook -i inventory/mycluster/hosts.yaml  --become --become-user=root cluster.yml
```