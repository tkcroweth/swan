# FCP 部署
## 部署要求
- FCP 官方硬件要求：
- 拥有公网IP (没有公网IP可以用frp端口映射)
- 拥有域名
- 拥有 SSL 证书 (去ohttps.com搞免费的)
- 至少拥有一个 GPU (没有GPU也能跑，积分会减少一半)
- 至少 8 个 vCPU (建议12核以上，否则可能接不到任务)
- 至少 300GB SSD 存储
- 最低 64GB 内存 (32g也可以)
- 最低 50MB 带宽

## 相关链接
- 官方教程 https://docs.swanchain.io/orchestrator/as-a-computing-provider/fcp-fog-computing-provider/computing-provider-setup
- 官方discord https://discord.com/invite/swanchain
- 官方tw https://twitter.com/swan_chain
- 配置小狐狸钱包：网络、代币等 https://docs.swanchain.io/swan-testnet/atom-accelerator-race/before-you-get-started/set-up-metamask
- 领水+跨链桥 https://docs.swanchain.io/swan-testnet/atom-accelerator-race/before-you-get-started/claim-sepoliaeth
- 领水代币：https://docs.swanchain.io/swan-testnet/atom-accelerator-race/before-you-get-started/claim-testswan


## 基础环境
- 系统  ubuntu 22.04 
- golang 1.21.7
- 以下流程都使用root执行，非root用户 部分命令需要自行添加sudo

## 部署

### 安装显卡驱动
- 查看硬件驱动（选择返回结果中 recommended结尾的驱动） 
    ```
    ubuntu-drivers devices
    ```

- 安装驱动 
    ```
    apt install nvidia-driver-535 -y
    ```

- 查看显卡信息
    ```
    nvidia-smi
    ```

### 安装golang

- 国内环境可能需要挂代理或使用类似链接找到相应版本 https://studygolang.com/dl
    ```
    wget -c https://golang.org/dl/go1.21.7.linux-amd64.tar.gz -O - | sudo tar -xz -C /usr/local
    ```

- 加载环境变量
    ```
    echo "export PATH=$PATH:/usr/local/go/bin" >> ~/.bashrc && source ~/.bashrc
    ```

- 查看go版本
    ```
    go version
    ```


### 安装docker
- apt 安装docker
    ```bash
    apt update && apt install docker.io -y
    ```

- 设置docker的cgroupdriver为systemd
    ```
    cat > /etc/docker/daemon.json << EOF
      {
          "exec-opts": ["native.cgroupdriver=systemd"]
      }
    EOF
    ```

- 重启docker
    ```
    systemctl restart docker
    ```

### 安装cri-dockerd
- 下载包
    ``` 
    wget https://testnetcn.oss-cn-hangzhou.aliyuncs.com/src/cri-dockerd_0.3.3.3-0.ubuntu-jammy_amd64.deb
    ```

- 安装
    ```
    dpkg -i cri-dockerd_0.3.3.3-0.ubuntu-jammy_amd64.deb
    ```

- 查看cri-dockerd
    ```
    systemctl status cri-docker.service
    ```

### 安装k8s相关
- 安装依赖
    ``` 
    apt-get install -y apt-transport-https ca-certificates curl gpg
    ```

- 配置源
    ```
    curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
    echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
    ```

- 安装k8s相关
    ```
    apt-get update && apt-get install -y kubelet kubeadm kubectl
    ```

### 初始化k8s
- 关闭交换
    ``` 
    swapoff -a
    
    # 同时注释掉 /etc/fstab 文件里swap那一行
    ```

- 初始化
    ```
    # 国内环境可以使用阿里源 追加参数 --image-repository registry.aliyuncs.com/google_containers
    kubeadm init --pod-network-cidr=192.168.0.0/16 \
    --upload-certs \
    --control-plane-endpoint=服务器内网IP \
    --apiserver-advertise-address=服务器内网IP \
    --service-cidr=172.36.1.0/24 \
    --v=5 --cri-socket  unix:///var/run/cri-dockerd.sock
    
    # 初始成功后，输出内容包含下面3行，直接执行
    mkdir -p /$HOME/.kube
    cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    chown $(id -u):$(id -g) $HOME/.kube/config
    ```

