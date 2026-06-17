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
<img width="2181" height="2593" alt="서비스 핵심 기술" src="https://github.com/user-attachments/assets/12f1a947-7293-432e-88db-9a4fc8c02318" />

### 3-2. 통합 워크플로우 다이어그램
<img width="2531" height="1387" alt="통합 워크플로우 다이어그램" src="https://github.com/user-attachments/assets/8a21edd7-0137-4a4b-9ec0-0d39ab822755" />

### 3-3. 세부 기능 소개

#### [기능 1] Thanos 기반 Prometheus 이중화 및 Grafana 통합 관측 플랫폼 구축

* **기능 설명**

Kubernetes 환경에서 Prometheus 단일 장애점을 제거하기 위해 kube-prometheus-stack의 Prometheus를 2개의 replica로 구성하고, 각 Prometheus Pod에 Thanos Sidecar를 함께 배치하였습니다.

각 Prometheus는 WAS Pod, Kubernetes 리소스, Node 등의 메트릭을 수집하고, Thanos Query는 여러 Prometheus Sidecar의 데이터를 통합 조회하며 replica 라벨을 기준으로 중복 메트릭을 제거합니다. Grafana는 개별 Prometheus가 아닌 Thanos Query를 단일 DataSource로 사용하여, Prometheus 장애 상황에서도 안정적인 통합 모니터링 화면을 제공합니다.

* **핵심 코드(스크립트)**

```bash
  prometheus:
    enabled: true
  
    thanosService:
      enabled: true
      clusterIP: "None"
  
    prometheusSpec:
      replicas: 2
  
      externalLabels:
        cluster: won-card-eks
  
      replicaExternalLabelName: replica
  
      thanos:
        image: quay.io/thanos/thanos:v0.39.0
  
  grafana:
    enabled: true
  
    sidecar:
      datasources:
        defaultDatasourceEnabled: false
  
    additionalDataSources:
      - name: Thanos
        type: prometheus
        url: http://thanos-query.monitoring.svc.cluster.local:9090
        isDefault: true
```

* **코드(스트립트) 링크**
  
https://github.com/FISA-BOCE-backup/WON-Infra-IaC-backup/blob/main/Application-Monitoring/servicemonitor-securities-was.yaml
https://github.com/FISA-BOCE-backup/WON-Infra-IaC-backup/blob/main/Application-Monitoring/thanos-query.yaml
https://github.com/FISA-BOCE-backup/WON-Infra-IaC-backup/blob/main/Application-Monitoring/values-kps-thanos.yaml

<br/>

#### [기능 2] Keepalived와 Route Table 전환 기반 WireGuard VPN Tunnel HA


* **기능 설명**

1. WireGuard EC2 및 VM의 Health Check 과정

WireGuard 이중화 환경에서는 온프레미스 VM과 AWS EC2가 각각 3초 주기로 Health Check를 수행합니다. VM은 Keepalived vrrp_script로 AWS EC2 터널 IP를 확인하고, EC2는 systemd timer 기반 스크립트로 온프레미스 VM과 Peer EC2의 Private IP를 확인합니다. 2회 연속 실패 시 장애로 판단하며, VM 구간은 VIP 이동, EC2 구간은 Route Table ENI 전환으로 장애에 대응합니다.

2. VPN Tunnel 장애 시 EC2 Failover 과정

AWS WireGuard EC2는 온프레미스 VM과의 터널 상태를 ping으로 확인하고, 2회 연속 실패 시 장애로 판단합니다. 장애 발생 시 Route Table의 10.1.200.0/24 대상 ENI를 조회하여 자신이 Active인 경우 replace-route 명령으로 Peer EC2 ENI로 라우팅을 전환합니다. Standby 상태라면 별도 변경 없이 로그만 남깁니다.

3. VPN Tunnel 장애 시 VM Failover 과정

온프레미스 WireGuard VM은 Keepalived 기반으로 VIP 10.1.200.60을 공유하며, 내부 시스템은 해당 VIP를 Next-hop으로 사용합니다. 각 VM은 AWS EC2 터널 대상에 3초마다 ping을 수행하고, 2회 연속 실패 시 장애로 판단합니다. Active VM 장애 시 Standby VM이 VRRP를 통해 VIP를 인계받아 내부 라우팅 변경 없이 트래픽을 정상 VM으로 전환합니다.


