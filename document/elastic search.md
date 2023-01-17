elastic search 

강의 순서 
1. 새로운 elasticsearch 데이터 구조 이해하기 
2. Elastic Stack 에서 한국어 NLP 사용하기
3. Elasticsearch 검색에서 확률 사용하기

get select
put update
post insert
delete delte


1. datastream - 새로운 elasticsearch 데이터 구조 이해하기 

agenda
- elasticsearch의 시스템 및 데이터 구조(node, index, shard)
- alias, rollover, api
- datastream
- integrations, elastic agent, fleet
- demo

es 시스템 구조
- 클러스터 : 독립된 es 시스템 환경. 1개 이상의 노드로 구성
- 노드 : 실행중인 es 시스템 프로세스
- 도큐먼트 : 저장된 단일 데이터 단위
- 인덱스 : 도큐먼트의 논리적 집합. 1개 이상의 샤드로 구성
- 샤드 : 색인 & 검색을 진행하는 작업 단위

실행된 프로세스가 노드 데이터 넣으면 인덱스로 들어가고 인덱스는 샤드로 나뉘어서
각각 샤드로 나뉘어서 도큐먼트로 분리되어 저장
엘라스틱서치 시스템은 클러스터로 운용 

실행하면 실행된 프로세스가 노드
데이터 집어넣으면 인덱스로 들어가서 샤드로 나뉘어서 샤드의 도큐먼트들이 분리되어 저장
클러스터라는 단위로 운용됨 

- 샤드는 기본적으로 프라이머리 샤드와 복제본 구성
- 각 샤드들은 클러스터 내의 노드들에 분산되어 저장
- 같은 프라이머리 샤드와 복제본은 같은 데이터셋을 담고 있으며 반드시 다른 노드에 저장
- 데이터 노드가 1개인 경우 복제본은 생성되지 않음 
- 복제본을 통해 무결성을 유지 
- 노드가 유실되면 남아있는 샤드의 데이터를 다른 노드로 복사 

- 각 인덱스별로 프라이머리 샤드와 복제본 세트 수 설정
- 복제본 수는 변경 가능하지만 프라이머리 샤드 수는 기본적으로 처음 인덱스 생성 시점에서 설정 이후 변경 불가능

PUT books/_settings
{
    "index.number_of_replicas":2
}

- 데이터 색인 시 인덱스의 모든 샤드에 round-robin 방식으로 입력
- 사용자는 어떤 도큐먼트가 어떤 샤드에 적재되는지 알 수 없음 (알 필요없음)

- 검색을 할 때도 요청을 받은 노드가 해당되는 모든 샤드에 검색 명령을 전달
- 검색은 각 샤드별로 분산 실행되고 결과는 다시 요청받은 노드로 전달, 취합되어 클라이언트로 응답
- 쿼리를 요청받아 전달하고 다시 취합, 응답하는 노드를 코디네이터 노드라고 함

기본 샤드 개수 - settings : {index.number_of_shards}
- 인덱스 용량에 따라 적절한 수의 샤드로 구성
- 인덱스를 선언하지 않아도 도큐먼트를 입력하면 자동으로 인덱스 생성 -schemless
- 인덱스 당 설정 가능한 최대 샤드 개수 : 1024개
- 6.x 버전 까지는 디폴트로 프라이머리 샤드 5개
- 7.0 부터는 디폴트 프라이머리 샤드 1개 

멀티테넌시 - multitenancy
, * 사용가능 (서로 다른 인덱스 묶어서 한꺼번에 검색가능)
로그 데이터 같은경우 날짜 단위로 데이터를 쌓는 경우가 용이(오버헤드 최소)

요즘은 인덱스로 나눈 멀티테넌시 활용 많아짐 
권장 사이즈는 10~50gb 

alias api 
- 하나의 alias 에 복수개의 인덱스 연결 가능 
- 여러개의 인덱스가 연결된 경우 조회만 가능 
- alias 와 인덱스가 1:1인 경우 색인 가능 
- 입력은 색인 alias, 검색은 조회 alias 로 하면 클라이언트 설정 변경 필요없음
팁은 용도를 다르게 alias를 2개 설정

