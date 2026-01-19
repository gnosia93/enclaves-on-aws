실습: 첫 번째 앙클레이브(Enclave) 구동하기

### 1. 환경 준비 (EC2 인스턴스 생성) ###
아무 인스턴스나 되는 게 아니라 Nitro 기반 인스턴스여야 합니다.
* 사양: c5.xlarge 이상의 인스턴스 추천 (최소 4 vCPU 이상 필요).
* 설정: 인스턴스 시작 시 Enclave 지원 활성화 옵션을 반드시 'Enable'로 체크해야 합니다.
* 도구 설치: aws-nitro-enclaves-cli를 설치하여 앙클레이브 제어 준비를 마칩니다.

### 2. 소스 코드 및 도커 이미지 생성 ###
* 앙클레이브 내부에서 실행할 로직(비식별 처리 로직 등)을 작성합니다.
* 언어: 말씀하신 Rust나 Java로 작성된 간단한 "Hello Enclave" 앱.
* 패키징: 이 앱을 일반적인 Docker 이미지로 빌드합니다. 앙클레이브는 운영체제 통째로 들어가는 게 아니라, 이 도커 이미지를 기반으로 구동됩니다.

### 3. EIF(Enclave Image File) 변환 ###
도커 이미지를 앙클레이브 전용 파일 형식인 EIF로 변환합니다.
* 명령어: nitro-cli build-enclave --docker-uri <이미지태그> --output-file sample.eif
* 중요: 이 과정에서 PCR(Platform Configuration Registers)이라는 해시값이 생성됩니다. 이게 나중에 "이 금고 안의 코드는 변조되지 않았음"을 증명하는 디지털 지문이 됩니다.

### 4. 앙클레이브 실행 및 리소스 할당 ###
부모 EC2의 자원 중 일부를 떼어서 앙클레이브를 가동합니다.
* 명령어: nitro-cli run-enclave --eif-path sample.eif --cpu-count 2 --memory 512
* 결과: 이제 부모 서버와 격리된, 네트워크도 안 되고 관리자도 못 들어가는 독립된 공간이 서버 내부에 생깁니다.

### 5. 통신 테스트 (vsock) ###
* 앙클레이브는 인터넷이 안 되므로, 부모 서버와 가상 소켓(vsock) 통신으로 데이터를 주고받습니다.
* 부모 서버에서 비식별화할 샘플 데이터를 소켓으로 쏴주고, 앙클레이브가 이를 처리해 다시 돌려주는 과정을 확인하며 실습을 마무리합니다.
---

```
{
  "transaction_id": "TX-99821",
  "user_name": "홍길동",
  "resident_number": "900101-1234567",
  "card_number": "1234-5678-9012-3456",
  "amount": 55000,
  "location": "서울특별시 강남구 역삼동"
}
```

### 부모 서버 로직 (Java / Spring Boot) ###
사용자의 요청을 받아 앙클레이브 금고로 데이터를 전달하는 역할을 합니다.
EnclaveService.java
```
import org.springframework.stereotype.Service;
import com.amazon.aws.nitrowalle.VsockClient; // 가상의 vsock 라이브러리

@Service
public class EnclaveService {
    public String processSensitiveData(String jsonData) {
        try {
            // 1. 앙클레이브와 vsock 연결 (CID는 보통 16 이상 할당됨)
            VsockClient client = new VsockClient(16, 5000); 
            
            // 2. 금고(Enclave) 안으로 원본 데이터 전송
            client.send(jsonData);
            
            // 3. 비식별화된 결과만 수신
            String result = client.receive();
            
            client.close();
            return result; // "사용자A", "900101-1******" 가 포함된 데이터
        } catch (Exception e) {
            return "Enclave 통신 실패";
        }
    }
}
```

### Enclave 내부용 Java 코드 (Main.java) ###
이 코드는 앙클레이브 안에서 5000번 포트를 열고 기다리다가, 부모 서버가 데이터를 던져주면 비식별화해서 다시 돌려줍니다.
```
import java.io.*;
import java.net.StandardProtocolFamily;
import java.nio.ByteBuffer;
import java.nio.channels.ServerSocketChannel;
import java.nio.channels.SocketChannel;
// 주의: vsock 라이브러리(예: junixsocket)를 사용하여 하드웨어 소켓에 접근해야 합니다.

public class EnclaveProcessor {
    public static void main(String[] args) throws Exception {
        System.out.println("Enclave Java Processor Starting...");

        // 1. VSOCK 서버 소켓 생성 (포트 5000)
        // 실제 환경에서는 Nitro Enclaves 전용 vsock 라이브러리를 종속성에 추가해야 합니다.
        try (ServerSocketChannel serverChannel = ServerSocketChannel.open(StandardProtocolFamily.valueOf("VSOCK"))) {
            serverChannel.bind(new VsockAddress(VsockAddress.CID_ANY, 5000));

            while (true) {
                try (SocketChannel clientChannel = serverChannel.accept()) {
                    // 2. 데이터 수신
                    ByteBuffer buffer = ByteBuffer.allocate(2048);
                    clientChannel.read(buffer);
                    buffer.flip();
                    
                    String input = new String(buffer.array(), 0, buffer.limit());
                    System.out.println("Received sensitive data in Enclave.");

                    // 3. 비식별 처리 (주민번호 마스킹 예시)
                    String deIdentified = deIdentify(input);

                    // 4. 결과 전송
                    ByteBuffer outputBuffer = ByteBuffer.wrap(deIdentified.getBytes());
                    clientChannel.write(outputBuffer);
                }
            }
        }
    }

    private static String deIdentify(String data) {
        // 간단한 정규식을 이용한 비식별화 로직 (이름, 주민번호 마스킹)
        return data.replaceAll("\"user_name\":\"[^\"]+\"", "\"user_name\":\"사용자A\"")
                   .replaceAll("\"resident_number\":\"(\\d{6})-\\d{7}\"", "\"resident_number\":\"$1-1******\"");
    }
}
```

Dockerfile 
```
# 1단계: 빌드
FROM maven:3.8-openjdk-17-slim AS build
COPY src /app/src
COPY pom.xml /app
RUN mvn -f /app/pom.xml clean package

# 2단계: 실행 (앙클레이브 탑재용)
FROM openjdk:17-jdk-slim
COPY --from=build /app/target/enclave-app.jar /app.jar

# 앙클레이브 통신에 필요한 라이브러리 경로 설정 등
ENTRYPOINT ["java", "-jar", "/app.jar"]
```
* 메모리 최적화: Java는 JVM 때문에 기본적으로 메모리를 많이 먹습니다. 앙클레이브 실행 시 --memory 옵션을 넉넉히(최소 1GB 이상) 주거나, GraalVM을 이용해 네이티브 이미지로 빌드하여 메모리 사용량을 줄이는 것이 금융권 실무 팁입니다.
* 의존성 관리: 앙클레이브 안은 인터넷이 안 되므로 모든 라이브러리가 JAR 파일 안에 포함(Fat JAR)되어 있어야 합니다.
