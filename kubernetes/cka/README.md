
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

