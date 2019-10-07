이전에 수동으로 스카우터 메트릭 로그 데이터를 엘라스틱 서치에 넣을때에는 filebeat와 logstash를 사용한 
파이프라인 수동 구성하고 키바나로 대시보드 만들어 구성보았습니다. 하지만 수동으로 매번 같은 작업을  한다면 불편하다고 생각했습니다.

리서치를 해본결과 filebeat를 사용해서 이러한 문제를 해결할 수 있을것으로 기대했습니다.  

처음 전체 런타임 파이프 라인 구조를 살펴 보도록 하죠  
# 런타임 파이프 라인 구조 
  Log -> filebeat -> elasticsearch -> Kibana

1. filebeat에서 로그 데이터를 읽어 엘라스틱 서치에 전달 합니다. 
2. 엘라스틱서치에는 데이터 가공 후 실제 indexing 작업을 수행 합니다. 공식 명칭은 ingest 라고 합니다. 
 ingest에 궁금하신분들은 아래 링크에서 확인 하세요 <br/>https://www.elastic.co/guide/en/elasticsearch/reference/current/ingest.html  
3. 최종 kibana에서는 엘라스틱 서치에서 처리한 indexing 결과를 확인 할수 있습니다.     

위에 순서 처럼 엘라스틱 서치가 일을 할수 있도록 filbeat 모듈만 개발하기만 하면 됩니다. 사실 개발이라기 보다는 설정에 더 가깝습니다.
제가 그럼 어떻게 설정 했는지 잘 보시기면 하면됩니다. 어려워 보이겠지만 아주 쉬워요 
      
# 전체 개발 과정 요약 

1. 개발환경구성
1. Beats 빌드 시도   
1. filebeat module 생성 
1. filebeat module fileset 생성
1. filebeat module indexing pattern 등록 확인   
1. filebeat ingest pipeline 작성
1. filebeat ingest pipeline 테스트
1. filebeat module 신규 dashboard import 하기           
1. filebeat 만 빌드 하기 
1. filebeat 모듈 테스트

개발 환경은 Linux 또는 Mac을 추천드립니다. 전체 개발 과정은 우분투(18.04) 환경을 기준으로 설명합니다. 
 ## 개발환경구성
