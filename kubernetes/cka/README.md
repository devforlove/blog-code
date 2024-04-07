
### 목록에서 업데이트할 버전 조회
```shell
apt update
apt-cache madison kubeadm
```

### kubeadm upgrade 호출 
```shell
apt-mark unhold kubeadm && \
apt-get update && apt-get install -y kubeadm=1.29.x-00 && \
apt-mark hold kubeadm
```

### kubeadm 업그레이드 계획을 확인합니다. 
```shell
kubeadm upgrade plan
```

### 업그레이드할 버전을 선택하고, 적절한 명령을 실행한다. 
해당 명령어를 실행하면 마스터의 컴포넌트들을 다운 받게 됩니다. 
```shell
sudo kubeadm upgrade apply v1.29.x
```

### 노드 드레인 
마스터 노드에 있는 파드들을 전부 삭제해서 비워주는 노드 드레인 명령을 수행합니다. 
```shell
kubectl drain <node-to-drain> --ignore-daemonsets
```

### kubelet과 kubectl 업그레이드 
```shell
ssh <node> # 노드로 이동 
apt-mark unhold kubelet kubectl && \
apt-get update && apt-get install -y kubelet=1.29.x-00 kubectl=1.29.x-00 && \
apt-mark hold kubelet kubectl
```

### kubelet을 다시 시작한다. 
데몬 리스타트를 수행합니다. 
```shell
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

### 노드 uncordon
스케쥴 금지되어 있던 상태를 uncordon 명령어를 이요해서 풀어줍니다. 
```shell
kubectl uncordon <node-to-uncordon>
```


## 유저 생성

### 인증서 생성 
```shell
openssl genrsa -out app-manager.key 2048 # 키 생성 
openssl req -new -key app-manager.key -out app-manager.csr -subj "/CN=app-manager" # 인증서 요청시 필요한 csr 파일 생성
cat app-manager.csr | base64 | tr -d "\n" # Base64 인코딩된 csr 요청을 추출
```

```shell
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: app-manager
spec:
  request: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURSBSRVFVRVNULS0tLS0KTUlJQ1ZqQ0NBVDRDQVFBd0VURVBNQTBHQTFVRUF3d0dZVzVuWld4aE1JSUJJakFOQmdrcWhraUc5dzBCQVFFRgpBQU9DQVE4QU1JSUJDZ0tDQVFFQTByczhJTHRHdTYxakx2dHhWTTJSVlRWMDNHWlJTWWw0dWluVWo4RElaWjBOCnR2MUZtRVFSd3VoaUZsOFEzcWl0Qm0wMUFSMkNJVXBGd2ZzSjZ4MXF3ckJzVkhZbGlBNVhwRVpZM3ExcGswSDQKM3Z3aGJlK1o2MVNrVHF5SVBYUUwrTWM5T1Nsbm0xb0R2N0NtSkZNMUlMRVI3QTVGZnZKOEdFRjJ6dHBoaUlFMwpub1dtdHNZb3JuT2wzc2lHQ2ZGZzR4Zmd4eW8ybmlneFNVekl1bXNnVm9PM2ttT0x1RVF6cXpkakJ3TFJXbWlECklmMXBMWnoyalVnald4UkhCM1gyWnVVV1d1T09PZnpXM01LaE8ybHEvZi9DdS8wYk83c0x0MCt3U2ZMSU91TFcKcW90blZtRmxMMytqTy82WDNDKzBERHk5aUtwbXJjVDBnWGZLemE1dHJRSURBUUFCb0FBd0RRWUpLb1pJaHZjTgpBUUVMQlFBRGdnRUJBR05WdmVIOGR4ZzNvK21VeVRkbmFjVmQ1N24zSkExdnZEU1JWREkyQTZ1eXN3ZFp1L1BVCkkwZXpZWFV0RVNnSk1IRmQycVVNMjNuNVJsSXJ3R0xuUXFISUh5VStWWHhsdnZsRnpNOVpEWllSTmU3QlJvYXgKQVlEdUI5STZXT3FYbkFvczFqRmxNUG5NbFpqdU5kSGxpT1BjTU1oNndLaTZzZFhpVStHYTJ2RUVLY01jSVUyRgpvU2djUWdMYTk0aEpacGk3ZnNMdm1OQUxoT045UHdNMGM1dVJVejV4T0dGMUtCbWRSeEgvbUNOS2JKYjFRQm1HCkkwYitEUEdaTktXTU0xMzhIQXdoV0tkNjVoVHdYOWl4V3ZHMkh4TG1WQzg0L1BHT0tWQW9FNkpsYWFHdTlQVmkKdjlOSjVaZlZrcXdCd0hKbzZXdk9xVlA3SVFjZmg3d0drWm89Ci0tLS0tRU5EIENFUlRJRklDQVRFIFJFUVVFU1QtLS0tLQo=
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 86400  # one day
  usages:
  - client auth