rollover api 
(index lifecycle management)로 제어하는 것이 바람직함
날짜기준일때 샤드 용량최적화 되지 못함
인덱스 용량을 기준으로 새로 나누어 생성
인덱스가 아닌 alias 또는 datastream을 대상으로 실행
자동으로 생성되는게 아니므로 POST <인덱스>/_rollover

datastream
색인은 입력 alias, 검색은 조회 alias 고민하기 싫을경우
"Is_Wrtie_index":true 옵션으로 rollover할때마다
새로운 입력 alias 분리가능 
-> 모든 것들을 자동화하기 위해 나온 기능이 데이터스트림 

- 데이터가 추가되는 시계열(로그, 메트릭)데이터에서 사용 
- 일반 인덱스를 사용하듯 데이터스트림을 대상으로 데이터 색인, 검색 가능
- 데이터스트림 뒤에 하나 이상의 숨겨진 인덱스들을 자동으로 구성,확장
- 반드시 기준이 되는 date, date_nanos 타입의 @timestamp 필드 필요
- 읽기 요청은 전체 인덱스, 쓰기 요청은 쓰기(최근) 인덱스로 자동 전달 
- alias 와 마찬가지로 rollover 대상으로 지정 가능 
- 숨겨진 인덱스들은 정해진 네임규칙에 의해 자동 생성 
· ds-<데이터셋>-<yyyy.MM.dd>-<generation>
· 데이터셋 - 임의의 데이터스트림 구분자
· generation - 6자리의 십진수 순번 : 000001
- 데이터 추가, 조회만 가능하고 삭제, 변경은 불가능 - alias 와의 차이점
- update by query, delete by query api 를 사용하거나 숨겨진 인덱스에 직접
실행해서 데이터 삭제 또는 업데이트 가능 

설정 순서 
- index lifecycle policy 설정
- index template 설정
- datastream 생성 및 연결 

elastic fleet 
- agent 한곳에서 관리하는 도구 

PUT my-logs-000000
{
    "aliases":{
        "my-logs":{
            "is_write_index": true
        }
    }
}

인덱스 생성 

POST my-logs/_doc
{
    "message": "hello alias"
}

GET my-logs/_search

POST my-logs/_rollover{
    "conditions":{
        "max_age":"10h",
        "max_docs":5,
        "max_size":"1gb"
    }
}

-- 하나라도 만족할 시 rollover (한번실행해야 적용됨)

my-logs-000001로 들어감 

DELETE my-logs-* 



integration
첫화면에 - integrations로 들어가야함 
혹은 fleet 메뉴에서 add agent

integrations - system - add system - system-1 - aaa-policy

fleet - add agent - 명령어 한줄 씩 실행




2. Elastic Stack 에서 한국어 NLP 사용하기

사용하기 위해
1) 훈련된 모델 선택
2) 해당 모델을 엘라스틱 서치로 
3) 실행
4) 인덱스에 적용 

학습 모델은 





설치 환경 
ubuntu 





# rollover 

# 인덱스 추가 
PUT my-logs-000000
{
  "aliases" : {
    "my-logs": {
      "is_write_index": true
    }
  }
}


POST my-logs/_doc
{
  "message": "test alias"
}

GET my-logs/_search

# alias에 rollover (하나라도 만족)
# 조건만족 후 재실행해야 변경
POST my-logs/_rollover 
{
  "conditions": {
    "max_age": "10h",
    "max_docs": 3,
    "max_size": "1gb"
  }
}

# alias 는 delete 적용 x
DELETE my-logs-000001




# window
wget https://artifacts.elastic.co/downloads/beats/elastic-agent/elastic-agent-8.2.0-windows-x86_64.zip -OutFile elastic-agent-8.2.0-windows-x86_64.zip
Expand-Archive .elastic-agent-8.2.0-windows-x86_64.zip
cd elastic-agent-8.2.0-windows-x86_64
.\elastic-agent.exe install

# linux
wget https://artifacts.elastic.co/downloads/beats/elastic-agent/elastic-agent-8.2.0-windows-x86_64.zip -OutFile elastic-agent-8.2.0-windows-x86_64.zip
Expand-Archive .elastic-agent-8.2.0-windows-x86_64.zip
cd elastic-agent-8.2.0-windows-x86_64
.\elastic-agent.exe install