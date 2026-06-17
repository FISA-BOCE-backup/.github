# [우리FISA 6기] 클라우드 엔지니어링 과정 2팀 
> 클엔의 정석 : 서가영(PM), 이채유(PL), 박성준, 김종연


## 1. 프로젝트 개요

### 주제
일상 소비를 자산으로 자동 전환하는 그룹사(카드사, 증권사) 내 락인(Lock-In) 플랫폼

### 프로젝트 기획 배경
소멸되거나 방치되는 카드 포인트의 비효율을 해결하기 위해 기획된 프로젝트입니다. 
카드 결제로 적립된 포인트를 고객이 직접 관리하지 않아도 ETF 투자로 자동 전환·운용함으로써, 흩어진 소액 포인트를 자산 형성의 수단으로 연결합니다. 

이는 카드 포인트가 같은 계열 증권사의 투자 자산으로 이어지는 구조이기에 고객은 계열사 금융 서비스를 지속적으로 이용하게 되어 자연스러운 락인(Lock-in) 시너지 효과가 발생하며, 충성 고객 확보와 자산 이탈 방지로 이어집니다. 

고객에게는 포인트 활용도를 높여주는 동시에 금융사에게는 계열사 간 자산 선순환과 고객 유지라는 비즈니스 가치를 함께 제공하는 것을 목표로 합니다.
또한 금융권 망분리 규제를 준수하는 하이브리드 클라우드 아키텍처 상에서 카드사·증권사 간 안전한 데이터 연계 구조를 구현합니다.

### 기술 스택
- **Infra**: AWS (EC2, EKS, ECR, VPC, WAF, API Gateway, Route 53, SQS, PrivateLink, Site-to-Site VPN), Azure (Container Apps, OpenAI, Key Vault, VPN Gateway), Terraform, Docker, Ansible, WireGuard, Nginx, NLB, vSphere, VM Workstation, Ubuntu
- **Backend**: Java 17, Spring Boot, React Native, Expo, TypeScript
- **Database**: MySQL 8.0, Neo4j (GraphDB), Amazon RDS, Redis 7.2, ProxySQL
- **CI/CD**: GitHub Actions, Jenkins, ArgoCD, LocalXpose
- **Monitoring**: Prometheus, Thanos, Grafana, Grafana Loki, Promtail, CloudWatch
- **Collaboration**: GitHub, Slack, Notion, Discord, Swagger, CodeRabbit, n8n, Figma

<br/>

## 2. 아키텍쳐

### 2-1. 시스템 아키텍쳐
<img width="2784" height="2716" alt="시스템 아키텍처 drawio" src="https://github.com/user-attachments/assets/a95139bf-58f3-4dab-853d-0a8a108f51fc" />

#### 설명

금융 서비스의 보안성, 확장성, 고가용성을 확보하기 위해 온프레미스 데이터센터, AWS 클라우드, Microsoft Azure를 연계한 하이브리드 멀티 클라우드 아키텍처로 구성하였습니다. 핵심 계정계 시스템과 주요 데이터베이스는 온프레미스에 배치하고, 사용자 요청을 처리하는 채널계 서비스는 AWS 기반으로 구성하여 금융권 환경에 적합한 망 분리 구조를 적용하였습니다.

> AWS

사용자 요청은 Route 53을 통해 유입되며, WAF에서 웹 공격 및 비정상 트래픽을 1차적으로 차단합니다.
이후 API Gateway가 각 서비스 도메인별 요청을 분기하고, VPC Link를 통해 외부에 직접 노출되지 않는 AWS Private Subnet 내부로 트래픽을 전달합니다. 
이를 통해 실제 애플리케이션 서버와 내부 서비스는 외부 접근으로부터 보호됩니다.

AWS 영역은 카드망, 증권망, 공통망 VPC로 분리하였습니다. 
카드망과 증권망은 각각 독립적인 서비스 영역으로 구성하여 장애 전파를 최소화하고, 공통망은 인증 및 공통 기능을 담당하는 영역으로 설계하였습니다. 
각 VPC는 DMZ, EKS, Data Layer, Public Subnet으로 세분화되어 역할별 네트워크 경계를 명확히 합니다.