```

```shell
kubectl apply -f app-manager.yaml 
kubectl certificate approve app-manager
kubectl get csr app-manager -o jsonpath='{.status.certificate}'| base64 -d > app-manager.crt # 인증성 생성 
```

### 인증서를 사용하는 유저 생성 
```shell
kubectl config set-credentials app-manager --client-key=app-manager.key --client-certificate=app-manager.crt --embed-certs=true
```

### 유저에 컨텍스트 등록 
```shell
kubectl config set-context app-manager --cluster=arn:aws:eks:ap-northeast-2:457516223683:cluster/test-eks-cluster --user=app-manager
```

## static pod

### 설정 파일 위치 
```shell
ssh hk8s-w1
sudo -i 
cat /var/lib/kubelet/config.yaml
```
- 기본 위치는 staticPodPath: /etc/kubernetes/manifests

pod yaml파일을 만들고 ```/etc/kubernetes/manifests``` 위치에 둡니다. 
```shell
kubectl run nginx-static-pod --image=nginx --port=80 --dry-run=client -o yaml > nginx-static-pod.yaml
```

## Pod

## Deployment

### 문제 1 (스케일 아웃 )
```
문제 
a. webserver라는 이름으로 deployment를 생성하시오
- Name: webserver
- 2 replicas
- label: app_env_stage=dev
- container name: webserver
- container image: nginx:1.14
b. 다음, webserver Deployment의 pod 수를 3개로 확장하시오 
```

```shell
kubectl create deployment webserver --image=nginx:1.14 --replicas=2 --dtry-run=client -o yaml > webserver.yaml
```

deployment의 yaml 파일로 만들고 ```kubectl apply``` 명령어로 deployment 생성합니다.

```shell
kubectl scale deployment webserver --replicas=3
```

### 문제 2 (롤링 업데이트 & 롤백)
```
문제 
Deployment를 이용해 nginx 파드를 3개 배포한 다음 컨테이너 이미지 버전을 rolling update 하고 
update record를 기록합니다. 마지막으로 컨테이너 이미지를 previous revision 으로 roll back 합니다. 
- name: eshop-payment
- image: nginx
- image version: 1.16
- label: app=payment, environment=production
- update image version: 1.17 
```
 
```shell
kubectl set image deployment eshop-payment nginx=nginx:1.17 -- record
```

```shell
kubectl rollout history deployment eshop-payment # 히스토리 확인
kubectl rollout undo deployment eshop-payment 
```


## Node


### 문제 1
```
문제
k8s-worker2 노드를 스케쥴링 불가능하게 설정하고, 해당 노드에서 실행 중인 모든 pod를 다른 node로 reschedule 하세요 
```
```shell
kubectl drain k8s-worker2 --ignore-daemonsets --force
```


### 문제 2
```
문제
Ready 상태 (NoSchedule로 Taint된 node는 제외)인 node를 찾아 그 수를 /var/CKA2022/notaint_ready_node에 기록하세요 
```

```shell
kubectl describe node hk8s-m | grep -i -e taint -e noschedule
```

### 문제 3 
```
문제
다음의 조건으로 pod를 생성하세요
Name: eshop-store
Image: nginx
Node Selector: disktype=ssd
```

```shell
kubectl run eshop-store --image=nginx --dtry-run=client -o yaml 
```

```yaml
...
nodeSelector:
    disktype: "ssd" # disktype이 키이고, ssd가 벨류인 label을 가진 노드에 스케쥴링  
...
```

## configmap 

```
다음의 변수를 configMap eshop으로 등록하세요.
    - DBNAME: mysql 
    - USER: admin
등록한 ehsop configMap의 DBNAME을 ehsop-configmap이라는 이름의 nginx 컨테이너에 DB라는 환경변수로 할당하세요
```

```shell
# configmap 생성 
kubectl create configmap eshop --from-literal=DBNAME=mysql --from-literal=USER=admin
```

```yaml 
# pod yaml 파일에 추가 
...
env: 
- name: DB
  valueFrom:
    configMapKeyRef:
      name: eshop
      key: DBNAME
...
```

## secret 

secret은 base64에 인코딩되어서 관리된다.  

### 문제 1
```
Secret Name: super-secret
Password: bob
pod-secrets-via-file 이라는 파드를 생성하고, 이미지는 redis를 사용한다. 마운트를 이용해서 super-secret을 /secrets 디렉토리에 위치시킨다.
pod-secrets-via-env 이라는 파드를 생성하고, 이미지는 redis를 사용한다. Password를 CONFITENTIAL 환경변수로 사용하도록 한.  
```


```shell
kubectl create secret generic super-secret --from-literal=Password=bob 
```

```
kubectl run pod pod-secret-via-file --image=redis --dry-run=client -o yaml > pod pod-secret-via-file.yaml 
```

```yaml
spec:
  containers:
    - name: mypod
      image: redis
      volumeMounts:
        - name: foo
          mountPath: "/secrets"
  volumes:
    - name: foo
      secret:
        secretName: super-secret
