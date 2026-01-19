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

### 1. 1대 1 방식 (사이드카 스타일) ###
가장 보안성이 높고 직관적인 방식입니다.
* 구조: 하나의 비즈니스 파드(Java/Spring)가 자신만을 위한 전용 앙클레이브를 점유합니다.
* 장점: 파드 간 간섭이 전혀 없고 보안 격리가 완벽합니다.
* 단점: EC2 노드의 자원(CPU/Mem) 소모가 큽니다. (앙클레이브마다 자원을 떼어줘야 하므로)

### 2. 1대 N 방식 (공유 서비스 스타일) ###
성능과 비용을 고려한 대규모 트래픽용 방식입니다.
* 구조: 특정 노드에 '비식별 처리 전문 파드'를 띄우고, 이 파드가 앙클레이브를 관리합니다. 다른 여러 개의 일반 파드들이 이 전문 파드에게 "이것 좀 비식별화해줘"라고 요청(gRPC 등)을 보냅니다.
* 장점: 자원을 효율적으로 쓰고 관리가 편합니다.
* 단점: 공유 파드가 죽으면 연결된 모든 파드의 비식별 처리가 중단됩니다.

----
앙클레이브의 물리적 원칙은 "한 놈(부모)하고만 소켓(vsock)으로 대화한다"는 것입니다.
이를 EKS에서 구현할 때의 정확한 매커니즘은 다음과 같습니다.

### 1. 앙클레이브의 통신 원칙 (vsock) ###
앙클레이브는 외부 IP가 없습니다. 오직 자기를 실행한 EC2 호스트(노드)와만 vsock이라는 특수 통로로 연결됩니다.

### 2. 그럼 1:N(여러 파드 공유)은 어떻게 가능한가? ###
앙클레이브가 직접 여러 파드와 대화하는 게 아니라, 중간에 '대리인' 파드를 두는 방식입니다.
구조:
* 대리인 파드 (Proxy): 이 파드가 앙클레이브를 실행한 '진짜 부모'입니다. 앙클레이브와 vsock으로 연결되어 있습니다.
* 일반 파드들 (Clients): 이들은 앙클레이브를 직접 못 봅니다. 대신 대리인 파드에게 일반적인 네트워크(HTTP/gRPC)로 "이것 좀 처리해줘"라고 부탁합니다.
* 전달: 대리인 파드가 그 데이터를 받아 vsock을 통해 앙클레이브로 넘겨주고, 결과를 받아 다시 일반 파드들에게 돌려줍니다.

### 3. 워크숍에서 권장하는 방식: 1대 1 (Sidecar 패턴) ###
복잡한 공유 방식보다는 쿠버네티스 사이드카(Sidecar) 패턴으로 1:1 매칭을 가르치는 것이 훨씬 깔끔합니다.
* 구조: 하나의 Pod 안에 [애플리케이션 컨테이너]와 [앙클레이브 관리 컨테이너]를 같이 넣습니다.
* 장점: 앙클레이브가 해당 파드 전용 금고가 되어 보안 사고 시 추적이 쉽고 설계가 단순합니다.

-----

### 1단계: 파드(Pod) 구조 설계 ###
하나의 파드 안에 두 개의 컨테이너를 정의합니다.
* 메인 컨테이너 (App): 비즈니스 로직(Java/Spring)을 수행하며, 앙클레이브에 데이터를 던지는 역할.
* 사이드카 컨테이너 (Enclave Manager): 앙클레이브(EIF 파일)를 실제로 로드하고 vsock 프록시 역할을 수행.

### 2단계: EKS 배포 매니페스트 (YAML) ###
이게 실습의 핵심 코드입니다. nitro-enclaves 리소스를 요청하는 부분이 포인트입니다.
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: confidential-app
spec:
  template:
    spec:
      containers:
      # 1. 메인 앱 컨테이너
      - name: main-app
        image: my-repo/spring-boot-app:latest
        env:
        - name: ENCLAVE_CID
          value: "16" # 앙클레이브와 통신할 ID

      # 2. 앙클레이브 사이드카 컨테이너
      - name: enclave-sidecar
        image: my-repo/nitro-enclave-manager:latest
        resources:
          limits:
            ://aws.amazon.com: "1" # 앙클레이브 자원 점유
        securityContext:
          privileged: true # 하드웨어 직접 제어를 위해 필요
        volumeMounts:
        - name: enclave-info
          mountPath: /data
      
      volumes:
      - name: enclave-info
        emptyDir: {}

```
### 3단계: 컨테이너 간 통신 (vsock 통로 구축) ###
사이드카 컨테이너가 앙클레이브를 띄우면, 메인 컨테이너는 어떻게 대화할까요?
* 부팅 시: 사이드카 컨테이너가 nitro-cli run-enclave 명령어로 앙클레이브를 실행합니다.
* 연결: 메인 컨테이너(Java)는 Amazon vsock 라이브러리를 사용하여 호스트의 vsock 인터페이스에 접근합니다.
* 데이터 흐름: Main App -> Vsock -> Nitro Hypervisor -> Enclave

### 앙클레이브 이미지 ###
nitro-enclave-manager는 AWS가 공식적으로 "이거 쓰세요"라고 제공하는 단일 이미지는 없으며, AWS Nitro Enclaves CLI가 설치된 환경에서 직접 빌드해야 합니다

* 1. AWS 공식 샘플 활용 (가장 추천)
AWS에서 제공하는 Nitro Enclaves Kubernetes 가이드나 공식 워크숍 레포지토리에 Dockerfile 샘플이 포함되어 있습니다.
이 파일들을 가져와서 docker build를 통해 본인의 리포지토리(ECR 등)에 올린 뒤 워크숍에서 사용해야 합니다.

* 2. 직접 빌드하는 법 (Dockerfile 구성)
사이드카 컨테이너는 앙클레이브를 실행(run-enclave)하고 관리하는 역할을 해야 하므로, 아래와 같은 구성으로 이미지를 만듭니다.
```
# 앙클레이브 매니저 이미지 예시
FROM amazonlinux:2

# 1. Nitro CLI 도구 설치
RUN amazon-linux-extras install aws-nitro-enclaves-cli -y
RUN yum install -y aws-nitro-enclaves-cli-devel

# 2. 앙클레이브 이미지(EIF) 파일 복사 (미리 빌드해둔 파일)
COPY my-enclave-app.eif /data/my-enclave-app.eif

# 3. 앙클레이브 실행 스크립트 실행
# (메모리와 CPU 자원을 할당하며 앙클레이브를 띄움)
ENTRYPOINT ["nitro-cli", "run-enclave", "--eif-path", "/data/my-enclave-app.eif", "--memory", "512", "--cpu-count", "2"]

```

#### 3. AWS 마켓플레이스 및 오픈소스 검색 ####
직접 만들기 번거롭다면, 보안 기업들이 제공하는 이미지를 참고할 수 있습니다.
Anjuna나 Fortanix 같은 기밀 컴퓨팅 전문 기업들이 앙클레이브 관리를 돕는 런타임을 제공하지만, 대부분 유료 솔루션입니다.


참가자가 "복호화 코드가 사이드카에 있나요?"라고 물으면 이렇게 답하세요:
"아니요, 사이드카는 단순히 '금고를 설치하고 문을 여는 관리자'일 뿐입니다. 실제 복호화가 일어나는 핵심 로직은 사이드카가 띄운 '앙클레이브 내부'라는 격리된 방 안에서만 실행됩니다. 그래서 사이드카를 관리하는 서버 관리자조차 복호화되는 원본 데이터를 볼 수 없는 것입니다."
