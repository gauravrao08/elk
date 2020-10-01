```
s3 or cloud watch read only => role and user
```

```
logstash.conf

if [log][file][path] == "/var/log/mysql/invbox-prod.log"  {
                elasticsearch {

                        hosts => ["localhost:9200"]
                        index => "mysql-slow-rds-invbox-%{+YYYY.MM.dd}"
                        user => elastic
                        password => xAlubuMyB85g
                }
     }
```

```
*/36 * * * * /bin/awslogs  get  --aws-access-key-id  key --aws-secret-access-key key  /aws/rds/instance/invbox/postgresql --start='36 minutes' |sed  "s/^\/aws\/rds\/instance\/invbox\/postgresql .*#/#/g"  > /var/log/mysql/invbox-prod.log





*/37 * * * * /bin/awslogs  get /aws/rds/instance/invbox/postgresql --start='37 minutes' |sed  "s/^\/aws\/rds\/instance\/invbox\/postgresql .*#/#/g"  > /var/log/mysql/invbox.log

```
