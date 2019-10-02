# 데모 실행 환경 요구 사항 
 - Linux 4.x 이상    
 - docker 설치    
   - 설치 가이드는 https://docs.docker.com/install 여기에 리눅스 부분를 참고한다.   
 - docker-compose 설치 
   - 설치 가이드는 https://docs.docker.com/compose/install 여기에 리눅스 부분를 참고한다.                 
#  Step 1
 - 스카우터 메트릭 샘플 로그를 다운로드 한다. 
```
 wget https://github.com/eskrug/filebeat-scouter-module/releases/download/7.3.3/2019-09-25-scouter-sample-files.tar.gz
```
 - 여기 프로젝트에 릴리스된 Filebeat를 다운로드 한다.
```
 wget https://github.com/eskrug/filebeat-scouter-module/releases/download/7.3.3/filebeat-oss-7.3.3-SNAPSHOT-linux-x86_64.tar.gz
```
 - 그리고 압축 해제  
```
$ tar xvfz scouter-sample-files.tar.gz
$ tar xvfz filebeat-oss-7.3.3-SNAPSHOT-linux-x86_64.tar.gz 
```    
# Step 2 
 - Elastic + Kibana를 로컬에 docker-compose를 이용하여 서버를 실행 한다.
```       
$cat > docker-compose.yml 
version: '3.2'
services:
 es:
  image: sebp/elk:730
  environment:
   - LOGSTASH_START=0
  ports:
    - "5601:5601"
    - "9200:9200"
$docker-compose up -d     
```
 - 참고로 영구볼륨을 설정 하지 않았기 때문에 만약 재기동시에는 이전에 엘라스틱 서치에 들어간 데이터는 사란진다.
 - elastic + kibana가 정상적으로 기동 여부 확인 후 파일 비트 설정을 한다.
# Step 3   
 - 스카우터 대시보드 Setup 
```   
$ cd ~/filebeat-7.3.3-SNAPSHOT-linux-x86_64 
$ ./filebeat setup -E setup.dashboards.zip=https://github.com/eskrug/filebeat-scouter-module/releases/download/7.3.3/scouter-dashboard.zip
Index setup finished.
Loading dashboards (Kibana must be running and reachable)
Loaded dashboards
Loaded machine learning job configurations
Loaded Ingest pipelines
```
 - 스카우터 모듈 활성화 
```
$./filebeat modules enable scouter
```
 - 스카우터 모듈 설정
```
$ vi modules.d/scouter.yml 
...
# Module: scouter
# Docs: https://www.elastic.co/guide/en/beats/filebeat/7.3/filebeat-module-scouter.html
- module: scouter
  # All logs
  log :
    enabled: true
    # Set custom paths for the log files. If left empty,
    # Filebeat will choose the paths depending on your OS.
    # 주석을 푸고 아까 받았던 샘플 파일 경로 및 패턴을 아래와 같이 설정한다.     
    var.paths:
     - /home/{user}/sample-files/scouter-*.json

```  
 - Filebeat 실행 
```
$./filebeat -e
019-10-02T17:40:42.533+0900	INFO	instance/beat.go:607	Home path: [/home/kranian/sample/filebeat-7.3.3-SNAPSHOT-linux-x86_64] Config path: [/home/kranian/sample/filebeat-7.3.3-SNAPSHOT-linux-x86_64] Data path: [/home/kranian/sample/filebeat-7.3.3-SNAPSHOT-linux-x86_64/data] Logs path: [/home/kranian/sample/filebeat-7.3.3-SNAPSHOT-linux-x86_64/logs]
2019-10-02T17:40:42.545+0900	INFO	instance/beat.go:615	Beat ID: 1022b20d-d6a2-4776-a631-a5877fbf6e17
2019-10-02T17:40:42.550+0900	INFO	[seccomp]	seccomp/seccomp.go:124	Syscall filter successfully installed
2019-10-02T17:40:42.550+0900	INFO	[beat]	instance/beat.go:903	Beat info	{"system_info": {"beat": {"path": {"config": "/home/kranian/sample/filebeat-7.3.3-SNAPSHOT-linux-x86_64", "data": "/home/kranian/sample/filebeat-7.3.3-SNAPSHOT-linux-x86_64/data", "home": "/home/kranian/sample/filebeat-7.3.3-SNAPSHOT-linux-x86_64", "logs": "/home/kranian/sample/filebeat-7.3.3-SNAPSHOT-linux-x86_64/logs"}, "type": "filebeat", "uuid": "1022b20d-d6a2-4776-a631-a5877fbf6e17"}}}
...
2019-10-02T17:40:42.555+0900	INFO	instance/beat.go:292	Setup Beat: filebeat; Version: 7.3.3
2019-10-02T17:40:42.555+0900	INFO	[index-management]	idxmgmt/std.go:178	Set output.elasticsearch.index to 'filebeat-7.3.3' as ILM is enabled.
2019-10-02T17:40:42.555+0900	INFO	elasticsearch/client.go:170	Elasticsearch url: http://localhost:9200
2019-10-02T17:40:42.555+0900	INFO	[publisher]	pipeline/module.go:97	Beat name: orange
2019-10-02T17:40:42.560+0900	INFO	instance/beat.go:422	filebeat start running.
2019-10-02T17:40:42.560+0900	INFO	[monitoring]	log/log.go:118	Starting metrics logging every 30s
2019-10-02T17:40:42.578+0900	INFO	registrar/registrar.go:145	Loading registrar data from /home/user/sample/filebeat-7.3.3-SNAPSHOT-linux-x86_64/data/registry/filebeat/data.json
2019-10-02T17:40:42.578+0900	INFO	registrar/registrar.go:152	States Loaded from registrar: 4
2019-10-02T17:40:42.578+0900	INFO	crawler/crawler.go:72	Loading Inputs: 1
2019-10-02T17:40:42.594+0900	INFO	log/input.go:148	Configured paths: [/home/user/sample/sample-files/scouter-*.json]
2019-10-02T17:40:42.594+0900	INFO	crawler/crawler.go:106	Loading and starting Inputs completed. Enabled inputs: 0
2019-10-02T17:40:42.594+0900	INFO	cfgfile/reload.go:171	Config reloader started
2019-10-02T17:40:42.798+0900	INFO	log/input.go:148	Configured paths: [/home/kranian/sample/sample-files/scouter-*.json]
2019-10-02T17:40:42.798+0900	INFO	elasticsearch/client.go:170	Elasticsearch url: http://localhost:9200
2019-10-02T17:40:42.801+0900	INFO	elasticsearch/client.go:743	Attempting to connect to Elasticsearch version 7.3.0
2019-10-02T17:40:42.817+0900	INFO	input/input.go:114	Starting input of type: log; ID: 10102912876840846912 
2019-10-02T17:40:42.817+0900	INFO	cfgfile/reload.go:226	Loading of config files completed. 
```
 - 컴퓨터 성능에 따라 스카우터 Xlog로그 데이터가 전부 들어가는데는 1시간 이상 소요 된다.
# Step 4   
 - api 실행하여 정상적으로 엘라스틱 서치에 잘 스카우터 데이터가 잘 들어가고 있는지 확인한다. 
```
 curl -v GET "localhost:9200/filebeat-*/_count?pretty"
``` 
 - 마지막 브라우저에 아래 주소를 입력후 키바나에 접속해 스카우터 대시보드를 확인한다. 
```
 http://localhost:5601
```  
    
 
 
 
  
  