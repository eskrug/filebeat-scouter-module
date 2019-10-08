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