* **핵심 코드(스크립트)**
```bash
  #!/usr/bin/env bash
  set -euo pipefail
  source /etc/wg-ha.env
  
  ping_ok() {
    ping -c 1 -W 1 "$1" >/dev/null 2>&1
  }
  
  get_current_route_target() {
    aws ec2 describe-route-tables \
      --region "$REGION" \
      --route-table-ids "$ROUTE_TABLE_ID" \
      --query "RouteTables[0].Routes[?DestinationCidrBlock=='$DEST_CIDR'].NetworkInterfaceId | [0]" \
      --output text
  }
  
  switch_route() {
    aws ec2 replace-route \
      --region "$REGION" \
      --route-table-id "$ROUTE_TABLE_ID" \
      --destination-cidr-block "$DEST_CIDR" \
      --network-interface-id "$PEER_ENI"
  }
  
  if ! ping_ok "$MY_VM_IP"; then
    current_target="$(get_current_route_target)"
  
    if [[ "$current_target" == "$MY_ENI" ]]; then
      echo "Tunnel failure detected. Switch route to peer EC2."
      switch_route
    else
      echo "Standby node detected failure. No route change."
    fi
  fi
```

* **코드(스트립트) 링크**

https://github.com/FISA-BOCE-backup/WON-Infra-IaC-backup/blob/main/03-compute/01-wireguard/main.tf
https://github.com/FISA-BOCE-backup/WON-Infra-IaC-backup/blob/main/03-compute/01-wireguard/ansible/playbook.yml

<br/>

#### [기능 3] MySQL Primary-Replica/Redis Sentinel 기반 DB/캐시 고가용성 구성

* **기능 설명**

1. MySQL Primary-Replica 기반 데이터 복제 구성

계정계 DB의 단일 장애점을 줄이기 위해 MySQL 2대를 Primary-Replica 구조로 구성하고, Primary의 데이터 변경 사항이 Replica로 복제되도록 하였습니다. 복제는 GTID 기반으로 구성하여 장애 복구나 Replica 재합류 시 binlog 위치를 직접 지정하지 않고도 복제 연결을 재설정할 수 있도록 하였습니다.

2. Redis Sentinel 기반 HA 구성
   
카드망과 증권망 각각에 Redis EC2 3대를 배치하고, 1 Master + 2 Replica와 Redis Sentinel 3개로 HA 구조를 구성하였습니다.

Sentinel은 quorum 2 기준으로 Master 장애를 감지하고 Replica를 자동 승격하며, 애플리케이션은 Sentinel을 통해 현재 Master를 조회하여 설정 변경 없이 Redis 연결을 유지합니다.

* **핵심 코드(스크립트)**
1. MySQL failover 구성
```
  #!/bin/bash
  
  SUCCESSOR_HOST="${4:-}"
  SUCCESSOR_PORT="${5:-3306}"
  
  WRITER_HOSTGROUP=10
  MYSQL_CNF="/etc/orchestrator/secrets/proxysql-admin.cnf"
  
  map_host() {
    case "$1" in
      mysql-01) echo "10.1.200.201" ;;
      mysql-02) echo "10.1.200.202" ;;
      *) echo "$1" ;;
    esac
  }
  
  SUCCESSOR_PROXYSQL_HOST="$(map_host "${SUCCESSOR_HOST}")"
  
  mysql --defaults-extra-file="${MYSQL_CNF}" -e "
  DELETE FROM mysql_servers WHERE hostgroup_id=${WRITER_HOSTGROUP};
  
  INSERT INTO mysql_servers(hostgroup_id, hostname, port, max_connections)
  VALUES (${WRITER_HOSTGROUP}, '${SUCCESSOR_PROXYSQL_HOST}', ${SUCCESSOR_PORT}, 1000);
  
  LOAD MYSQL SERVERS TO RUNTIME;
  SAVE MYSQL SERVERS TO DISK;
  "
```

2. Sentinel 기반 장애 감지 및 자동 Failover

```bash
  port 26379
  
  sentinel monitor card-redis-master 10.11.31.101 6379 2
  sentinel auth-pass card-redis-master ${REDIS_PASSWORD}
  sentinel down-after-milliseconds card-redis-master 5000
  sentinel failover-timeout card-redis-master 60000
```

* **코드(스트립트) 링크**

#### [기능 4] Neo4j+ MysQL + Azure AI API 기반 질문 유형별 응답 최적화

* **기능 설명**
* **핵심 코드(스크립트)**
* **코드(스트립트) 링크**

#### [기능 4] SQS와 스케줄러 기반 카드 포인트 ETF 전환 비동기 배치

* **기능 설명**
* **핵심 코드(스크립트)**
* **코드(스트립트) 링크**