- 安装K8S网络插件
    ```
    kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.1/manifests/tigera-operator.yaml
    kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.1/manifests/custom-resources.yaml
    ```

- 查看pod运行状态，等待所有为running状态 
    ```
    watch kubectl get pods -n calico-system
    
    # 输出如下
    # NAME                                       READY   STATUS    RESTARTS   AGE
    # calico-kube-controllers-6b78568556-9xq4d   1/1     Running   1          30h
    # calico-node-lfdgg                          1/1     Running   1          30h
    # calico-typha-55b7458bb-2nffn               1/1     Running   1          30h
    # csi-node-driver-p747w                      2/2     Running   2          30h
    ```

- 去除k8s master节点污点
    ```
    kubectl taint nodes --all node-role.kubernetes.io/control-plane-
    kubectl taint nodes --all node-role.kubernetes.io/master-
    ```
- k8s相关命令
    ```
    # 查看节点
    kubectl get nodes
    
    kubectl describe nodes 主机名
    
    kubectl get pod -A 
    
    kubectl describe pod -n xxx  xxx 
    
    journalctl -u kubelet -f
    ```


### 安装k8s 插件

- 官方文档 
    - https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html#nvidia-drivers
    - https://github.com/NVIDIA/k8s-device-plugin#quick-start
 

- 下载并安装
    ```
    distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
    
    curl -s -L  https://nvidia.github.io/libnvidia-container/gpgkey | apt-key add -
    
    curl -s -l https://nvidia.github.io/libnvidia-container/$distribution/libnvidia-container.list  | tee /etc/apt/sources.list.d/libnvidia-container.list
    
    apt-get update && apt-get install -y nvidia-container-toolkit 
    ```

- 配置docker
    ```
    cat > /etc/docker/daemon.json << EOF
    {
        "exec-opts": ["native.cgroupdriver=systemd"],
        "default-runtime": "nvidia",
        "runtimes": {
            "nvidia": {
                "path": "/usr/bin/nvidia-container-runtime",
                "runtimeArgs": []
            }
        }
    }
    EOF
    
    # 重启docker
    systemctl restart docker
    ```

- 启动插件
    ```
    kubectl create -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v0.15.0/deployments/static/nvidia-device-plugin.yml
    
    # 检查插件运行情况，输出结果
    kubectl get po -n kube-system |grep -i nvidia
    
    # 预期输出如下
    # nvidia-device-plugin-daemonset-qkklz   1/1     Running   60         64d
    ```

- 测试gpu调度是否正常
    ```
    cat > gpu-pod-test.yaml << EOF
    apiVersion: v1
    kind: Pod
    metadata:
      name: gpu-pod
    spec:
      restartPolicy: Never
      containers:
        - name: cuda-container
          image: nvcr.io/nvidia/k8s/cuda-sample:vectoradd-cuda10.2
          resources:
            limits:
              nvidia.com/gpu: 1 # requesting 1 GPU
      tolerations:
      - key: nvidia.com/gpu
        operator: Exists
        effect: NoSchedule
    EOF
    
    # 启动pod
    kubectl apply -f gpu-pod-test.yaml
    
    # 查看日志
    kubectl logs gpu-pod
    
    # 正常输出如下：
    # [Vector addition of 50000 elements]
    # Copy input data from the host memory to the CUDA device
    # CUDA kernel launch with 196 blocks of 256 threads
    # Copy output data from the CUDA device to the host memory
    # Test PASSED
    # Done
    ```

### 安装 Ingress-nginx 
- 创建Ingress-nginx 
    ``` 
    kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.7.1/deploy/static/provider/cloud/deploy.yaml

    # 查看运行状态
    kubectl get pod -n ingress-nginx

    # 查看ingress-nginx端口:
    kubectl get svc -n ingress-nginx
    # 正常输出如下
    # NAME                                 TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE
    # ingress-nginx-controller             LoadBalancer   172.36.1.62    <pending>     80:32293/TCP,443:32540/TCP   31h
    # ingress-nginx-controller-admission   ClusterIP      172.36.1.194   <none>        443/TCP                      31h
    
    # 需要记录端口
    # 80:32293/TCP，其中<32293>接下来搭建nginx反向代理需要用到（端口因人而异）
    ```


