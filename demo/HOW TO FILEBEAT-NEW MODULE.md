이전에 수동으로 스카우터 메트릭 로그 데이터를 엘라스틱 서치에 넣을때에는 filebeat와 logstash를 사용한 
파이프라인 수동 구성하고 키바나로 대시보드 만들어 구성보았습니다. 하지만 수동으로 매번 같은 작업을  한다면 불편하다고 생각했습니다.

리서치를 해본결과 filebeat를 사용해서 이러한 문제를 해결할 수 있을것으로 기대했습니다.
 
 
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
이제 파일 모듈을 생성을 하겠습니다. 

```bash
$ cd ${GOPATH}/src/github.com/elastic/beats/filebeat
$ make create-module MODULE=scouter
```

```
module/scouter
├── module.yml
└── _meta
    └── docs.asciidoc
    └── fields.yml
    └── kibana                  
``` 

## filebeat module fileset 생성
```bash
$ cd ${GOPATH}/src/github.com/elastic/beats/filebeat
$ make create-fileset MODULE=scouter FILESET=log
```
```bash
module/scouter/log
├── manifest.yml
├── config
│   └── {fileset}.yml
├── ingest
│   └── pipeline.json
├── _meta
│   └── fields.yml
│   └── kibana
│       └── default
└── test
```
  
## filebeat ingest pipeline 작성
```
...
```
## filebeat ingest pipeline 시뮬레이션 테스트
```bash
curl -X POST "localhost:9200/_ingest/pipeline/my-pipeline-id/_simulate?pretty" -H 'Content-Type: application/json' -d'
{
  "docs": [
    {
      "_index": "index",
      "_id": "id",
      "_source": {
        "foo": "bar"
      }
    },
    {
      "_index": "index",
      "_id": "id",
      "_source": {
        "foo": "rab"
      }
    }
  ]
}
'
```
 - https://www.elastic.co/guide/en/elasticsearch/reference/master/simulate-pipeline-api.html

## filebeat module fileset fields 생성

```bash
$ cd ${GOPATH}/src/github.com/elastic/beats/filebeat
$ make create-fields MODULE=scouter FILESET=log
```
 
           
## filebeat 만 빌드 하기
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