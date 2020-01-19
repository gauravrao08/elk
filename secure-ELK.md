# secure ELK
https://www.elastic.co/blog/getting-started-with-elasticsearch-security

```
conf ==> /etc/elasticsearch/
data ==> /var/lib/elasticsearch/
data ==> /var/log/elasticsearch/
plugin ==>/usr/share/elasticsearch/plugins/
Home ==> /usr/share/elasticsearch/
bin ==> /usr/share/elasticsearch/bin
```

cd /usr/share/elasticsearch/bin
```
step 1:
./elasticsearch-certutil cert -out config/elastic-certificates.p12 -pass ""

step 2:
vim /etc/elasticsearch/elasticsearch.yaml 
----------
xpack.security.enabled: true
---------
#xpack.security.transport.ssl.enabled: true
#xpack.security.transport.ssl.verification_mode: certificate
#xpack.security.transport.ssl.keystore.path: elastic-certificates.p12
#xpack.security.transport.ssl.truststore.path: elastic-certificates.p12

##use ssl only if ssl is enabled othewise ignore

```
systemctl start elasticsearch

```
cd /usr/share/elasticsearch/bin

./elasticsearch-setup-passwords auto
#store  the username and password in some text file

optional ##if you have node other than master node then copy the master node conf to that node
cp ../elasticsearch-7.1.0-master/config/* config/
```

Modify the kibana conf file
vim /etc/kibana/kibana.yml

```
elasticsearch.username: "kibana"
elasticsearch.password: "hxeMv1r3J5Erxf0sFphv"   ## this is auto generated password from above

```
systemctl start kibana


login to kibana dashboard login with 

user : elastic

password : from the auto generated password of "elastic"
