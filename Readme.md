### AWS EKS Cluster with GPU instances and miners for Ethereum mining
I'll try out different options and different miners to find a way to make profitable ETH mining.

Benchmarking different instance types and mining clients.

#### Requirements
- awscli v2
- eksctl 0.35
- kubectl v1.19.1
- aws credentials to access AWS account
- User profile set properly for setup EKS cluster

#### Setup EKS cluster
1. We start from `g4dn.xlarge`, change it in 01-cluster.yaml to feet your needs
2. Create cluster and node group ( 2 instances)
```
eksctl create cluster -f 01-cluster.yaml
```
3. Add created cluster to environment
```
aws eks update-kubeconfig --name [EKS_CLUSTER_NAME] --region us-east-1
```
4. Verify nodes are ready to use
```
kubectl get nodes -o wide
```

#### Update k8s-device-plugin 
1. After  GPU nodes join your cluster, you must apply the NVIDIA device plugin for Kubernetes as a DaemonSet on your cluster with the following command.
```
kubectl apply -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v0.7.3/nvidia-device-plugin.yml
```
2. Verify 
```
kubectl get nodes "-o=custom-columns=NAME:.metadata.name,GPU:.status.allocatable.nvidia\.com/gpu"
NAME                             GPU
ip-192-168-2-148.ec2.internal    1
ip-192-168-31-129.ec2.internal   1
```
3. Check if driver installed:
```
kubectl.exe get pods
NAME                     READY   STATUS    RESTARTS   AGE
t-rex-5549bfccdb-rbp5d   1/1     Running   0          6h12m
t-rex-5549bfccdb-wnvch   1/1     Running   0          6h12m
```
`kubectl exec --stdin --tty t-rex-5549bfccdb-rbp5d -- nvidia-smi`
```
# nvidia-smi
Thu Jan  7 20:37:42 2021       
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 450.51.06    Driver Version: 450.51.06    CUDA Version: 11.0     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|                               |                      |               MIG M. |
|===============================+======================+======================|
|   0  Tesla T4            Off  | 00000000:00:1E.0 Off |                    0 |
| N/A   43C    P0    70W /  70W |   4282MiB / 15109MiB |    100%      Default |
|                               |                      |                  N/A |
+-------------------------------+----------------------+----------------------+
                                                                               
+-----------------------------------------------------------------------------+
| Processes:                                                                  |
|  GPU   GI   CI        PID   Type   Process name                  GPU Memory |
|        ID   ID                                                   Usage      |
|=============================================================================|
+-----------------------------------------------------------------------------+
```
#### Deploy miners

Currently, 2 miners was tested, etherminer and t-rex.
1. ethminer
- Worked with the ethermine.org pool. Deploy the mainer
Add to environment following
- This worker name will shown on https://ethermine.org/

`export WORKER_NAME=ethminer-$HOSTNAME`
- This is a "Miner Address" on https://ethermine.org/ e.g ERC20 address You get the earning.

`export ETH_WALLET=0x1Fa418c70C5f14b21D00c242Bf369A875F129d12`
- Deploy
```
cat "ethminer-deployment.yaml" | sed "s/{{WORKER_NAME}}/$WORKER_NAME/g" | sed "s/{{ETH_WALLET}}/$ETH_WALLET/g" | kubectl apply -f -
```
- Check log to see all working as expected
```
kubectl logs ethminer-6888dd59cc-q2z6w
```
- Remove
```
kubectl.exe delete deployment ethminer
```
#### Deploy t-rex miner
Add to environment following
- This is a "Miner Address" on https://ethermine.org/ e.g ERC20 address You get the earning.

`export ETH_WALLET=0x1Fa418c70C5f14b21D00c242Bf369A875F129d12`
- Connect to preferred pull. Chose pool closet to AWS region e.g us-east-1

