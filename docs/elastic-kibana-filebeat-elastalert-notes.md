### Run Elasticsearch & Kibana as docker containers
```
git clone https://github.com/justmeandopensource/elk
cd elk/docker
mv docker-compose-v7.1.1.yml docker-compose.yml
sudo sysctl -w vm.max_map_count=262144
docker-compose up -d
```
### Install Filebeat on client machine (Ubuntu 18.04)
```
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
sudo apt-get install apt-transport-https
echo "deb https://artifacts.elastic.co/packages/7.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-7.x.list
sudo apt-get update && sudo apt-get install filebeat
```
### Configure Filebeat and enable system module
Edit filebeat configuration and update kibana and elasticsearch url
```
sudo vi /etc/filebeat/filebeat.yml
```
```
sudo filebeat modules enable system
sudo filebeat test config
sudo filebeat test output
sudo filebeat setup
sudo systemctl start filebeat
```
### Install and setup elastalert
```
sudo apt-get install -y python
sudo apt-get install -y python-pip python-dev libffi-dev libssl-dev
git clone https://github.com/Yelp/elastalert.git
cd elastalert
sudo pip install "setuptools>=11.3"
sudo pip install pyOpenSSL
sudo python setup.py install
sudo pip install "elasticsearch>=5.0.0"
cp config.example.yaml config.yaml
```
# install elast alert in centos 7
```
yum install -y python3
yum search pip | grep python3

pip3 -V 

git clone https://github.com/Yelp/elastalert.git

cd elastalert
pip3 install "setuptools>=11.3"
pip3 install pyOpenSSL
yum install gcc -y
yum install python3-devel -y

python3 setup.py install

pip3 install "elasticsearch>=5.0.0"
cp config.example.yaml config.yaml



yum install python36
yum install python36-devel
yum install python36-setuptools
```
Edit config.yaml and update es_host with IP address or dns name of the elasticsearch server
```
elastalert-create-index
```
Configure example_rules/example_frequency.yaml

### example_rules/example_frequency.yaml

```
name: mail subject
type: frequency
index: index_pattern_name*
num_events: 1
timeframe:
  hours: 1
filter:
- term:
    environment: "production"
alert:
- "email"
email:
- "gauravyadav1991gy@gmail.com"

smtp_host: "smtp_host_name"
smtp_port: 587
smtp_ssl: false
from_addr: "name_from_which_you_will_name@SMTP host"
smtp_auth_file: "/etc/mail/auth.yaml" 
```

### Testing a rule
```
elastalert-test-rule --config config.yaml example_rules/example_frequency.yaml
```
### Running elastalert
```
python -m elastalert.elastalert --verbose --rule example_frequency.yaml
```


## Postfix Gmail SMTP
Enable 2-factor authentication and Generate app password

### Install Postfix
```
sudo apt-get install postfix mailutils libsasl2-2 ca-certificates libsasl2-modules
```

### Postfix configuration to add

```
vim /etc/postfix/main.cf
-------------
inet_interfaces = localhost
or
inet_interfaces = all

and
inet_protocols = ipv4

## if you will not menstion ipv4 then you mail will be in queue

----------------

relayhost = [smtp.gmail.com]:587
smtp_sasl_auth_enable = yes
smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd
smtp_sasl_security_options = noanonymous
smtp_tls_CApath = /etc/ssl/certs
smtpd_tls_CApath = /etc/ssl/certs
smtp_use_tls = yes
```
```
cat /etc/postfix/sasl_passwd

[smtp.gmail.com]:587	gauravyadav1991gy@gmail.com:qpdm vkxv hdpc boja

sudo chmod 400 /etc/postfix/sasl_passwd
sudo postmap /etc/postfix/sasl_passwd
sudo systemctl restart postfix
```
```
echo "Testing" | mail -s "Test Email" gauravyadav1991gy@gmail.com
or
echo "This is the body" | mail -s "Subject" -aFrom:Harry\<ubuntu@ip-SES.ap-southeast-1.compute.internal\> gauravyadav1991gy@gmail.com
sudo postqueue -p
postqueue -f

```
