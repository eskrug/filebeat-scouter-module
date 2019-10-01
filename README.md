# filebeat-scouter-module
Filebeat scouter module

# Installtion Steps
1. Download and unzip Filebeat 
1. Edit the filebeat.yml configuration file
1. Dashboard Setup <br/>```./filbebeat setup``` 
1. Set scouter module <br/>```./filebeat modules enable scouter``` 
1. Edit the modules.d/scouter.yml configuration file
1. Start the daemon by running sudo <br/>```./filebeat -e -c filebeat.yml```

# modules.d/scouter.yml 설명 
```
- module: scouter
  # All logs
  log :
    enabled: true
    # Set custom paths for the log files. If left empty,
    # Filebeat will choose the paths depending on your OS.
    # 스카우터 메트릭 로그 파일 위치와 파일 패턴을 설정한다. 
    var.paths:
     - /path/scouter/server/ext_plugin_filelog/scouter-*.json   
                                                                     
```
# support elastic version 
1. elastic search version : 7.x 이상 
