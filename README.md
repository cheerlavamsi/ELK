# ELK
Elastic Logstash Kibana

Installation Steps.

Logstash with single file from filebeat

FileName : cat /etc/logstash/conf.d/sample.conf

input {
  beats {
    port => 5044
  }
}

filter {
      grok {
        match => { "message" => "%{MONTH:Month} %{MONTHDAY:MonthDay} %{HOUR:Hour}:%{MINUTE:Minute}:%{SECOND:Second} %{HOSTNAME:Hostname} %{HOSTNAME:ProgramName}:%{GREEDYDATA:LogMessage}" }
      }
     if [ProgramName] == "filebeat" {
            drop {}
      }
      mutate {
        remove_field => [ "Hostname" ]
      }
    }

output {
  elasticsearch {
    hosts => ["http://localhost:9200"]
  }
}

Logstash with multiple files from filebeat
On Filebeat Server
FileBeat Configuration:
File Name : /etc/filebeat/filebeat.yml
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /var/log/messages
  fields:
    logtype: system
- type: log
  enabled: true
  paths:
    - /var/log/nginx/access.log
  fields:
    logtype: nginx
filebeat.config.modules:
  path: ${path.config}/modules.d/*.yml
  reload.enabled: false
setup.template.settings:
  index.number_of_shards: 1
setup.kibana:
output.logstash:
  hosts: ["172.31.41.67:5044"]
processors:
  - add_host_metadata: ~
  - add_cloud_metadata: ~
  - add_docker_metadata: ~
  - add_kubernetes_metadata: ~
On Logstash Server
FileName : cat /etc/logstash/conf.d/sample.conf
input {
  beats {
    port => 5044
  }
}

filter {
  if [fields][logtype] == "system" {
      grok {
        match => { "message" => "%{MONTH:Month} %{MONTHDAY:MonthDay} %{HOUR:Hour}:%{MINUTE:Minute}:%{SECOND:Second} %{HOSTNAME:Hostname} %{HOSTNAME:ProgramName}:%{GREEDYDATA:LogMessage}" }
      }
     if [ProgramName] == "filebeat" {
            drop {}
      }
      mutate {
        remove_field => [ "Hostname" ]
        add_field => {
          "indexname" => "system"
        }
      }
    } 
  else if [fields][logtype] == "nginx" {
    grok {
        match => { "message" => "%{IP:SourceIP} - - \[%{MONTHDAY:Monthday}/%{MONTH:Month}/%{YEAR:Year}:%{HOUR:Hour}:%{MINUTE:Minute}:%{SECOND:Second} -%{INT}\] \"%{WORD:HTTP_METHOD} %{URIPATH:HTTP_PATH} %{WORD:PROTO_TYPE}/%{BASE16FLOAT:PROTO_VERSION}\" %{INT:HTTP_RESPONSE} %{INT:HTTP_CONTENT_SIZE} \"-|%{URI:URL}\" %{GREEDYDATA}" }
      }
    mutate {
        add_field => {
          "indexname" => "nginx"
        }
      }
    }
  else if [fields][logtype] == "tomcat" {
    grok {
      match => { "message" => "%{MONTHDAY:MonthDay}-%{MONTH:Month}-%{YEAR:Year} %{HOUR:Hour}:%{MINUTE:Minute}:%{SECOND:Second}\.%{INT:MilliSecond} %{WORD:LogLevel} \[%{HOSTNAME:ClassName}\] %{GREEDYDATA}"}
    }
    mutate {
        add_field => {
          "indexname" => "tomcat"
        }
      }
  }
}


output {
  elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "%{indexname}-%{+yyyy-MM-dd}"
  }
}