### 域名解析

- 创建泛域名解析
    - 记录类型 A记录 
    - 主机记录 *.domain.com     // 也可使用二级域名
    - 记录值   x.x.x.x         // 你的公网ip
- 验证域名解析
    ``` 
    ping a.domain.com
    ping b.domain.com
    
    # 返回结果应是你的公网ip
    ```
- 证书
    - 免费签发网址 ohttps.com 
    - 依照网站要求添加cname验证域名，然后领取证书
    - 证书传到服务器备用
    
### 安装nginx
- 安装
    ```
    sudo apt update
    sudo apt install nginx -y
    ```
- 配置
    ```
    vim /etc/nginx/conf.d/swan.conf 
    ```
    
    ```
    # 配置如下
    map $http_upgrade $connection_upgrade {
        default upgrade;
        ''      close;
    }
    
    server {
            listen 80;
            listen [::]:80;
            server_name *.domain.com; 
            return 301 https://$host$request_uri;
            #client_max_body_size 1G;
    }
    server {
            listen 443 ssl;
            listen [::]:443 ssl;
            ssl_certificate  /etc/nginx/ssl/example.com/fullchain.pem;     # 修改为你的ssl证书文件所在路径
            ssl_certificate_key  /etc/nginx/ssl/example.com/privkey.key;   # 修改为你的ssl证书文件所在路径
    
            server_name *.domain.com;
            location / {
              proxy_pass http://127.0.0.1:<port>;  # <port>使用 Ingress-nginx 记录的端口
              proxy_set_header Host $http_host;
              proxy_set_header Upgrade $http_upgrade;
              proxy_set_header Connection $connection_upgrade;
           }
    }
    ```

- 重启
   ``` 
   nginx -t
   
   
   nginx -s reload
   
   
   # 浏览器访问改域名验证解析是否成功
   # https://xxx.domain.com
   ```
    
### 安装resource-exporter
- 硬件配置汇报插件
    ``` 
    # 配置文件
    cat <<EOF | kubectl apply -f -
    apiVersion: apps/v1
    kind: DaemonSet
    metadata:
      namespace: kube-system
      name: resource-exporter-ds
      labels:
        app: resource-exporter
    spec:
      selector:
        matchLabels:
          app: resource-exporter
      template:
        metadata:
          labels:
            app: resource-exporter
        spec:
          containers:
          - name: resource-exporter
            image: filswan/resource-exporter:v11.2.6
            imagePullPolicy: IfNotPresent
    EOF
    
    # 查看运行中的pod
    kubectl get po -n kube-system
    
    # 输出结果应该看到如下内容
    # resource-exporter-ds-xxxx running
    ```

### 安装redis
- 安装redis
    ```
    sudo apt update
    
    sudo apt install redis-server
    
    systemctl start redis-server.service
 
    ```
    
    
### 安装 computing-provider
- 二进制安装
    ```
    # 创建目录
    mkdir -p /data/swan/ && cd /data/swan/
    
    # 下载二进制文件
    wget https://github.com/swanchain/go-computing-provider/releases/download/v0.4.7/computing-provider
    
    # 授予权限，创建链接
    chmod +x computing-provider
    ln -s /data/swan/computing-provider /usr/local/bin/
    
    # 查看版本
    computing-provider -v
    ```