카드망과 증권망의 DMZ 영역에는 Nginx 기반 EC2 인스턴스와 Internal NLB를 배치하여 외부 API Gateway와 내부 WAS 계층 사이의 중계 구간을 구성하였습니다. 
이후 트래픽은 EKS 내부 서비스로 전달되며, EKS는 업무 WAS Pod와 모니터링 Pod를 운영한다. Auto Scaling Group과 Node Group을 통해 서비스 부하 증가 및 노드 장애 상황에서도 안정적인 서비스 운영이 가능하도록 하였습니다.

데이터 계층은 Redis, PostgreSQL, MySQL 등으로 분리 구성하였습니다. Redis는 Sentinel 기반으로 장애 조치가 가능하도록 설계하였으며, 서비스별 DB를 분리하여 장애 영향 범위를 줄였습니다. 온프레미스 계정계 DB와의 연계를 통해 중요 원장 데이터는 내부망에서 관리하고, 클라우드는 채널계 처리 중심으로 운영합니다.

> 온프레미스

온프레미스 데이터센터는 핵심 계정계 시스템과 DB 고가용성 구성을 담당합니다. 
Nginx, 애플리케이션 VM, ProxySQL, Orchestrator, MySQL 이중화 구조를 통해 DB 장애 발생 시 Primary 전환과 접속 경로 변경이 가능하도록 구성하였다. 이를 통해 계정계 DB 장애 상황에서도 서비스 연속성을 확보할 수 있습니다.

> VPN 연결

AWS와 온프레미스 간 통신은 Site-to-Site VPN 및 WireGuard 기반 사설망 연결을 통해 수행됩니다. 이를 통해 클라우드 채널계 서비스와 온프레미스 계정계 시스템이 공용 인터넷에 직접 노출되지 않고 안전하게 연계됩니다.

> Azure

Microsoft Azure 영역은 외부 AI 서비스 연계를 위한 별도 망으로 구성하였습니다. Azure Container Apps, Managed Identity, Key Vault, Private Endpoint, Azure OpenAI 연계 구조를 통해 AI 기능을 금융 서비스 영역과 분리된 환경에서 안전하게 처리할 수 있도록 설계하였습니다.

### 2-2. 소프트웨어 아키텍처
<img width="4746" height="3110" alt="소프트웨어 아키텍처" src="https://github.com/user-attachments/assets/ffe68efc-04ae-4f65-aa4a-095c32de8dc1" />

#### 설명

이 아키텍처는 사용자 접근 계층부터 업무 서비스, AI 처리, 플랫폼 공통 기능, 데이터 계층까지 역할에 따라 분리된 계층형 구조로 구성되어 있습니다.

사용자의 요청은 모바일 앱, 관리자 대시보드, 내부 운영 호출을 통해 유입되며, 보안 및 라우팅 계층을 거쳐 각 도메인 서비스로 전달됩니다. 

업무 서비스 계층에서는 카드, 증권, 공통 업무를 분리하여 각 서비스의 책임을 명확히 했고, AI 처리 계층에서는 챗봇 질의 분석, 데이터 조회 분기, 비동기 이벤트 처리, 배치 작업을 담당하도록 구성했습니다. 

또한 인증/인가, 감사 로그, 배포, 보안 연동과 같은 공통 운영 기능은 플랫폼 계층에서 관리하고, 데이터 계층에서는 서비스별 데이터베이스, 캐시, AI 분석 DB, 그래프 DB를 목적에 따라 분리하여 확장성과 유지보수성을 고려했습니다.

<br/>

## 3. 주요 기능 소개

### 3-1. 핵심 기술 구성

### 3-2. 통합 워크플로우 다이어그램
<img width="2531" height="1387" alt="통합 워크플로우 다이어그램" src="https://github.com/user-attachments/assets/8a21edd7-0137-4a4b-9ec0-0d39ab822755" />

### 3-3. 세부 기능 소개

#### [기능 1] Redis 분산 락을 이용한 재고 감소 로직

* **기능 설명**
* **핵심 코드(스크립트)**
* **코드(스트립트) 링크**
