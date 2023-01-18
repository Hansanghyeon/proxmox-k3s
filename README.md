## k3s란

K3S란 가벼운 Kubernetes로 쉽게 설치하고 적은 메모리/binary 파일을 사용하여 Edget/IoT 환경 혹은 CI/Dev 환경에서 k8s를 쉽게 사용할 수 있도록 도와주는 도구이다.

Rancher Labs에서 만든 Kubernetes의 또 다른 버전. Kubernetes와 비교해서는 크게 두가지의 차이점이있다.

경량화, 외부 클라우드 서비스와의 연동 기능을 최소한으로 줄이고, 고가용성(HA) 배포를 위해 기본으로 사용하던 etcd 의존성을 없애고 sqlite를 기본값으로 사용한다. 또한 Docker와 같은 의존성을 모두 삭제하고 containerd와 같은 가벼운 대체재를 사용한다. 기존 Kubernetes에서 지원하는 과거 버전의 API또한 지원하지 않는다.

설치가 무척 간단하다. 경량화를 통해 시스템 요구 사항을 극단적으로 줄인 결실이 설치 과정에서 드러나는데, 쉘 스크립트 하나로 대부분의 배포판에서 설치할 수 있다. 설치 후 자동으로 systemd 서비스 또한 만들기 때문에, 사용자가 신경 써 주어야 할 것이 거의 없다.

## proxmox에서 k3s

k3s의 노드가될 LXC 컨테이너 생성

- control.k8s
- worker-1.k8s
- worker-2.k8s

컨테이너에서 사용할 SSH Public Key를 생성한다.

### Control Plane LXC컨테이너 생성

<kbd>Create CT</kbd> 클릭해서 CT 생성

다음과 같이 설정

<img width="723" alt="스크린샷 2023-01-18 오후 5 06 47" src="https://user-images.githubusercontent.com/42893446/213116959-52d0afa7-fa0a-42e2-86dd-59331db4011f.png">
<img width="720" alt="스크린샷 2023-01-18 오후 5 07 04" src="https://user-images.githubusercontent.com/42893446/213117009-8e8f3a94-0887-405b-8666-7544b69d0985.png">
<img width="728" alt="스크린샷 2023-01-18 오후 5 07 29" src="https://user-images.githubusercontent.com/42893446/213117080-d1a2dd3b-6b7c-4987-8cba-8300071dbc93.png">
<img width="732" alt="스크린샷 2023-01-18 오후 5 07 42" src="https://user-images.githubusercontent.com/42893446/213117122-997cffa8-a297-497f-adf2-81982e65e0bc.png">
<img width="729" alt="스크린샷 2023-01-18 오후 5 07 51" src="https://user-images.githubusercontent.com/42893446/213117166-38177263-ee66-43ff-a2ed-52b309e43b3f.png">
<img width="728" alt="스크린샷 2023-01-18 오후 5 08 14" src="https://user-images.githubusercontent.com/42893446/213117234-f86c15d6-4f25-423b-af00-b8ae5e1ea0ac.png">
<img width="724" alt="스크린샷 2023-01-18 오후 5 08 29" src="https://user-images.githubusercontent.com/42893446/213117288-e7a54ee9-c053-4e7c-b4be-bc833148dd6b.png">

### Worker 노트 LXC 컨테이너 생성

Control 노드와 동일하게 생성하고 Hostnmae을 `worker-[id].k8s`로 변경해서 생성한다.

### 컨테이너 권한부여

이 단계에서는 LXC 컨테이너를 중지해야합니다.

1. pve의 shell or SSH로 연결한다.
2. `vim /etc/pve/lxc/300.conf`
3. 다음을 추가합니다
  ```
  lxc.apparmor.profile: unconfined
  lxc.cgroup.devices.allow: a
  lxc.cap.drop:
  lxc.mount.auto: "proc:rw sys:rw"
  ```
4. 설정 파일을 저장
5. 그런다음 woker 노드에 대해서 동일한 작업을 반복합니다.
  `vim /etc/pve/lxc/[CT_ID].conf`
  
해당 설정을 LXC 컨테이너를 시작하여 설정을 반영합니다.

1. pve의 shell or SSH로 연결한다.
2. `pct push 300 /boot/config-$(uname -r) /boot/config-$(uname -r)`
3. 그런다음 woker 노드에 대해서 동일한 작업을 반복합니다.

### 각 LXC 컨테이너에서

#### 운영체제에서 필요한 프로그램 설치

```
apt-get install neovim git curl
```

#### create `conf-kmsg.sh`