- 配置CP
    ``` 
    # 下载配置模板
    cd /data/swan/ 
    wget https://testnetcn.oss-cn-hangzhou.aliyuncs.com/src/config.toml.sample
    mv config.toml.sample  config.toml
    ```
    
    修改配置文件，内容如下
    ```
    [API]
    Port = 8085                                                 # The port number that the web server listens on
    MultiAddress = "/ip4/<公网地址>/tcp/8085"                    # The multiAddress for libp2p  # 这里填写服务器公网IP地址(如果使用frp映射，则填写frp服务端的IP地址
    Domain = "cp.baidu.com"                                     # The domain name  # 你解析的泛域名，假如解析的是 *.cp.baidu.com，则需要填写cp.baidu.com
    NodeName = "cp.baidu.com"                                   # The computing-provider node name  # 要显示在dashboard上的节点名，随意填写
    
    RedisUrl = "redis://127.0.0.1:6379"                         # The redis server address
    RedisPassword = ""                                          # The redis server access password  # Redis默认监听本机，且无密码，无需设置
    
    [UBI]
    UbiTask = true                                              # Accept the UBI task (Default: true)
    UbiEnginePk = "0xB5aeb540B4895cd024c1625E146684940A849ED9"  # UBI Engine's public key, CP only accept the task from this UBI engine 
    UbiUrl ="https://ubi-task.swanchain.io/v1"                  # UBI Engine's API address
    
    [LOG]
    CrtFile = "/etc/nginx/cert/example.com/fullchain.pem"       # Your domain name SSL .crt file path  # 泛域名证书，和nginx上配置的证书一样。
    KeyFile = "/etc/nginx/cert/example.com/privkey.key"         # Your domain name SSL .key file path  # 泛域名证书，和nginx上配置的证书一样。
    
    [HUB]
    ServerUrl = "https://orchestrator-api.swanchain.io"         # The Orchestrator's API address
    AccessToken = "注释里的网站去获取-点击右上角头像-展示key-没有则点击创建"                            # The Orchestrator's access token, Acquired from "https://orchestrator.swanchain.io" 
    WalletAddress = "使用computing-provider命令创建或导入的钱包地址,可使用computing-provider wallet list或computing-provider info查看"                                  # The cp‘s wallet address
    BalanceThreshold= 0.01                                      # The cp’s collateral balance threshol
    OrchestratorPk = "0x29eD49c8E973696D07E7927f748F6E5Eacd5516D" 
    VerifySign = true  
    
    [MCS]                                                       # 相关网站使用钱包登录时，需要钱包有0.01ETH或者10个MATIC，10个MATIC更简单，交易所充值即可
    ApiKey = "注释里网站去创建key"                                # Acquired from "https://www.multichain.storage" -> setting -> Create API Key
    BucketName = "后面的去创建bucket"                             # Acquired from "https://www.multichain.storage" -> bucket -> Add Bucket
    Network = "polygon.mainnet"                                 # polygon.mainnet for mainnet, polygon.mumbai for testnet
    FileCachePath = "/tmp"                                      # Cache directory of job data
    
    [Registry]
    ServerAddress = ""                            # The docker container image registry address, if only a single node, you can ignore  # 无需设置
    UserName = ""                                 # The login username, if only a single node, you can ignore
    Password = ""                                 # The login password, if only a single node, you can ignore
    
    [RPC]
    SWAN_TESTNET ="https://rpc-proxima.swanchain.io"  # Swan testnet RPC
    SWAN_MAINNET= ""
    
    [CONTRACT]
    SWAN_CONTRACT="0x91B25A65b295F0405552A4bbB77879ab5e38166c"   # Swan token's contract address
    SWAN_COLLATERAL_CONTRACT="0xfD9190027cd42Fc4f653Dfd9c4c45aeBAf0ae063"   # Swan's collateral address
    ```

- 配置环境变量
    ``` 
    echo 'export CP_PATH=/data/swan' >> /etc/profile
    echo 'export FIL_PROOFS_PARAMETER_CACHE=/var/tmp/filecoin-proof-parameters' >> /etc/profile
    source /etc/profile  &&  echo $CP_PATH && echo $FIL_PROOFS_PARAMETER_CACHE
    ```

### 安装AI推理依赖
- 安装
    ``` 
    # 创建目录
    mkdir -p /data/swan/src  &&  cd /data/swan/src
    
    # 下载并解压
    wget https://github.com/swanchain/go-computing-provider/archive/refs/tags/v0.4.6.tar.gz
    tar xvf v0.4.6.tar.gz
    
    # 执行安装
    cd go-computing-provider-0.4.6/
    ./install.sh
    
    ```
    
### 下载UBI文件
- 下载lotus-shed
    ``` 
    cd /data/swan
    
    # 安装hwloc
    apt install hwloc -y
    
    # 下载lotus-shed可执行文件
    wget https://github.com/swanchain/ubi-benchmark/releases/download/v0.0.1/lotus-shed
    
    chmod +x lotus-shed
    ln -s /data/swan/lotus-shed /usr/local/bin/
    ```   

