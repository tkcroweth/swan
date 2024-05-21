# SWAN 计算提供者节点部署教程
## 介绍
项目方主活动：https://docs.swanchain.io/swan-testnet/atom-accelerator-race

竞赛节点分为ECP和FCP，ECP比较简单，FCP要复杂一些。

## ECP 

### 前置要求
- 配置（官方要求）
    ```拥有公网IP
  至少拥有一个 GPU
  至少 4 个 vCPU
  至少 300GB 硬盘存储
  最低 32GB 内存
  最低 20MB 带宽
 ```
  
- 系统
 - ubuntu版本22.04
  作者使用的aliyun ecs，请务必不要使用共享型实例
 - 映射本地端口到公网，公网端口自定义
  <Intranet_IP>:<9085> --> <Public_IP>:<PORT>
 - 领水&充值
    ```配置小狐狸钱包：网络、代币等
  https://docs.swanchain.io/swan-testnet/atom-accelerator-race/before-you-get-started/set-up-metamask
  领水+跨链桥
  https://docs.swanchain.io/swan-testnet/atom-accelerator-race/before-you-get-started/claim-sepoliaeth
  领水代币：
  https://docs.swanchain.io/swan-testnet/atom-accelerator-race/before-you-get-started/claim-testswan
    ```
    
### 部署
- 环境变量
    - 提前配置环境变量
    ```
    cat >> /etc/profile <<EOF
    export ECP_IP=<映射到公网的IP>
    export ECP_PORT=<映射到公网的端口>
    export PARENT_PATH="<本地存储位置，如：/root/V28>，注意文件会比较大，需要保证200G"
    export FIL_PROOFS_PARAMETER_CACHE=$PARENT_PATH
    export RUST_GPU_TOOLS_CUSTOM_GPU="<GPU model>:<GPU cores>"
    EOF
    
    source /etc/profile
    
    # RUST_GPU_TOOLS_CUSTOM_GPU，可以去此网页查询本机显卡对应的 deviceName：Cores
    # https://github.com/filecoin-project/bellperson?tab=readme-ov-file#supported--tested-cards
    ```

- 初始化脚本
    - 一键安装依赖软件，如docker、显卡驱动等
    ```
    curl -fsSL https://raw.githubusercontent.com/swanchain/go-computing-provider/releases/ubi/setup.sh | bash
 ```

- 下载ZK-FIL  v28 parameters
    - 下载512MiB parameters
    ```
    curl -fsSL https://raw.githubusercontent.com/swanchain/go-computing-provider/releases/ubi/fetch-param-512.sh | bash
    ```
    - 32GiB parameters
    ```
    curl -fsSL https://raw.githubusercontent.com/swanchain/go-computing-provider/releases/ubi/fetch-param-32.sh | bash
    ```
    - PS：下载慢的处理方法
    ``` 
    # 2-3两个步骤通过docker方式下载V28-params 下载速度慢的话，可以使用以下方法加速
    # 1、下载执行文件
    wget https://raw.githubusercontent.com/swanchain/go-computing-provider/releases/ubi/fetch-param-512.sh
    wget https://raw.githubusercontent.com/swanchain/go-computing-provider/releases/ubi/fetch-param-32.sh
    
    # 2、修改脚本 fetch-param-512.sh/fetch-param-32.sh
    #'docker run' 后增加参数 -e IPFS_GATEWAY=https://proof-parameters.s3.cn-south-1.jdcloud-oss.com/ipfs/ -e FIL_PROOFS_PARAMETER_CACHE=/var/tmp/filecoin-proof-parameters
    
    # 3、博主香港实例，实测11M/s下载速度（修改前150-1000k/s），根据自己网络环境配置，海外实例直接执行速度也许会更快
    ```   
    
- 执行脚本
 - 下载执行文件
    ``` 
    wget https://github.com/swanchain/go-computing-provider/releases/download/v0.4.7/computing-provider
    ```
    - 初始化
    ``` 
    ./computing-provider init --multi-address=/ip4/$ECP_IP/tcp/$ECP_PORT --node-name=<自定义ECPnode名称>
    ```
 
 - 生成钱包
 ``` 
 ./computing-provider wallet new
    # 输出： 0x9024a875f44172591373b92...31d67AcCEa
    ``` 
    
    - 也可导入钱包（evm地址即可，需要private_key）
    ```
    ./computing-provider wallet import private.key
 ```

 - 初始化ECP账号
 ``` 
 ./computing-provider account create --ownerAddress <你的钱包地址> --ubi-flag=true
 ```
 - 启动ECP服务
 ``` 
 nohup ./computing-provider ubi daemon >> cp.log 2>&1 &
 ```


### 检查ECP服务
- 检查服务
    ``` 
    curl --location --request GET 'http://$ECP_IP:$ECP_PORT/api/v1/computing/cp' --header 'Content-Type: application/json'
    
    # 注意环境变量是在服务器上设置，本机可以这样调用，其他机器需要修改成指定的ip:port
    # 正确是输出格式参考：https://docs.swanchain.io/orchestrator/as-a-computing-provider/ecp-edge-computing-provider/ecp-faq
    
    ```
