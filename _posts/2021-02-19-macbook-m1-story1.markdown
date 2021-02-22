---
layout: post
title:  "MacBook M1 > Tomcat8 on Kubernetes"
date:   2021-02-19 11:11:00
---

![Image](./_images/macbook-m1-story1/title001.png)

# I. 목적
- MacBook M1 에 Kubernetes 로 Tomcat8 애플리케이션 띄우는 테스트 환경 구축

# II. 준비물
## 1. Device
-  MacBook M1 Chip (Apple Silicon) (다른 M1 이 아니어도 문제는 없어요. 도커 이미지만 바꾸면 돼요)
- Internet

## 2. Software & Scripts
### 1) Docker Desktop Preview (아직 정식버전 아님, 2021/02/19 기준)
- 참고 링크 : https://docs.docker.com/docker-for-mac/apple-m1/
- 다운로드 (2021/02/19 기준) : https://desktop.docker.com/mac/stable/arm64/60984/Docker.dmg
- 트러블 슈팅 : 도커가 계속 starting 상태라면.. 전 아래거로 설치 했어요 ;;
https://desktop.docker.com/mac/stable/arm64/60902/Docker.dmg

### 2) yaml 및 스크립트
- Text는 본문내용 참고
- 다운로드 : curl -L https://github.com/gereon-park/gereon-park.github.io/blob/main/_files/tomcat8_on_kube.tar.gz?raw=true -o tomcat8_on_kube.tar.gz

## 3. 배경지식
### 1) 솔루션 엔지니어로 현재 알고 있는 지식
### 2) Kubernetes Object 개념
- Volume : pv / pvc / hostpath
- Deploy & Pod Control
- Kubernetes Object 와 약어 (단축어)

# III. MacBook M1 > Kubernetes + Tomcat8 기동
## 1. 환경 설정 - Docker Desktop (Preview)
### 1) Resource 탭 설정
![Image](./_images/macbook-m1-story1/docker001.png)

- 사용할 환경에 맞게 리소스 사이즈 조정
- MacBook Pro M1 기준 CPU 8 Core, RAM 8 G or 16 G (전 16 G)
- Disk 는 Docker Images 등이 들어갈 공간이라 그렇게 크게 잡지 않아도 됩니다.
- 실제 사용할 디스크는 MacBook Local 의 아래 디스크 공유를 통해 활용
- 좌측 하단 도커가 붉은색 계열이면 안돼요

## 2. 환경 설정 - Kubernetes
### 1) kubectl 설치
참고 URL : https://kubernetes.io/docs/tasks/tools/install-kubectl/

````
> brew install kubectl
````
or

````
> brew install kubernetes-cli
````

설치 스크린샷>
![Image](./_images/macbook-m1-story1/kube000.png)

### 2) Docker Desktop (Preview)  의 Kubernetes 탭 설정
![Image](./_images/macbook-m1-story1/docker002.png)

- Kubernetes 사용하기 선택
- 좌측 하단 Kubernetes 가 붉은색 계열이면 조금 더 기다려주세요

### 3) 환경 확인
![Image](./_images/macbook-m1-story1/docker003.png)

````
# 참고 - 제 계정에 추가된 설정
alias k=kubectl
alias kns="kubectl config set-context --current --namespace"
complete -F __start_kubectl k
````


## 2. Pod 에서 사용 할 디스크 공유 환경 만들기
### 1) 공통으로 사용할 폴더 만들기
````
# 개인상황에 맞게 만들면 돼요
> mkdir -p ~/WhaTap/BASE 
````

### 2) yaml 폴더 만들기
````
# 개인상황에 맞게 만들면 돼요
> mkdir -p ~/WhaTap/BASE/yaml 
````

### 3) pv 만들기
````
❯ cat pv-BASE.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-local
  labels:
    type: local
spec:
  storageClassName: local
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteMany
  hostPath:
    path: "/Users/mkpark/WhaTap/BASE"

> k apply -f pv-BASE.yaml
````
![Image](./_images/macbook-m1-story1/kube001.png)

