```
*/13 * * * * /bin/awslogs get /aws/rds/instance/bikedekho-master/slowquery --start='13 minutes' |  sed  "s/\/aws\/rds\/instance\/bikedekho-master\/slowquery bikedekho-master //g" | perl /root/.script/parser-perl  > /var/log/mysql/bikedekho-master.log
```