- 下载UBI
    ```
    # 文件较大，使用后台命令下载（ssh客户端终端后可以重连）
    # 创建 screen
    screen -S  ubi
    
    # 退出screen ctrl + A + D
    
    # 查看 screen 列表
    screen -ls
    
    # 进入指定screen
    screen -r ubi

    # 安装依赖
    apt-get install libhwloc-dev libhwloc5 -y
    
    # 开始下载，可以断线重连，避免失败后重新下载 要求至少160G磁盘空间
    lotus-shed fetch-params --proving-params 512MiB  &&  lotus-shed fetch-params --proving-params 32GiB

    # 如果下载速度过慢的话，可以ctrl+c 退出上面的命令，执行如下命令后重试
    export IPFS_GATEWAY=https://proof-parameters.s3.cn-south-1.jdcloud-oss.com/ipfs/ 
    export FIL_PROOFS_PARAMETER_CACHE=/var/tmp/filecoin-proof-parameters

    ```

- 配置fil-c2
    ``` 
    # 查看显卡型号 cores 
    # https://github.com/filecoin-project/bellperson?tab=readme-ov-file#supported--tested-cards
    
    # 创建配置文件（自行修改 RUST_GPU_TOOLS_CUSTOM_GPU=<你的型号:cores>）
    cat > fil-c2.env << EOF
    FIL_PROOFS_PARAMETER_CACHE="/var/tmp/filecoin-proof-parameters"
    RUST_GPU_TOOLS_CUSTOM_GPU="GeForce RTX 4090:16384"
    EOF
    
    # 没有gpu 需要将第二行修改为一下内容，否则无法接收任务
    BELLMAN_NO_GPU=1
    
    ```

### 创建钱包
- 创建钱包
    ```
    # 创建钱包
    computing-provider wallet new
    
    # 列出钱包：
    computing-provider wallet list
    
    # 导出钱包私钥:
    computing-provider wallet export 0x钱包地址
    ```
- 导入钱包
    ```
    # 导入已有的钱包，小狐狸的即可，private.key文件写入钱包私钥
    computing-provider wallet import private.key
    ```
    
- 领水
    
    - 领水+跨链桥参考该网址
    
        https://docs.swanchain.io/swan-testnet/atom-accelerator-race/before-you-get-started/claim-sepoliaeth
        
    - 跨链后需要等待几分钟
    - 查看余额
        ```
        computing-provider wallet list
        ```
- 初始化CP
    ```
    # 创建账户 0x钱包地址 使用你创建/导入的地址
    computing-provider account create --ownerAddress 0x钱包地址
    
    
    # 正常输出内容如下
    Contract deployed! Address: 0x3091c9647Ea5248079273B52C3707c958a3f2658
    Transaction hash: 0xb8fd9cc9bfac2b2890230b4f14999b9d449e050339b252273379ab11fac15926
    The height of the block: 44900354
    
    # 查看CP信息
    computing-provider info
    
    # 质押ETH （应该质押超过1个，否则会报错余额不足）
    computing-provider collateral add <0x钱包地址> <质押数量>
    # 比如
    computing-provider collateral add <0x钱包地址> 1
    
    # 启动CP节点
    nohup computing-provider run >> cp.log 2>&1 &    # 启动并放入后台
    
    # 查看运行日志
    tail -f /data/swan/cp.log

    # 以下命令仅供参考
    # 重启CP进程
    ps aux | pgrep computing-provider | xargs kill
    nohup computing-provider run >> cp.log 2>&1 &
    
    # 运行正常的话，几分钟后可以官网看到自己的节点，dashboard自行搜索
    https://orchestrator.swanchain.io/provider-status

    ```
    
### 验证CP
- 查看普通任务（启动一段时间，大概1小时以上，可以接收任务）
    ```
    # 查看任务列表
    computing-provider task list

    # 查看任务详情
    computing-provider task list -v
    computing-provider task get 任务ID
    
    # 查看ubi任务
    computing-provider ubi list
    
    ```
    