- Retain 은 디폴트 값으로 유지한다는 걸로 필수
- RWX : ReadWriteMany 로 복수의 컨테이너에 연결해야 하므로 필수

### 4) 작업 디렉토리 (Namespace) 만들기
````
> k create ns tomcat8-test
````
![Image](./_images/macbook-m1-story1/kube002.png)

### 5) 작업 디렉토리에서 pvc 만들기
````
> kns tomcat8-test

> cat pvc-BASE.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-local
spec:
  storageClassName: local
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
      
> k apply -f pvc-BASE.yaml
````
![Image](./_images/macbook-m1-story1/kube003.png)


#### 6) 컨테이너 기반 솔루션 띄우기
##### (1) 기본 yaml 만들기
````
❯ k create deploy tomcat8 --image=arm64v8/openjdk:8 --dry-run=client -oyaml > deploy-tomcat8.yaml

❯ cat deploy-tomcat8.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: tomcat8
  name: tomcat8
spec:
  replicas: 1
  selector:
    matchLabels:
      app: tomcat8
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: tomcat8
    spec:
      containers:
      - image: arm64v8/openjdk:8
        name: openjdk
        resources: {}
status: {}
````
![Image](./_images/macbook-m1-story1/container001.png)

#### (2) 기본 yaml 에 	공유 Volume 붙이기
````
        volumeMounts:
          - mountPath: "/data"
            name: pvc-volume
      volumes:
        - name: pvc-volume
          persistentVolumeClaim:
            claimName: pvc-local
````
![Image](./_images/macbook-m1-story1/container002.png)

#### (3) 기본 yaml에 CPU / Memory Resource 제한 넣기
````
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "384Mi"
            cpu: "500m"
````
![Image](./_images/macbook-m1-story1/container003.png)

#### (4) 컨테이너에서 실행할 솔루션을 공유 볼륨으로 복사
````
# 개인상황에 맞게 솔루션을 넣으시면 돼요
> cd ~/WhaTap/BASE/

> curl -LO https://downloads.apache.org/tomcat/tomcat-8/v8.5.63/bin/apache-tomcat-8.5.63.tar.gz

> tar -zxf apache-tomcat-8.5.63.tar.gz
````
![Image](./_images/macbook-m1-story1/container004.png)
=> apache-tomcat-8.5.63/bin/catalina.sh 의 501/511 라인의 백그라운드 실행(&) 부분을 지워 주세요.

#### (5) 기본 yaml 에 솔루션 실행환경 넣기
````
        command: ["/bin/sh", "-c"]
        args:
          - echo starting;
            cd /data/apache-tomcat-8.5.63/bin;
            ./startup.sh;
            echo done;
````
![Image](./_images/macbook-m1-story1/container005.png)

#### (6) 솔루션을 Pod (Deployments) 로 실행하기
````
> k apply -f deploy-tomcat8.yaml
````
![Image](./_images/macbook-m1-story1/container006.png)

#### (7) Service Port 붙이기
````
# 기본 yaml 만들기
> k expose deployment tomcat8 --name=svc-tomcat8 --type=NodePort --port=8080 --target-port=8080 --dry-run=client -oyaml > svc-tomcat8.yaml
````
![Image](./_images/macbook-m1-story1/container007.png)

````
# 다음 내용 추가
nodePort: 32000
````
접근을 편하게 하기 위해 NodePort 추가 (30000 ~ 32000 사이 지정)
![Image](./_images/macbook-m1-story1/container008.png)

````
# 서비스 생성
> k apply -f svc-tomcat8.yaml
````
![Image](./_images/macbook-m1-story1/container009.png)

#### 7) Tomcat8 기본 페이지 접근
##### (1) Node 주소 확인
````
> k get no -owide
````
![Image](./_images/macbook-m1-story1/container010.png)

##### (2) Tomcat 페이지 접속
````
http://192.168.65.4:32000
````
![Image](./_images/macbook-m1-story1/container011.png)



# IV. YouTube 동영상
[![IMacBook M1 으로 Tomcat8 on Kubernetes](./_images/macbook-m1-story1/youtube001.png)](https://youtu.be/wmbz-4IhZZ0)






















