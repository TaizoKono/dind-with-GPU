# Kubernrtes DinD SideCar with Nvidia GPU
KubernetsでDinD(Docker in Docker)でGPUコンテナをデプロイしたいときにサイドカーにするコンテナのイメージ
　  
※参考  
https://github.com/ehfd/nvidia-dind
https://github.com/Henderake/dind-nvidia-docker
　  
### 注意
- デプロイされたDinDコンテナはK8sの制御下にないので、リソース配分や監視は工夫する必要がある
- そもそもセキュリティ的に推奨されないので、Podからコンテナを起動したい場合は(Officially-supported Kubernetes client libraries)[https://kubernetes.io/docs/reference/using-api/client-libraries/#officially-supported-kubernetes-client-libraries]やArgo Workflos等の使用を考えた方がよい

### 使用例
- イメージをビルドして適当なリポジトリにPushする
- 以下のようにDocker runしたいPodのサイドカーコンテナとして追加する
~~~yaml
apiVersion: v1
kind: Pod
metadata: 
  name: dind-pod
spec:
  containers:
  # docker runする本体のコンテナ
  - name: docker-runner
    image: your-image
    env: 
    - name: DOCKER_HOST
      value: "tcp://localhost:2375"
    volumeMounts:
    - name: docker-secret
      mountPath: /root/.docker   # プライベートレポジトリからPullする際の認証情報など
    - name: data
      mountPath: /data

  # dind用のサイドカーコンテナ
  - name: dind-sidecar
    image: your-sidecar-image
    ports:
    - containerPort: 2375
    securityContext:
      privileged: true          # 必須
    env: 
    - name: DOCKER_TLS_CERTDIR
      value: ""
    volumeMounts:
    - name: docker-data             # 逐一イメージPullすると非効率なので、
      mountPath: /var/lib/docker    # Dockerのデータ領域はPV化がおすすめ
    - name: data                    # docker run -vのように本体とボリュームを繋げることも可能
      mountPath: /data

  volumes:
    # 以下略
~~~