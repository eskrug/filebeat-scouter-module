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
[![asciicast](https://asciinema.org/a/272978.svg)](https://asciinema.org/a/272978)

## filebeats 빌드 
그런 다음 Makefile을 사용하여 특정 Beat를 컴파일 할 수 있습니다. filebeat만 빌드를 해보겠습니다. 
```bash
$ cd ${GOPATH}/src/github.com/elastic/beats/filebeat
$ make
```  
[![asciicast](https://asciinema.org/a/272979.svg)](https://asciinema.org/a/272979)


## filebeat module 생성
### 개요 
각 Filebeat 모듈은 하나 이상의 "fileset"로 구성됩니다. 일반적으로 지원하는 각 서비스 (Nginx의 경우 nginx, Mysql의 경우 mysql 등)에 대한 모듈과 서비스가 생성하는 각 유형의 로그에 대한 파일 세트를 만듭니다. 
예를 들어 Nginx 모듈에는 액세스 및 오류 파일 세트가 있습니다. 새 모듈 또는 기존 모듈에 새 파일 세트를 제공 할 수 있습니다.

filebeat 폴더 안에서 이제 새 파일 모듈을 생성을 하겠습니다.여기서는 신규 module 이름을 scouter로 하겠습니다.         
```bash
$ cd ${GOPATH}/src/github.com/elastic/beats/filebeat
# make create-module MODULE={module}
$ make create-module MODULE=scouter
```
make create-module 명령을 실행하면 생성 된 파일과 함께 module/scouter 에서 모듈을 찾을 수 있습니다. 이 디렉토리에는 다음 파일이 포함되어 있습니다  

```
module/scouter
├── module.yml
└── _meta
    └── docs.asciidoc
    └── fields.yml
    └── config.yml    
    └── kibana                  
```

 
[![asciicast](https://asciinema.org/a/272981.svg)](https://asciinema.org/a/272981)

하나씩 파일을 살펴 보도록 하겠습니다. 
***module.yml***

이 파일에는 모듈에 사용 가능한 모든 대시 보드 목록이 포함되어 있으며  각 대시 보드는 대시 보드가 로컬로 저장된 ID 및 json 파일 이름으로 정의됩니다.
새 fileset 생성시 이 파일은 새 fileset 에 대한 **기본** 대시 보드 설정로 자동 업데이트됩니다. 처음 생성시에는 내용이 비어 있습니다. 
나중에 dashbaord를 import 할때 사용합니다.  
```yaml
dashboards:
``` 

***_meta/docs.asciidoc***

이 파일에는 모듈 별 설명서가 포함되어 있습니다. 빌드시 fileset에 field 에서 정의한 description이 자동으로 이 문서에 업데이트가 됩니다.     

 
***_meta/fields.yml***

fields.yml에는 이 모듈에 대한 간략한 설명이 포함되어 있습니다. 이 파일에 필요시 title 과 description 을 업데이트 합니다.   
```yaml
- key: scouter
  title: "scouter"
  description: >
    scouter Module
  fields:
    - name: scouter
      type: group
      description: >
      fields:
```

***_meta/config.yml***

해당 모듈에 대한 default 설정을 합니다. 빌드 과정에서 해당 파일을 참고하여 모듈 설정 파일로 생성합니다.  

기존 :  
```yaml
- module: scouter
  # All logs
  {fileset}:
    enabled: true
    # Set custom paths for the log files. If left empty,
    # Filebeat will choose the paths depending on your OS.
    #var.paths:                  
``` 
변경 :  fileset 플레이스 홀더 부분의 값을 log로 변경 하고 로그 경로 샘플 값을 넣습니다.   
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

스카우터 로그를 읽어 동작시키기 위한 fileset 생성을 하겠습니다.


```bash
$ cd ${GOPATH}/src/github.com/elastic/beats/filebeat
# make create-fileset MODULE={module} FILESET={fileset}
$ make create-fileset MODULE=scouter FILESET=log
```


make create-fileset 명령을 실행하면  생성 된 파일과 함께 scouter/log에서 fileset을 찾을 수 있습니다. 
이 디렉토리에는 다음 파일이 포함되어 있습니다.     

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
[![asciicast](https://asciinema.org/a/272982.svg)](https://asciinema.org/a/272982)

하나씩 파일을 살펴 보도록 하겠습니다. 

***manifest.yml***

manifest.yml은 변수를 정의하고 다른 파일을 참조하는 모듈의 제어 파일입니다.
파일의 var 섹션은 파일 세트 변수 및 해당 기본값을 정의합니다.
 
모듈 변수는 다른 구성 파일에서 참조 할 수 있으며 Filebeat 구성으로 런타임시 해당 값을 대체 할 수 있습니다.각 변수에는 기본값이 있어야합니다. 가장 간단한 형태로, 사용자가 미지정 할경우 기본값을 설정 위해 사용 합니다.

기존 :
```yaml 
module_version: 1.0

var:
  - name: paths
    default:
      - /example/test.log*
    os.darwin:
      - /usr/local/example/test.log*
    os.windows:
      - c:/programdata/example/logs/test.log*

ingest_pipeline: ingest/pipeline.json
input: config/log.yml
```
변경 :  스카우터 로그가 있는 곳으로 기본 설정값으로 변경합니다. 하지만 로그 위치가 유저 기준으로 설치 되기 때문에 큰 의미는 없습니다.   
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

filebeat입력 구성을 생성하는 템플릿 파일 입니다. filebeat입력은 주로 테일링 파일, 필터링 및 여러 줄을 담당하는 설정을 합니다. 

기존 : 
```yaml
type: log
paths:
{{ range $i, $path := .paths }}
 - {{$path}}
{{ end }}
exclude_files: [".gz$"]
```
변경 : 
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
paths 변수는 입력 경로 옵션에 대한 경로 목록을 구성하는 데 사용하고 default로 로그타입으로 정의 되어 있습니다. 변경이 없이 이부분은 나둡니다. 
스카우터 메트릭 로그 형태가 라인별 JSON으로 구성 되어 있기 때문에 추가적으로 JSON과 연관된 설정을 추가 합니다.

기본적으로 디코딩 된 JSON은 출력 문서에서 "json"키 아래에 배치되게 json.keys_under_root 속성값을 비활성화 하고 에러 정보를 확인 하기
위한  json.add_error_key 값을 활성화 합니다. 


그외에 상세한 타입 및 로그 설정은 아래 링크 주소를 참고해주세요
 - [filebeat inputs](https://www.elastic.co/guide/en/beats/filebeat/master/configuration-filebeat-options.html)
 - [filebeat log configuration](https://www.elastic.co/guide/en/beats/filebeat/master/filebeat-input-log.html)


***ingest/pipeline.json***

ingest/pipeline.json 폴더에는 Elasticsearch [Ingest Node](https://www.elastic.co/guide/en/elasticsearch/reference/master/ingest.html) 파이프 라인 구성이 포함되어 있습니다. ingest pipeline 은 로그 라인 파싱을 하고 데이터를 변환 하는 작업을 수행합니다. 설정은 yaml 또는 json으로 pipeline [정의](https://www.elastic.co/guide/en/elasticsearch/reference/master/pipeline.html)를 할수 있습니다.

여기서는 json으로 설정합니다. 이제 processor 값에 ingest가 처리할 방법을 정의하면 됩니다.   
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
processor로 정의 하기전 filebeat를 빌드하고 실행하여 processor의 내용보면서 개선을 진행합니다. 
  
## filebeat 실행 
일단 하면 어떻게 처리 되는지  스카우터 메트릭 샘플 로그 데이터를 준비하여 filebeat 신규 모듈에서 실행을 해보고 실제 데이터가 어떻게 들어가는지 확인하고
index mapping template 도 확인 해야 겠죠? 샘플 데이터는 따로 준비한 디렉토리에 존재 합니다. 샘플 로그를 가지고 실행을 해보죠.
  
```
$ make update
$ ./filebeat -e 
```

[![asciicast](https://asciinema.org/a/272989.svg)](https://asciinema.org/a/272989)

![filebeat실행](../assert/filebeat실행.gif)

filebeat가 읽은 시간 대로 기준으로 엘라스틱서치에 시계열로 누적 되고 있음알수 있습니다. 그냥 로그를 읽었다 수준입니다. 
이제 로그을  전처리 하고 indexing mapping 처리 되도록 pipeline 설정을 만들어 보도록 하겠습니다.  
      
## filebeat ingest pipeline 시뮬레이션 테스트

[Simulate Pipeline API](https://www.elastic.co/guide/en/elasticsearch/reference/master/simulate-pipeline-api.html)를 사용 하여
pipeline를 구성하여 테스트 해보도록 합니다. 


방금전 테스트 한 데이터를 하나면 가져와 샘플링 합니다.  


```
{
  "_index": "filebeat-8.0.0-2019.10.08-000001",
  "_type": "_doc",
  "_id": "DGIiqW0BqpddJseO2lCv",
  "_version": 1,
  "_score": null,
  "_source": {
    "@timestamp": "2019-10-08T02:11:34.745Z",
    "json": {
      "startTime": "20190925T035131.790+0900",
      "objHash": "zt77a8",
      "server_id": "SCCOUTER-DEMO-COLLECTOR",
      "objType": "HOST-ScouterDemo",
      "objId": "sc-api-demo-s01.localdomain",
      "objFamily": "host",
      "host": {
        "MemT": 1837,
        "TcpStatCLS": 1,
        "SysCpu": 0.60557353,
        "TcpStatTIM": 151,
        "UserCpu": 2.4217916,
        "NetOutBound": 67,
        "TcpStatFIN": 0,
        "MemU": 1074,
        "Swap": 0,
        "startTime": "20190925T035131.790+0900",
        "DiskReadBytes": 0,
        "Cpu": 3.2278678,
        "SwapU": 0,
        "TcpStatEST": 79,
        "NetTxBytes": 34902,
        "SwapT": 0,
        "PageOut": 0,
        "NetInBound": 164,
        "Mem": 58.48545,
        "NetRxBytes": 20492,
        "MemA": 762,
        "DiskWriteBytes": 198962,
        "PageIn": 0
      },
      "objName": "/sc-api-demo-s01.localdomain",
      "objHost": "sc-api-demo-s01.localdomain"
    },
    "service": {
      "type": "scouter"
    },
    "event": {
      "module": "scouter",
      "dataset": "scouter.log"
    },
    "host": {
      "name": "orange",
      "containerized": false,
      "hostname": "orange",
      "architecture": "x86_64",
      "os": {
        "kernel": "4.15.0-62-generic",
        "codename": "bionic",
        "platform": "ubuntu",
        "version": "18.04.3 LTS (Bionic Beaver)",
        "family": "debian",
        "name": "Ubuntu"
      },
      "id": "c00b4e802f284e49a05abeeb34ddd32f"
    },
    "log": {
      "offset": 13519441,
      "file": {
        "path": "/home/kranian/sample/scouter-counter-host_2019-09-25.json"
      }
    },
    "input": {
      "type": "log"
    },
    "agent": {
      "type": "filebeat",
      "ephemeral_id": "c02f62a8-a824-4aef-ac83-5058ae3de0eb",
      "hostname": "orange",
      "id": "f881d229-f982-4b35-a9f8-9f9dd80bf958",
      "version": "8.0.0"
    },
    "ecs": {
      "version": "1.1.0"
    },
    "fileset": {
      "name": "log"
    }
  },
  "fields": {
    "@timestamp": [
      "2019-10-08T02:11:34.745Z"
    ]
  },
  "sort": [
    1570500694745
  ]
}
```
그려면 원문 데이터 중 _source 항목 중 scouter 연관된 필드만 따로 추출 하여 simulate api에 넣고 실행 합니다. 
그리고 시뮬레이터 API 중 processors 정의 합니다. 
 
```
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
이런식으로 반복적으로 테스트 하고 원하는 결과가 나올때 까지 pipeline processors 테스트 합니다. 
테스트 했던 결과를 토대로 pipeline processors를 설정 합니다.   
 

# filebeat ingest pipeline 작성 

시뮬레이터로 확인이 끝났으면 감이 잡혔을겁니다. pipeline 설정하면 indexing 결과가 이렇게 나오겠구나 하고 예상 하실수 있을겁니다. 
파일비트 [Naming Convetntions](https://www.elastic.co/guide/en/beats/devguide/current/event-conventions.html) 규칙이 존재 하기때문에 
해당부분을 읽고 processors를 작성 해야 합니다.  
     

여기서 주의할점은 시간 필드 입니다. "date"로  선택 필드를 시계열 필드를 변환 후에는 기존 시계열 필드를 삭제 합니다.filebeat 기준 시계열 기준은 
@timestamp 입니다. timestamp값을 스카우터 로그 기록 시간(scouter.log.startTime) 필드로 변환 하고, 기존 필드를 remove 해야 합니다. 
  
만약 삭제 하지 않을시에는 filebeat에서 scouter.log.startTime가 존재시 date format error 문법 에러가 발생 하여 indexing 못한다고 에러 발생시킵니다.  
만약 기존 시간 필드를 유지 하고 있을경우 필드 다른 이름으로 저장 하세요
 
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
저는 이런식으로 processors를 작성했습니다.  fields를 이용하여 엘라스틱서치  index mapping template을 작성해 보죠  
  
**_meta/fields.yml**

fields.yml 파일에는 filebeat의 fileset 필드에 대한 최상위 구조가 포함되어 있습니다. 해당 필드 정의는 아래와 같은 용도로 사용됩니다.

- Elasticsearch indexing mapping template 생성
- Kibana 인덱스 패턴 생성
- index document exported 필드 생성 

pipeline에 처리결과가 indexing 생성 작업시 데이터 타입이 맵핑 되게 설정 합니다. fields 설정은 아래와 같습니다.   

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
$ cd ${GOPATH}/src/github.com/elastic/beats/filebeat
$ PLATFORMS='linux/amd64' make snapshot
$ cd ${GOPATH}/src/github.com/elastic/beats/filebeat/build/distributions
```

[![asciicast](https://asciinema.org/a/272999.svg)](https://asciinema.org/a/272999)


리눅스 외에 다른 플래폼 목록을 알고 싶다면 아래 명령어로 확인 합니다. 
```
$ go tool dist list
```

너무 내용이 길어져 대쉬보드 내재 시키는 방법은 아래 링크에 담도록 하겠습니다. 
[filebeat module 신규 dashboard 만들기](HOW TO FILEBEAT-DASHBOARD_IMPORT.md) 


# 결론 
기존 불편했던 filbeat,logstash의 수동 설정 부분을 스카우터 로그 모듈만으로 동작시켜 제외 시켜 봤습니다.     
이로서 목표로 삼았던 수동에 대한 불편한 사항을 해결 했습니다.
 
                 
         
# 개발 참고 
1. [개발 환경 구성 방법](https://www.elastic.co/guide/en/beats/devguide/current/beats-contributing.html)
2. [Beats 빌드 방법](https://discuss.elastic.co/t/how-to-compile-filebeat-source-code/174158/5)
3. [filebeat 신규 모듈 개발 가이드](https://www.elastic.co/guide/en/beats/devguide/current/filebeat-modules-devguide.html)
4. [신규 대시보드 개발 가이드](https://github.com/eskrug/beats/blob/master/docs/devguide/newdashboards.asciido)
5. [filebeat 소스 발드 방법-Beats Forum](https://discuss.elastic.co/t/how-to-compile-filebeat-source-code/174158/7)