```
Password라는 파일이 컨테이너의 /secrets 디렉토리 아래에 생기고, Password 파일에는 bob이 들어간다. 

```yaml
spec:
  containers:
  - name: pod-secret-via-env
    image: redis
    env:
    - name: CONFIDENTIAL
      valueFrom:
        secretKeyRef:
          name: super-secret
          key: Password
```

## Service

### 문제 1
```
devops namespace에서 운영되고 있는 eshop-order deplyment의 service를 만드세요
```

```shell
kubectl expose deployment --n devops eshop-order --type=ClusterIP --port=80 --target-port=80 --name=eshop-order-svc --dry-run=client -o yaml  
```

### 문제 2
```
미리 배포한 'front-end'에 기존의 nginx 컨테이너의 port 80/tcp를 expose하는 Http라는 이름을 추가합니다. 
컨테이너 포트 http를 expose하는 front-end-svc라는 새 service를 만듭니다. 또한 'NodePort'를 통해 개별 Pods를 expose 되도록 Service를 구성합니다. 
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    run: nginx
spec:
  containers:
  - name: http
    image: nginx
    ports:
      - containerPort: 80
        name: http

---
apiVersion: v1
kind: Service
metadata:
  name: front-end-svc
spec:
  type: NodePort
  selector:
    run: nginx
  ports:
  - name: name-of-service-port
    protocol: TCP
    port: 80
    targetPort: http # 네임 기반의 포트 지정 

```

## 문제 3
```
front-end deployment의 nginx 컨테이너를 expose하는 'front-end-nodesvc'라는 새 service를 만듭니다. 
Front-end로 동작중인 Pod에는 Node의 30200 포트로 접속되어야 합니다. 
구성 테스트 curl k8s-worker1:30200 연결 시 nginx 홈페이지가 표시되어야 합니다. 
```

```shell
kubectl expose deployment front-end --name front-end-svc --port=80 --target-port=80 --type NodePort --dry-run=client -o yaml 
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: front-end-nodesvc
spec:
  type: NodePort
  selector:
    run: nginx
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
      nodePort: 30200
```

## Network Policy 

- app: web 레이블을 가진 Pod에 특정 namespace의 Pod들만 접근 허용 
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: web-allow-prod
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          purpose: production
    ports:
      - protocol: TCP
        port: 80
```

### 문제 1

```
default namespace에 다음과 같은 Pod를 생성하세요 
- name: poc
- image: nginx
- port: 80
- label: app=poc

"partition=customera"를 사용하는 namespace에서만 poc의 80포트로 연결할 수 있도록 default namespace에 "allow-web-from-customera"라는 network policy를 설정해보세요. 
보안정책상 다른 Namespace의 접근은 제한합니다. 

```

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-web-from-customera
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: poc
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          partition: customera
    ports:
      - protocol: TCP
        port: 80
```

## Ingress

```
app-ingress.yaml 파일을 생성하고, 다음 조건의 ingress를 구성하세요 
- name: app-ingress
- NODE_PORT:30080으로 접속했을 때 nginx 서비스로 연결 
- NODE_PORT:30080/app으로 접속했을 때 appjs-service 서비스로 연결 

Ingress 구성에 다음의 annotation을 포함시키세요
annotations: 
    kubernetes.io/ingress.class: nginx
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  namespace: ingress-nginx
  name: app-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx
            port:
              number: 80
      - path: /app
        pathType: Prefix
        backend:
          service:
            name: appjs-service
            port:
              number: 80
```

## coreDNS

```
image nginx를 사용하는 resolver pod를 생성하고, resolver-service라는 service를 구성합니다. 
클러스터 내에서 service와 pod 이름을 조회할 수 있는지 테스트합니다. 
- dns 조회에 사용하는 pod 이미지는 busybox이고, service와 pod이름 조회는 nslookup을 사용합니다. 
- service 조회 결과는 /var/CKA2022/nginx.svc에 pod name 조회 결과는 /var/CKA2022/nginx.pod 파일에 기록합니다. 
```

```shell
kubectl run resolver --image=nginx # pod 생성  
kubectl expose pod resolver --port 80 --name=resolver-service # service 생성 
```

```shell
kubectl run test --image=busybox -it --rm -- /bin/sh 
nslookup 10-244-1-46.default.pod.cluster.local #pod 이름 -> pod dns 
nslookup 10.106.20.222 #clusterIP -> 서비스 dns
```

## Storage

### 문제 1
```
다음 조건에 맞춰서 nginx 웹서버 pod가 생성한 로그파일을 받아서 STDOUT 으로 출력하는 busybox 컨테이너를 운영하시오
- pod Name: weblog