1. `vim /usr/local/bin/conf-kmsg.sh`
2. 내용 추가
  ```
  #!/bin/sh -e
  if [ ! -e /dev/kmsg ]; then
    ln -s /dev/console /dev/kmsg
  fi
  mount --make-rshared /
  ```
  
#### create `conf-kmsg.service`

1. `vim /etc/systemd/system/conf-kmsg.service`
2. 내용 추가
  ```
  [Unit]
  Description=Make sure /dev/kmsg exists

  [Service]
  Type=simple
  RemainAfterExit=yes
  ExecStart=/usr/local/bin/conf-kmsg.sh
  TimeoutStartSec=0

  [Install]
  WantedBy=default.target
  ```
  
#### service 실행

1. `chmod +x /usr/local/bin/conf-kmsg.sh`
2. `systemctl daemon-reload`
3. `systemctl enable --now conf-kmsg`

### Control Plane에 k3s 설치

```sh
curl -fsL https://get.k3s.io | sh -s - --write-kubeconfig-mode 644 --disable traefik --node-name control.k8s
```

1. control plane의 ip 주소 가져오기<br/>
  `hostname -I`
2. k3s woker node 등록 토큰을 표시<br/>
  `cat /var/lib/rancher/k3s/server/node-token`
3. 클러스터에 액세스하기 위해서 `k3s.yaml`을 복사한다.<br/>
  `cp /etc/rancher/k3s/k3s.yaml ~/.kube/config`
  
### woker 노드에 k3s 설치

```
curl -fsL https://get.k3s.io | K3S_URL=https://[control-node-ip]:6443 K3S_TOKEN=[node-token] sh -s - --node-name worker-1.k8s
```

### 설치완료

control 노드에서

```
kubectl get nodes
```

확인가능

## helm을 이용한 nginx ingress controller 추가

helm `ingress-nginx/ingress-nginx` 차트를 사용하여 nginx ingress controller를 설정. 이를 위해서 repo 추가 repo metadata를 로드한 다음 차트를 설치합니다.

helm의 stable repo가 업데이트를 중단했고, k8s는 빠르게 업데이트 되는 중이다. `stable/nginx-ingress`는 사용하기엔 너무 옛날 버전이라서, k8s에서 따로 배포하는 ingress-nginx repo를 사용해 ingress-controller를 설정하자.

### helm repo 추가

```
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
```

```
helm repo update
```

### CLI로 예제

```
helm install nginx-ingress ingress-nginx/ingress-nginx --set controller.publishService.enabled=true
```

여기에서 `controller.publishService.enabled` 설정은 수신할 서비스 IP주소를 수신 리소스에 게시하도록 컨트롤러에 지시합니다.

차트가 완료되면 다양한 리소스가 `kubectl get all` 출력에 표시되어야 합니다. (컨트롤러가 온라인 상태가 되어 로드 밸러서에 IP주소를 할당하는 데 몇 분정도 걸릴 수 있습니다)

control plane의 ip로 접속하면 404 not found nginx를 볼 수 있습니다.

### k8s 파일 예제

#### nginx pod와 service 생성

```
# mynginx.yaml

apiVersion: v1
kind: Pod
metadata:
  labels:
    run: mynginx
  name: mynginx
spec:
  containers:
  - image: nginx:1.16
    name: mynginx
    resources: {}
  restartPolicy: Always
---
apiVersion: v1
kind: Service
metadata:
  name: nginxsvc
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    run: mynginx
```

Service의 Type은 ClusterIP로 설정한다.

```
kubectl apply -f mynginx.yaml
```

#### ingress namespace 생성

```
kubectl create ns ingress-nginx
```

#### helm repo update & search

```
helm repo update
```

```
helm search repo ingress-nginx
```

#### helm install ingress-nginx

```
helm install ingress-nginx ingress-nginx/ingress-nginx -n ingress-nginx
```

```
kubectl get pod -n ingress-nginx
kubectl get svc -n ingress-nginx
```

ingress 구동 

#### ingress-controller 생성 

```yaml
# mynginx-ingress.yaml

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations: 
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/rewrite-target: /
  name: mynginx-ingress
spec:
  rules:
  # - host: -> domain이 없는 경우 생략 가능
  - http:
      paths:
      - path: /nginx
        pathType: Prefix
        backend:
          service:
            name: nginxsvc
            port:
              number: 80
```

```
kubectl apply -f mynginx-ingress.yaml
```

```
kubectl get ing
```

#### 연결 확인

http://ip/nginx

접속하여 nginx가 재대로 실행되는지 확인
