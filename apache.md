

on ELK server

# Installation instructions for CentOS 7

## Assumptions
server.example.com __(ELK master)__

client.example.com __(client machine)__

sestatus 
```
SELinux status:                 enabled
SELinuxfs mount:                /sys/fs/selinux
SELinux root directory:         /etc/selinux
Loaded policy name:             targeted
Current mode:                   permissive
Mode from config file:          permissive
Policy MLS status:              enabled
Policy deny_unknown status:     allowed
Max kernel policy version:      31

```

getenforce 

vim /etc/selinux/config 
```
SELINUX=permissive
```
  ##if you are gett error "curl: (7) Failed to connect to server.example.com port 80: Connection refused" ##Stop the firewalld 
systemctl stop firewalld

## ELK Stack installation on server.example.com
###### Install Java 8
```
yum install -y java-1.8.0-openjdk
```
###### Import PGP Key
```
rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
```
###### Create Yum repository
```
cat >>/etc/yum.repos.d/elk.repo<<EOF
[ELK-6.x]
name=ELK repository for 6.x packages
baseurl=https://artifacts.elastic.co/packages/6.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
EOF
```
### Elasticsearch
###### Install Elasticsearch
```
yum install -y elasticsearch
```
rpm -qc elasticsearch    ##to check the path of all the package or conf file
journalctl --unit elasticsearch  ##to see the current logs

###### Enable and start elasticsearch service
```
systemctl daemon-reload
systemctl enable elasticsearch
systemctl start elasticsearch
```
### Kibana
###### Install kibana
```
yum install -y kibana
```
###### Enable and start kibana service
```
systemctl daemon-reload
systemctl enable kibana
systemctl start kibana
```
###### Install Nginx
```
yum install -y epel-release
yum install -y nginx
```
###### Create Proxy configuration
Remove server block from the default config file /etc/nginx/nginx.conf
And create a new config file
```
cat >>/etc/nginx/conf.d/kibana.conf<<EOF
server {
    listen 80;
    server_name server.example.com;
    location / {
        proxy_pass http://localhost:5601;
    }
}
EOF
```
###### Enable and start nginx service
```
systemctl enable nginx
systemctl start nginx
```
### Logstash
###### Install logstash
```
yum install -y logstash
```
vim /etc/logstash/conf.d/01-logstash-simple.conf

https://www.elastic.co/guide/en/logstash/current/logstash-config-for-filebeat-modules.html

```
input {
  beats {
    port => 5044
    host => "0.0.0.0"
  }
}
filter {
  if [fileset][module] == "apache2" {
    if [fileset][name] == "access" {
      grok {
        match => { "message" => ["%{IPORHOST:[apache2][access][remote_ip]} - %{DATA:[apache2][access][user_name]} \[%{HTTPDATE:[apache2][access][time]}\] \"%{WORD:[apache2][access][method]} %{DATA:[apache2][access][url]} HTTP/%{NUMBER:[apache2][access][http_version]}\" %{NUMBER:[apache2][access][response_code]} %{NUMBER:[apache2][access][body_sent][bytes]}( \"%{DATA:[apache2][access][referrer]}\")?( \"%{DATA:[apache2][access][agent]}\")?",
          "%{IPORHOST:[apache2][access][remote_ip]} - %{DATA:[apache2][access][user_name]} \\[%{HTTPDATE:[apache2][access][time]}\\] \"-\" %{NUMBER:[apache2][access][response_code]} -" ] }
        remove_field => "message"
      }
      mutate {
        add_field => { "read_timestamp" => "%{@timestamp}" }
      }
      date {
        match => [ "[apache2][access][time]", "dd/MMM/YYYY:H:m:s Z" ]
        remove_field => "[apache2][access][time]"
      }
      useragent {
        source => "[apache2][access][agent]"
        target => "[apache2][access][user_agent]"
        remove_field => "[apache2][access][agent]"
      }
      geoip {
        source => "[apache2][access][remote_ip]"
        target => "[apache2][access][geoip]"
      }
    }
    else if [fileset][name] == "error" {
      grok {
        match => { "message" => ["\[%{APACHE_TIME:[apache2][error][timestamp]}\] \[%{LOGLEVEL:[apache2][error][level]}\]( \[client %{IPORHOST:[apache2][error][client]}\])? %{GREEDYDATA:[apache2][error][message]}",
          "\[%{APACHE_TIME:[apache2][error][timestamp]}\] \[%{DATA:[apache2][error][module]}:%{LOGLEVEL:[apache2][error][level]}\] \[pid %{NUMBER:[apache2][error][pid]}(:tid %{NUMBER:[apache2][error][tid]})?\]( \[client %{IPORHOST:[apache2][error][client]}\])? %{GREEDYDATA:[apache2][error][message1]}" ] }
        pattern_definitions => {
          "APACHE_TIME" => "%{DAY} %{MONTH} %{MONTHDAY} %{TIME} %{YEAR}"
        }
        remove_field => "message"
      }
      mutate {
        rename => { "[apache2][error][message1]" => "[apache2][error][message]" }
      }
      date {
        match => [ "[apache2][error][timestamp]", "EEE MMM dd H:m:s YYYY", "EEE MMM dd H:m:s.SSSSSS YYYY" ]
        remove_field => "[apache2][error][timestamp]"
      }
    }
  }
}
output {
  elasticsearch {
    hosts => localhost
    manage_template => false
    index => "%{[@metadata][beat]}-%{[@metadata][version]}-%{+YYYY.MM.dd}"
  }
}
```

