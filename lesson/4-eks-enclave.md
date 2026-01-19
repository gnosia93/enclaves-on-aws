### 1. EKS에서 앙클레이브를 쓰는 방식 ###
* EKS 노드(EC2)가 앙클레이브를 지원하는 인스턴스 타입(예: c5.xlarge 등)이어야 하며, Nitro Enclaves Kubernetes Device Plugin을 설치해야 합니다.
* 역할: 이 플러그인이 쿠버네티스 스케줄러에게 "이 노드에는 앙클레이브를 띄울 수 있는 자원이 있다"라고 알려줍니다.

### 2. Pod 설정 (Manifest) ###
개발자가 yaml 파일에 "나는 앙클레이브 자원이 필요해"라고 명시해야 합니다. 
```
resources:
  limits:
    ://aws.amazon.com: "1" # 앙클레이브 자원 1개 요청
```