Beats 개발에 사용되는 [Go](https://golang.org/) 1.12.10 버전을 설치하세요.

- Go를 설치한 후 $GOROOT,$GOPATH 환경 변수를 설정 하세요 
- GOROOT는 Go를 설치한 경로로 가리키도록 합니다. 
- GOPATH는 개발위치를 가리키도록 합니다.
- $GOPATH/bin과 $GOROOT/bin이 $PATH 환경 변수에 있는지 확인하십시오.

위 절차대로  수행하고 git clone 리포지토리에 사용 된 URL과 일치하는 GOPATH 아래에 디렉토리 구조를 만든 다음 새 디렉토리에서 비트 저장소를 clone 하세요

```bash
# GO 설치 및 환경 변수 설정 
 $ wget https://dl.google.com/go/go1.12.10.linux-amd64.tar.gz;sudo tar xvfz go1.12.10.linux-amd64.tar.gz -C /usr/local
 $ export GOROOT=/usr/local/go
 $ export GOPATH=/home/{user}/elastic_beats
 $ export PATH=$PATH:$GOROOT/bin:$GOPATH:bin
# GO 프로젝트 설정  
 $ mkdir -p ${GOPATH}/src/github.com/elastic
 $ git clone https://github.com/elastic/beats ${GOPATH}/src/github.com/elastic/beats    
``` 
만약 Go 프로젝트 환경변수를 제대로 설정을 않한다면 빌드시 오류가 발생합니다.
'''녹화하면'''
## filebeats 빌드 
그런 다음 Makefile을 사용하여 특정 Beat를 컴파일 할 수 있습니다. 파일 비트의 경우 :
```bash
$ cd ${GOPATH}/src/github.com/elastic/beats/filebeat
$ make
```  
'''녹화하면'''

## filebeat module 생성
### 개요 
각 Filebeat 모듈은 하나 이상의 "fileset"로 구성됩니다. 일반적으로 지원하는 각 서비스 (Nginx의 경우 nginx, Mysql의 경우 mysql 등)에 대한 모듈과 서비스가 생성하는 각 유형의 로그에 대한 파일 세트를 만듭니다. 예를 들어 Nginx 모듈에는 액세스 및 오류 파일 세트가 있습니다. 하나 이상의 일 세트가있는 새 모듈 또는 기존 모듈의 새 파일 세트를 제공 할 수 있습니다.

filebeat 폴더 안에서 이제 파일 모듈을 생성을 하겠습니다.     
```bash
$ cd ${GOPATH}/src/github.com/elastic/beats/filebeat
# make create-module MODULE={module}
$ make create-module MODULE=scouter
```
make create-module 명령을 실행하면 생성 된 파일과 함께 module/scouter에서 모듈을 찾을 수 있습니다. 이 디렉토리에는 다음 파일이 포함되어 있습니다  

```
module/scouter
├── module.yml
└── _meta
    └── docs.asciidoc
    └── fields.yml
    └── config.yml    
    └── kibana                  
``` 
자 하나씩 파일을 보도록 하겠습니다. 

***module.yml***

이 파일에는 모듈에 사용 가능한 모든 대시 보드 목록이 포함되어 있으며  각 대시 보드는 대시 보드가 로컬로 저장된 ID 및 json 파일 이름으로 정의됩니다. 새 fileset 생성시 이 파일은 새 fileset 에 대한 "기본"대시 보드 설정으로 자동 업데이트됩니다.

```yaml
dashboards:
- id: 23d0ec10-e02f-11e9-b41a-bd190673ea83
  file: Filebeat-scouter-host.json
- id: bdb3c200-e00a-11e9-b41a-bd190673ea83
  file: Filebeat-scouter-xlog-overview.json
- id: d4430860-e027-11e9-b41a-bd190673ea83
  file: Filebeat-scouter-javaee.json
- id: f521e660-e017-11e9-b41a-bd190673ea83
  file: Filebeat-scouter-xlog-analysis.json
```

***_meta/docs.asciidoc***

이 파일에는 모듈 별 설명서가 포함되어 있습니다.테스트 된 서비스 버전 및 각 파일 세트에 정의 된 변수에 대한 정보를 포함해야합니다.
참고로 빌드시 fields에서 정의한 description이 자동으로 이 문서에 업데이트가 됩니다. 필요시 작성을 해주세요   
 
***_meta/fields.yml***

모듈 수준 fields.yml에는 모듈 수준 필드에 대한 설명이 포함되어 있습니다. 이 파일의 제목과 설명을 검토하고 업데이트하십시오. title 문서에서 제목으로 사용되므로 대문자를 사용하는 것이 가장 좋습니다.

```yaml
- key: scouter
  title: "Scouter"
  description: >
    scouter Module
  fields:
    - name: scouter
      type: group
      description: >
      fields:
```

***_meta/config.yml***

해당 모듈에 대한 default 설정값이 포함되어 있습니다. 빌드시 해당 파일을 참조하여 modules.d/scouter.yml.disabled 에 적용됩니다.

```yaml
- module: scouter
  # All logs
  log :
    enabled: true
    # Set custom paths for the log files. If left empty,
    # Filebeat will choose the paths depending on your OS.
    # var.paths:
    #  - /path/scouter/server/ext_plugin_filelog/scouter-counter-host.json
    #  - /path/scouter/server/ext_plugin_filelog/scouter-counter-javaee.json
    #  - /path/scouter/server/ext_plugin_filelog/scouter-counter-reqproc.json
    #  - /path/scouter/server/ext_plugin_filelog/scouter-xlog.json
``` 

## filebeat module fileset 생성
filebeat 폴더 안에서 이제 fileset을 생성을 하겠습니다.

```bash
$ cd ${GOPATH}/src/github.com/elastic/beats/filebeat
# make create-fileset MODULE={module} FILESET={fileset}
$ make create-fileset MODULE=scouter FILESET=log
```

make create-fileset 명령을 실행하면 생성 된 파일과 함께 모듈 scouter/log에서 fileset 를 찾을 수 있습니다. 이 디렉토리에는 다음 파일이 포함되어 있습니다. 해당 파일셋은 최종적으로 어떻 파일일때 어떻게 전처리 할것인지 정의 합니다.  

여기서는 스카우터 로그 데이터를 전처리하는 과정이 들어 갔니다.  

```bash
module/scouter/log
├── manifest.yml
├── config
│   └── log.yml
├── ingest
│   └── pipeline.json
├── _meta
│   └── fields.yml
│   └── kibana
│       └── default
└── test
```
***manifest.yml***

manifest.yml은 변수를 정의하고 다른 파일을 참조하는 모듈의 제어 파일입니다.

파일의 var 섹션은 파일 세트 변수 및 해당 기본값을 정의합니다. 모듈 변수는 다른 구성 파일에서 참조 할 수 있으며 Filebeat 구성으로 런타임시 해당 값을 대체 할 수 있습니다.각 변수에는 기본값이 있어야합니다. 가장 간단한 형태로, 사용자가 미지정 할경우 기본값을 설정 위해 사용 합니다.
 
```yaml
module_version: 1.0
var:
  - name: paths
    default:
      - /path/scouter/server/ext_plugin_filelog/scouter-counter-host.json
      - /path/scouter/server/ext_plugin_filelog/scouter-counter-javaee.json
      - /path/scouter/server/ext_plugin_filelog/scouter-xlog.json

ingest_pipeline: ingest/pipeline.json
input: config/log.yml
```
***config/log.yml***
config/  폴더에는 Filebeat입력 구성을 생성하는 템플릿 파일이 있습니다. Filebeat입력은 주로 테일링 파일, 필터링 및 여러 줄을 담당하므로 템플릿 파일에서 구성합니다

```yaml
type: log
paths:
{{ range $i, $path := .paths }}
 - {{$path}}
{{ end }}
exclude_files: [".gz$"]

json.keys_under_root: false
json.add_error_key: true
```

make create-fileset을 실행할 때 자동으로 생성되는 템플릿 파일에서이 찾을 수 있습니다. 이 paths 변수는 입력 경로 옵션에 대한 경로 목록을 구성하는 데 사용됩니다. 스카우터 로그를 읽기 위한 default 타입으로 log를 정의합니다. 
기본적으로 디코딩 된 JSON은 출력 문서에서 "json"키 아래에 배치되게 json.keys_under_root 속성값을 false 로 설정하고 에러 정보를 확인 하기
위한  json.add_error_key 값을 활성화 합니다.


그외에 상세한 타입 및 로그 설정은 아래 링크 주소를 참고해주세요
 - [filebeat inputs](https://www.elastic.co/guide/en/beats/filebeat/master/configuration-filebeat-options.html)
 - [filebeat log configuration](https://www.elastic.co/guide/en/beats/filebeat/master/filebeat-input-log.html)


***ingest/pipeline.json***

ingest/pipeline.json 폴더에는 Elasticsearch [Ingest Node](https://www.elastic.co/guide/en/elasticsearch/reference/master/ingest.html) 파이프 라인 구성이 포함되어 있습니다. ingest pipe line 은 로그 라인 파싱을 하고 데이터를 변환 하는 작업을 수행합니다. 설정은 yaml 또는 json으로 pipeline [정의](https://www.elastic.co/guide/en/elasticsearch/reference/master/pipeline.html)를 할수 있습니다.

여기서는 json pipeline을 구성합니다. processor 값에 ingest가 처리할 방법을 정의하면 됩니다.   
```json
{
  "description": "Pipeline for parsing scouter metric_log logs",
  "processors": [
    
   ],
  "on_failure" : [{
    "set" : {
      "field" : "error.message",
      "value" : "{{ _ingest.on_failure_message }}"
    }
  }]
}
```
processor로 정의 하기전 직접 한번 넣어보고 processor의 내용을 개선을 진행합니다.
  
## filebeat 실행
 - 스카우터 메트릭 샘플 로그 데이터를 준비하여 filebeat 신규 모듈에서 실행을 해보고 실제 데이터가 어떻게 들어가는지 한다.   
  
  
## filebeat ingest pipeline 시뮬레이션 테스트
작성한 파이프 라인이 정상 동작하는지 [Simulate Pipeline API](https://www.elastic.co/guide/en/elasticsearch/reference/master/simulate-pipeline-api.html)를 사용 합니다.
아래의 내용을 키비나 Dev Tools에 들어가서 실행 하여 시뮬레이션 결과를 확인 합니다.
 
```json
[실행구문]                                                                | [결과]
---------------------------------------------------------------------------------------------------------------------------------------------------
POST /_ingest/pipeline/_simulate                                         |  {
{                                                                        |    "docs" : [
  "pipeline": {                                                          |      {
    "description": "Pipeline for parsing scouter metric_log logs",       |        "doc" : {
    "processors": [                                                      |          "_index" : "filebeat-",
      {                                                                  |          "_type" : "_doc",
        "rename": {                                                      |          "_id" : "id",
          "field": "json",                                               |          "_source" : {
          "target_field": "scouter.log"                                  |            "@timestamp" : "2019-09-25T00:00:06.151+09:00",
        }                                                                |            "scouter" : {
      },                                                                 |              "log" : {
      {                                                                  |                "objHash" : "zt77a8",
        "date": {                                                        |                "objId" : "sc-api-demo-s01.localdomain",
          "field": "scouter.log.startTime",                              |                "host" : {
          "formats": [                                                   |                  "TcpStatFIN" : 1,
            "yyyyMMdd'T'HHmmss.SSSZ"                                     |                  "TcpStatCLS" : 1,
          ],                                                             |                  "MemU" : 1074,
          "timezone": "Asia/Seoul",                                      |                  "NetOutBound" : 70,
          "target_field": "@timestamp"                                   |                  "TcpStatEST" : 84,
        }                                                                |                  "NetInBound" : 165,
      }                                                                  |                  "MemA" : 762,
    ]                                                                    |                  "SwapT" : 0,
  },                                                                     |                  "Cpu" : 4.6330476,
  "docs": [                                                              |                  "SwapU" : 0,
    {                                                                    |                  "PageOut" : 0,
      "_index": "filebeat-",                                             |                  "UserCpu" : 2.8254328,
      "_id": "id",                                                       |                  "DiskReadBytes" : 0,
      "_source": {                                                       |                  "NetRxBytes" : 22375,
        "json": {                                                        |                  "TcpStatTIM" : 149,
          "startTime": "20190925T000006.151+0900",                       |                  "Mem" : 58.4846,
          "objName": "/sc-api-demo-s01.localdomain",                     |                  "Swap" : 0,
          "server_id": "SCCOUTER-DEMO-COLLECTOR",                        |                  "DiskWriteBytes" : 231628,
          "objHash": "zt77a8",                                           |                  "startTime" : "20190925T000006.151+0900",
          "objType": "HOST-ScouterDemo",                                 |                  "SysCpu" : 1.1066095,
          "objHost": "sc-api-demo-s01.localdomain",                      |                  "MemT" : 1837,
          "objId": "sc-api-demo-s01.localdomain",                        |                  "PageIn" : 0,
          "objFamily": "host",                                           |                  "NetTxBytes" : 41157
          "host": {                                                      |                },
            "startTime": "20190925T000006.151+0900",                     |                "objHost" : "sc-api-demo-s01.localdomain",
            "MemA": 762,                                                 |                "startTime" : "20190925T000006.151+0900",
            "TcpStatEST": 84,                                            |                "objFamily" : "host",
            "DiskReadBytes": 0,                                          |                "objName" : "/sc-api-demo-s01.localdomain",
            "DiskWriteBytes": 231628,                                    |                "server_id" : "SCCOUTER-DEMO-COLLECTOR",
            "TcpStatTIM": 149,                                           |                "objType" : "HOST-ScouterDemo"
            "UserCpu": 2.8254328,                                        |              }
            "NetRxBytes": 22375,                                         |            }
            "PageIn": 0,                                                 |          },
            "NetOutBound": 70,                                           |          "_ingest" : {
            "TcpStatFIN": 1,                                             |            "timestamp" : "2019-10-07T08:38:00.341Z"
            "SysCpu": 1.1066095,                                         |          }
            "NetTxBytes": 41157,                                         |        }
            "TcpStatCLS": 1,                                             |      }
            "MemU": 1074,                                                |    ]
            "Mem": 58.4846,                                              |  }
            "MemT": 1837,                                                |  
            "Cpu": 4.6330476,                                            |  
            "PageOut": 0,                                                |  
            "Swap": 0,                                                   |  
            "SwapU": 0,                                                  |  
            "SwapT": 0,                                                  |  
            "NetInBound": 165                                            |  
          }                                                              |  
        }                                                                |  
      }                                                                  |  
    }                                                                    |  
  ]                                                                      |  
}                                                                        | 
--------------------------------------------------------------------------------------------------------------------------------------------------------------

```
# filebeat ingest pipeline 작성 
시뮬레이터로 확인이 끝났으면 감이 잡혔을겁니다. 스카우터 메트릭 로그 형태는 json 필드로 구성 되었고 큰변화 없이 pipeline을 적용 하려고 했습니다.
하지만 파일비트 [Naming Convetntions](https://www.elastic.co/guide/en/beats/devguide/current/event-conventions.html)이 현재 필드와 맞지 않아 전체를 소문자로 
바꾸는 작업을 추가적으로 작성 합니다. 여기서 주의할점은 시간 필드 입니다. "date"로  선택 필드를 시계열 필드를 변환 후에는 기존 시계열 필드를 삭제 합니다.  

**만약 삭제 하지 않을시에는 filebeat에서 scouter.log.startTime가 존재시 date format error 문법 에러가 발생 하여 indexing 못한다고 에러를 뱃어냈습니다.** 

만약 기존 시간 필드를 유지 하고 있을경우 필드를 set를 이용하여 다른이름으로 저장 하세요
 
```json
{
  "description": "Pipeline for parsing scouter metric_log logs",
  "processors": [
      {
          "rename": {
              "field": "@timestamp",
              "target_field": "event.created"
          }
      },  
      {
          "rename": {
              "field": "json",
              "target_field": "scouter.log"
          }
      },
      {
          "date" : {
              "field" : "scouter.log.startTime",
              "formats": ["yyyyMMdd'T'HHmmss.SSSZ", "ISO8601"],
              "timezone": "Asia/Seoul",
              "target_field": "@timestamp"
          }
      },
      {
          "remove": {
              "field" : "scouter.log.startTime"
          }
      },   
      {
          "rename": {
              "field": "scouter.log.objName",
              "target_field": "scouter.log.object.name"
          }
      }    
   ],
  "on_failure" : [{
    "set" : {
      "field" : "error.message",
      "value" : "{{ _ingest.on_failure_message }}"
    }
  }]
}

``` 

**_meta/fields.yml**
fields.yml 파일에는 filebeat의 fileset 필드에 대한 최상위 구조가 포함되어 있습니다. 해당 필드 정의는 아래와 같은 용도로 사용됩니다.

- Elasticsearch indexing mapping template 생성
- Kibana 인덱스 패턴 생성
- index document exported 필드 생성 

pipeline에 처리결과가 indexing 생성 작업시 indexing mapping template에 맵핑 되게끝 설정 합니다. 만약 맵핑이 안되는 경우 엘라스틱서치에서는
다이나믹 맵핑을 시도 합니다. 

```yaml
- name: log
  type: group
  description: >
    scouter counter lines.
  fields:
  - name: server
    type: group
    fields:
     - name : id
       type : keyword
  - name: object
    type: group
    fields:
      - name: name
        type: keyword
        description: >
         object name 
      - name: hash
        type: keyword
        description: >
      - name: type
        type: keyword
        description: >
      - name: host
        type: keyword
        description: >
      - name: id
        type: keyword
        description: >
      - name: family
        type: keyword
        description: >
...
        
```
필드 정의가 잘 되었는지 중간 확인 합니다. 아래 명령을 입력해주세요 
```
$make update
```         
그려면 filebeat 내에 filed.yml 이 scouter 모듈이 필드가 업데이트가 되어 있을것을 확인 할수 있습니다.

```yaml
....
- key: scouter
  title: "scouter"
  description: >
    scouter Module
  fields:
    - name: scouter
      type: group
      description: >
      fields:
        - name: log
          type: group
          description: >
            scouter counter lines.
          fields:
          - name: server
            type: group
            fields:
             - name : id
               type : keyword
          - name: object
            type: group
            fields:
              - name: name
                type: keyword
              - name: hash
                type: keyword
              - name: type
                type: keyword
              - name: host
                type: keyword
              - name: id
                type: keyword
              - name: family
                type: keyword
....                
``` 
 
## filebeat 만 빌드 하기
이제 여기까지 왔다면 데이터 넣을 준비까지 끝났습니다. 그려면 여기서 배포 파일을 만들어볼까요?
아래 명령어로 filebeat만 배포 파일을 실행합니다. 
```
$ sudo PLATFORMS='!defaults +linux/amd64 +darwin/amd64' make snapshot
```

## filebeat module 신규 dashboard 만들기
 
```
$ cd ${GOPATH}/src/github.com/elastic/beats/filebeat
$ MODULE=scouter ID=AV4REOpp5NkDleZmzKkE mage exportDashboard
```
```
module/scouter/log
├── manifest.yml
├── config
│   └── {fileset}.yml
├── ingest
│   └── pipeline.json
├── _meta
│   └── fields.yml
│   └── kibana
│       └── 7
│         └──dashboards-AV4REOpp5NkDleZmzKkE.json
|             
└── test
```

```bash
$ cd ${GOPATH}/src/github.com/elastic/beats
$ go run dev-tools/cmd/dashboards/export_dashboards.go -yml filebeat/module/scouter/module.yml   
```
## dashboard index pattern 등록  
```
```
## filebeat dashboard 빌드 
```
$ cd ${GOPATH}/src/github.com/elastic/beats
$ make beats-dashboards
```
## build 
```
sudo PLATFORMS='!defaults +linux/amd64 +darwin/amd64' make snapshot
```
# 결론 sudo 
스카우터 로그 모듈은 제가 불편했던 기존 filbeat,logstash의 설정 과정을 제거 했고, 대시보드까지 내재되었습니다. 
이로서 목표로 삼았던 수동에 대한 불편한 사항을 해결 했습니다. 
                    
         
# 개발 참고 
1. [개발 환경 구성 방법](https://www.elastic.co/guide/en/beats/devguide/current/beats-contributing.html)
2. [Beats 빌드 방법](https://discuss.elastic.co/t/how-to-compile-filebeat-source-code/174158/5)
3. [filebeat 신규 모듈 개발 가이드](https://www.elastic.co/guide/en/beats/devguide/current/filebeat-modules-devguide.html)
4. [신규 대시보드 개발 가이드](https://github.com/eskrug/beats/blob/master/docs/devguide/newdashboards.asciido)
5. [filebeat 소스 발드 방법-Beats Forum](https://discuss.elastic.co/t/how-to-compile-filebeat-source-code/174158/7)