tail -f /var/log/elasticsearch/elasticsearch.log
check you can see metadata log 

###### Enable and Start logstash service
```
systemctl enable logstash
systemctl start logstash
```
## FileBeat installation on client.example.com
###### Create Yum repository
```
cat >>/etc/yum.repos.d/elk.repo<<EOF
[ELK-6.x]
name=ELK repository for 6.x packages
baseurl=https://artifacts.elastic.co/packages/6.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
EOF
```
###### Install Filebeat
```
yum install -y filebeat
```
vim /etc/filebeat/filebeat.yml 

```
#=========================== Filebeat inputs =============================

filebeat.inputs:

# Each - is an input. Most options can be set at the input level, so
# you can use different inputs for various configurations.
# Below are the input specific configurations.

- type: log

  # Change to true to enable this input configuration.
  enabled: true

  # Paths that should be crawled and fetched. Glob based paths.
  paths:
    - /var/log/httpd/*log
    #- c:\programdata\elasticsearch\logs\*
#============================== Kibana =====================================

# Starting with Beats version 6.0.0, the dashboards are loaded via the Kibana API.
# This requires a Kibana endpoint configuration.
setup.kibana:

  # Kibana Host
  # Scheme and port can be left out and will be set to the default (http and 5601)
  # In case you specify and additional path, the scheme is required: http://localhost:5601/path
  # IPv6 addresses should always be defined as: https://[2001:db8::1]:5601
  host: "server.example.com:5601"

#----------------------------- Logstash output --------------------------------
output.logstash:
  # The Logstash hosts
  hosts: ["server.example.com:5044"]

  # Optional SSL. By default is off.
  # List of root certificates for HTTPS server verifications
  #ssl.certificate_authorities: ["/etc/pki/tls/certs/logstash.crt"]

```

###### Enable and start Filebeat service
```
systemctl enable filebeat
systemctl start filebeat
```
journalctl --unit filebeat

##Enable apache log in kibana
```
filebeat test output

filebeat modules list

filebeat modules enable apache2
```

vim /etc/filebeat/modules.d/apache2.yml
```
- module: apache2
  # Access logs
  access:
    enabled: true

    # Set custom paths for the log files. If left empty,
    # Filebeat will choose the paths depending on your OS.
    #var.paths: ["/var/log/httpd/access_log"]

  # Error logs
  error:
    enabled: true

    # Set custom paths for the log files. If left empty,
    # Filebeat will choose the paths depending on your OS.
    #var.paths: ["/var/log/httpd/error_log"]

```
### Configure Kibana Dashboard
All done. Now you can head to Kibana dashboard and add the index.