- Web Container 
- image: nginx:1.17
- volume mount: /var/log/nginx
- readwrite

- Log Container
- image: busybox
- command: /bin/sh, -c, "tail -n+1 -f /data/access.log"
- Volume mount: /data
- readonly

emptyDir 볼륨을 통한 데이터 공유 

- Log Container
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: weblog
spec:
  containers:
  - image: nginx:1.17
    name: web
    volumeMounts:
    - mountPath: /var/log/nginx
      name: log-volume
  - image: busybox
    name: log
    args: [/bin/sh, -c, 'tail -n+1 -F /data/access.log']
    volumeMounts:
      - mountPath: /data
        name: log-volume
        readOnly: true
  volumes:
  - name: log-volume
    emptyDir: {}
```

### 문제 2

```
/data/cka/fluentd.yaml 파일에 다음 조건에 맞게 볼륨 마운트를 설정하시오 
worker node의 도커 컨테이너 디렉토리를 동일 디렉토리로 pod에 마운트 하시오
Worker node의 /var/log 디렉토리를 fluentd Pod에 동일 디렉토리로 마운트하시오 
```

```yaml
apiVersion: v1
kind: DaemonSet
metadata:
  name: fluentd
spec:
  selector:
    matchLables:
      name: fluentd
  template:
    metadata:
      labels:
        name: fluentd
    spec:
      volumes:
      - name: dockercontainerdir
        hostPath:
          path: /var/lib/docker/containers
      - name: varlogdir
        hostPath:
          path: /var/log
      containers:
        - name: fluentd
          image: fluentd
          volumeMounts:
          - mountPath: /var/lib/docker/containers
            name: dockercontainerdir
          - mountPath: /var/log
            name: varlogdir
```

### 문제 3 
```
pv001라는 이름으로 size 1Gi, access mode ReadWriteMany를 사용하여 persistent volume을 생성합니다. 
volume type은 hostPath이고 위치는 /tmp/app-config 입니다. 
```

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv001
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  hostPath:
    path: /tmp/app-config
```

### 문제 4
```
다음의 조건에 맞는 새로운 PersistentVolumeClaim을 생성하시오 
Name: pv-volume
Class: app-hostpath-sc
Capacity: 10Mi

앞서 생성한 pv-volume PersistentVolumeClaim을 mount하는 Pod를 생성하시오
Name: web-server-pod
Image: nginx
Mount path: /usr/share/nginx/html
volume에서 ReadWriteMany 액세스 권한을 가지도록 구성합니다. 
```

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata: 
  name: pv-volume
spec:
  storageClassName: app-hostpath-sc 
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Mi
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-server-pod
spec:
  containers:
    - name: myfrontend
      image: nginx
      volumeMounts:
      - mountPath: "/usr/share/nginx/html"
        name: mypd
  volumes:
    - name: mypd
      persistentVolumeClaim:
        claimName: pv-volume
```

## trouble shooting

### 문제 1

```
Pod custom-app의 로그 모니터링 후 'file not found' 오류가 이쓴ㄴ 로그 라인 추출해서 
/var/CK2022/CUSTOM-LOG001 파일에 저장하시오
```

```yaml
kubectl logs custom-app | grep 'file not found' > /var/CK2022/CUSTOM-LOG001
```

### 문제 2
```
클러스터에 구성된 모든 PV를 capacity 별로 sort하여 /var/CKA2022/my-pv-list 파일에 저장하시요
PV 출력 결과를 sort하기 위해 kubectl 명령만 사용하고, 그외 리눅스 명령은 적용하지 마시오
```

```
kubectl get pv --sort.by='{.spec.capacity.storage}' > /var/CKA2022/my-pv-list
```

### 문제 3

```
Worker Node 동작 문제 해결 
hk8s-w2라는 이름의 worker node가 현재 NotReady 상태에 있습니다. 이 상태의 원인을 조사하고 hk8s-w2 노드를 Ready 상태로 전환하여 영구적으로 유지되도록 운영하시오
```

```shell
ssh hk8s-w2
sudo -i 
docker ps 
systemctl status docker # 도커 엔진 시작중인지 확인  
systemctl status kubelet # kubelet 데몬이 시작중인지 확인 
systemctl enable --now kubelet # 지금 당장 시작하고 다음 번에도 영구적으로 시작한다. 
```

```shell
ssh hk8s-w2
sudo -i 
docker ps 
systemctl status docker # 도커 엔진 시작중인지 확인  
systemctl status kubelet # kubelet 데몬이 시작중인지 확인 
systemctl enable --now docker # 지금 당장 시작하고 다음 번에도 영구적으로 시작한다. 
```
