### 신뢰할 수 있는 컴퓨팅(Confidential Computing) 아키텍처 구축 ###
* 목표: AWS EC2의 최신 보안 기술을 활용하여 규제를 준수하는 비식별 처리 시스템 설계 능력 배양
* 트랙 A: 개발자/엔지니어 워크숍 (실습 중심)
* 목표: AWS Nitro Enclaves 및 보안 그룹 설정 실습을 통한 기술 숙련도 향상

|순서|	내용|	상세 목표 및 실습 내용|
|---|---|---|
|1교시|	EC2 기본 및 Nitro System 이해|	AWS Nitro 시스템, NitroTPM, 하드웨어 격리 원리 이해|
|2교시|	금융권 컴플라이언스와 비식별화|	KISA 가이드라인, 비식별 처리 기법, 공동 책임 모델 이해|
|3교시|	실습: 첫 앙클레이브 띄우기|	EC2 인스턴스 생성, CPU/RAM 할당, Docker 이미지를 EIF로 변환|
|4교시|	실습: 보안 시스템 구현 (Java/Rust)	|앙클레이브 내에서 KMS 키를 이용한 암호화/복호화 및 Attestation 구현|

* 트랙 B: 고객/경영진 워크숍 (비전 및 전략 중심)
* 목표: AWS의 보안 기술을 활용한 비즈니스 혁신 및 규제 리스크 해소 전략 제시

|순서|	내용|	상세 목표 및 토의 내용|
|---|---|---|
|1교시|	금융 클라우드: 위기인가, 기회인가?|	규제 완화 트렌드, 데이터 활용의 중요성, 리스크 최소화 전략|
|2교시|	AWS 보안 전략: 제로 트러스트 아키텍처|	Nitro Enclaves, CloudHSM 등 하드웨어 보안 기술로 신뢰 확보|
|3교시|	성공 사례 및 ROI 분석|	Fidelity 등 실제 적용 사례 분석, 비용 절감(Graviton, Spot) 효과|
|4교시|	Q&A 및 차세대 아키텍처 로드맵|	질의응답, 우리 회사의 비식별 처리 시스템 로드맵 토의|

준비물: 발표 자료(PPT), 성공 사례 백서(AWS Artifact 활용), 네트워킹을 위한 다과

## 워크샵 제목 ##
* 개발자 대상 (Hands-on Focus)
  * 제목: 기밀 컴퓨팅(Confidential Computing) 실무: AWS Nitro Enclaves와 Rust/Java를 활용한 데이터 격리 보안
  * 부제: 관리자도 볼 수 없는 '제로 트러스트' 실행 환경 구축하기
* 비즈니스/고객 대상 (Strategy Focus)
  * 제목: 금융 및 민감 데이터 혁신을 위한 기밀 컴퓨팅(Confidential Computing) 전략
  * 부제: 클라우드 규제 대응과 데이터 활용 극대화를 위한 보안 아키텍처의 미래
* 통합 세션 (General)
  * 제목: [Confidential Computing Day] AWS Nitro 기반의 안전한 데이터 비식별화 아키텍처 설계

## 참고자료 ##

* https://aws.amazon.com/ko/blogs/korea/aws-nitro-enclaves-isolated-ec2-environments-to-process-confidential-data/
* https://aws.amazon.com/ko/ec2/nitro/