`export POOL=us1.ethermine.org:4444`
- Deploy
```
cat "t-rex-deployment.yaml" | sed "s/{{POOL}}/$POOL/g" | sed "s/{{ETH_WALLET}}/$ETH_WALLET/g" | kubectl apply -f -

```
- Benchmark
```
kubectl exec --stdin --tty t-rex-5549bfccdb-rbp5d -- ./t-rex --benchmark -a ethash -o stratum+tcp://$POLL -u $ETH_WALLET -p x -w $HOSTNAME --mt 4 --api-bind-telnet 0 --api-bind-http 0 --cpu-priority 5
```

- You can start port forwarding and have nice dashboard:

![T-rex Dashboard](https://i.imgur.com/T7aQk9J.png)

#### To test if Scale matter:
`kubectl.exe scale deployments ethminer  --replicas=12`

## The profit result's so far:
#### Spot pricing
https://aws.amazon.com/ec2/spot/pricing/

These are the g- and p- series instances: g2, g3, and g4 types are generalized GPU instance types with additional optimizations for graphics-intensive applications like gaming, 
p2 and p3 instances are optimized for CUDA and machine learning applications
```
+---------------+-----------------------------+-------+--------+-----------+-------------------+--------------------+---------+-----------------+------+----------+
| Instance Name | GPUs series                 | GPUs  | vCPUs  | RAM (GiB) | GPU Memory (GiB)  | Spot Price         |   Pods  |   HasRate       | Pods | HasRate  |
+-----------+---------------------------------+-------+--------+-----------+-------------------+--------------------+---------+-----------------+------+----------+
| g4dn.xlarge   | NVIDIA T4 Tensor            |   1   |   4    |    16     |                   | $0.157 per Hour    |    1    |   ~25 Mh/s      |  12  | 7.7 Mh/s |
| g2.2xlarge    | NVIDIA GRID K520 (Kepler)   |   1   |   8    |    64     |                   | $0.195 per Hour    |    1    |   ~2.34 Mh/s    |      |          |
| g3s.xlarge    | NVIDIA Tesla M60            |   1   |   4    |    30.5   |        8          | $0.225 per Hour    |    1    |   ~2.34 Mh/s    |      |          |
| p2.xlarge     | NVIDIA K80                  |   1   |   4    |    61     |        12         | $0.270 per Hour    |    1    |   ~2.34 Mh/s    |      |          |
| p3.2xlarge    | NVIDIA Tesla V100           |   1   |   8    |    61     |        16         | $0.918 per Hour    |         |                 |      |          |
| g4ad.4xlarge  | AMD Radeon Pro V520         |   1   |   16   |    64     |       --          | $0.260 per Hour    |         |                 |      |          |
+---------------+-------------------------------------+-------+---------+----------+-------------------+----------------- --+---------+---------+------+----------+
```
### ethminer
pod log show 25.51 Mh
### t-rex miner
pod log show 25.76Mh/s

pretty much ethermine

![ethermine](https://i.imgur.com/C8YPiGx.png)



### Cleanup
```
eksctl create cluster -f 01-cluster.yaml
```
Or only specific node group:
```
eksctl delete nodegroup --cluster=cluster-1 --name=gpu --region=us-east-1

```

### Docker containers to use
https://hub.docker.com/r/ahtonen/docker-ethminer

- Build
```
git clone https://github.com/ahtonen/ethminer.git
cd ethminer
git submodule update --init --recursive
docker build -t myminer .
```

- Optional Upload to AWS ECR
1. Create ECR registry (if not already done)
```
aws ecr-public get-login-password --region us-east-1 | docker login --username AWS --password-stdin public.ecr.aws/m4n6i3j0
docker tag myminer:latest public.ecr.aws/m4n6i3j0/myminer:latest
docker push public.ecr.aws/m4n6i3j0/myminer:latest
```

### To Do
1. Automatic node's scaling
2. Add pod's autoscaling 
3. Find best performed instance ($/Mhz)
4. Manage entrypoint in kubernetes config map

## Buy me a Coffe
ETH: 0x1Fa418c70C5f14b21D00c242Bf369A875F129d12
