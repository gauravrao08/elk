```
*/13 * * * * /bin/awslogs get /aws/rds/instance/bikedekho-master/slowquery --start='13 minutes' |  sed  "s/\/aws\/rds\/instance\/bikedekho-master\/slowquery bikedekho-master //g" | perl /root/.script/parser-perl  > /var/log/mysql/bikedekho-master.log
```

```# This perl script parses a MySQL slow_queries log file
# ignoring all queries less than $min_time and prints
# the required data in a signle line, for easy parsing of it
# by any log parser.
#
#
# Usage: mysql_slow_log_parser logfile
#
# ------------------------
# SOMETHING TO THINK ABOUT (aka: how to read output)
# ------------------------
#
# Also, it does to regex substitutions to normalize
# the queries...
#
#   $query_string =~ s/\d+/XXX/g;
#   $query_string =~ s/([\'\"]).+?([\'\"])/$1XXX$2/g;
#
# These replace numbers with XXX and strings found in
# quotes with XXX so that the same select statement
# with different WHERE clauses will be considered
# as the same query.
#
# so these...
#
#   SELECT * FROM offices WHERE office_id = 3;
#   SELECT * FROM offices WHERE office_id = 19;
#
# become...
#
#   SELECT * FROM offices WHERE office_id = XXX;
#
#
# And these...
#
#   SELECT * FROM photos WHERE camera_model LIKE 'Nikon%';
#   SELECT * FROM photos WHERE camera_model LIKE '%Olympus';
#
# become...
#
#   SELECT * FROM photos WHERE camera_model LIKE 'XXX';
#
#
# ---------------------
# THIS MAY BE IMPORTANT (aka: Probably Not)
# ---------------------
#
# *SO* if you use numbers in your table names, or column
# names, you might get some oddities, but I doubt it.
# I mean, how different should the following queries be
# considered?
#
#   SELECT car1 FROM autos_10;
#   SELECT car54 FROM autos_11;
#
# I don't think so.
#


$min_time       = 0;    # Skip queries less than $min_time
$min_rows       = 0;
$max_display    = 10;   # Truncate display if more than $max_display occurances of a query

#print "\n Starting... \n";

$query_string   = '';
$time           = 0;
$new_sql        = 0;
$user                                   = '';
$host                                   = '';

##############################################
# Loop Through The Logfile
##############################################

while (<>) {

        # Skip Bogus Lines

        next if ( m|/.*mysqld, Version:.+ started with:| );
        next if ( m|Tcp port: \d+  Unix socket: .*mysql.sock| );
        next if ( m|Time\s+Id\s+Command\s+Argument| );
        next if ( m|administrator\s+command:| );

        # if ( /Query_time:\s+(.*)\s+Lock_time:\s+(.*)\s/ ) {
        #if ( /Query_time:\s+(.*)\s+Lock_time:\s+(.*)\s+Rows_examined:\s+(\d+)/ ) {

        if ( /User\SHost:\s+([\S]*)\s+\S\s+\S*\s+\[+(.*)\][\s+\S]*/ ) {

  # Try This if the above doesn't work
  # if ( /User\SHost:\s+(.*)\s\S\s+\[+(.*)\][\s+\S]*/ ) {

        $user = $1;
                $host   = $2;
                next;

        }

        if ( /Query_time:\s+(.*)\s+Lock_time:\s+(.*)\s+Rows_examined:\s+(.*)/ ) {

                $time    = $1;
                $rows    = $3;
                $new_sql = 1;
                # print "found $1 $3\n";
                next;

        }


        if ( /^\#/ && $query_string ) {

                        if (($time > $min_time) && ($rows >= $min_rows)) {
                                $orig_query = $query_string;

                                $query_string =~ s/\d+/XXX/g;
                                $query_string =~ s/'([^'\\]*(\\.[^'\\]*)*)'/'XXX'/g;
                                $query_string =~ s/"([^"\\]*(\\.[^"\\]*)*)"/"XXX"/g;
                                #$query_string =~ s/([\'\"]).+?([\'\"])/$1XXX$2/g;
                                #$query_string =~ s/\s+/ /g;
                                #$query_string =~ s/\n+/\n/g;

                                $users{$query_string} = $user;
                                $hosts{$query_string} = $host;
                                push @{$queries{$query_string}}, $time;
                                push @{$queries_rows{$query_string}}, $rows;
                                $queries_tot{$query_string} += $time;
                                $queries_orig{$query_string} = $orig_query;
                                $query_string = '';

                        }

        } else {

                if ($new_sql) {
                        $query_string = $_;
                        $new_sql = 0;
                } else {
                        $query_string .= $_;
                }
        }

}


##############################################
# Display Output
##############################################

foreach my $query ( sort { $queries_tot{$b} <=> $queries_tot{$a} } keys %queries_tot )  {
        my $total = 0;
        my $cnt = 0;
        my @seconds = sort { $a <=> $b } @{$queries{$query}};
        my @rows    = sort { $a <=> $b } @{$queries_rows{$query}};
        ($total+=$_) for @seconds;
        ($cnt++) for @seconds;

        print "User: ".$users{$query};
        print " | Host: ".$hosts{$query};
        print " | NoQuery: ".@{$queries{$query}};
        print " | TotalTime: " . $total;
        print " | AverageTime: ".($total/$cnt);
        print " | TimeTaken: ";
        print @seconds > $max_display ? "$seconds[0] to $seconds[-1]" : sec_joiner(\@seconds);
        print " seconds to complete ";
        print " | RowsAnalyzed: ";
  print @rows > $max_display ? "$rows[0] - $rows[-1]": sec_joiner(\@rows);
  $orignal = $query;
        $query =~ s/\n/ /g;
        $queries_orig{$orignal} =~ s/\n/ /g;
        print " | Pattern: $query";
        print " | Orignal: ";
        print $queries_orig{$orignal}."\n";
}


sub sec_joiner {
        my ($seconds) = @_;
        $string = join(", ", @{$seconds});
        $string =~ s/, (\d+)$/ and $1/;
        return $string;
}

exit(0);

```


```
td-agent-bit.conf

[SERVICE]
    Flush        5
    Daemon       Off
    #Log_Level    info
    Log_Level    Debug
    Parsers_File parsers.conf
@INCLUDE bikedekho-slave.conf
@INCLUDE bikedekho-master.conf
[INPUT]
    Mem_Buf_Limit 500MB
    Name tail
    DB fluent-bit.db
    Buffer_Max_Size 500MB
    Buffer_Chunk_Size 128k
    tag rds-slave-1
    Path /var/log/mysql/rds-slow-slave1.log
    Parser rds-slow
    Refresh_Interval 5
[INPUT]
    Mem_Buf_Limit 500MB
    Name tail
    DB fluent-bit.db
    Buffer_Max_Size 500MB
    Buffer_Chunk_Size 128k
    tag rds-slave-2
    Path /var/log/mysql/rds-slow-slave2.log
    Parser rds-slow
    Refresh_Interval 5
[INPUT]
    Mem_Buf_Limit 500MB
    Name tail
    DB fluent-bit.db
    Buffer_Max_Size 500MB
    Buffer_Chunk_Size 128k
    tag rds-master
    Path /var/log/mysql/rds-slow-master.log
    Parser rds-slow
    Refresh_Interval 5
[INPUT]
    Mem_Buf_Limit 500MB
    Name tail
    DB fluent-bit.db
    Buffer_Max_Size 500MB
    Buffer_Chunk_Size 128k
    tag mobile-verification-master
    Path /var/log/mysql/rds-slow-mobile-verification.log
    Parser rds-slow
    Refresh_Interval 5
[INPUT]
    Mem_Buf_Limit 500MB
    Name tail
    DB fluent-bit.db
    Buffer_Max_Size 500MB
    Buffer_Chunk_Size 128k
    tag mobile-verification-slave
    Path /var/log/mysql/rds-slow-mobile-verification-slave.log
    Parser rds-slow
    Refresh_Interval 5

[FILTER]
    Record hostname rds-slave-1
    Name record_modifier
    Match rds-slave-1
[FILTER]
    Record hostname rds-slave-2
    Name record_modifier
    Match rds-slave-2
[FILTER]
    Record hostname rds-master
    Name record_modifier
    Match rds-master
[FILTER]
    Record hostname mobile-verfifcation-master
    Name record_modifier
    Match mobile-verification-master
[FILTER]
    Record hostname mobile-verfifcation-slave
    Name record_modifier
    Match mobile-verification-slave

[OUTPUT]
    Index slow-query-rds-slave-1
    Name es
    Logstash_Format On
    Retry_Limit 1
    Host 1.0.25.IP
    Logstash_Prefix slow-query-rds-slave-1
    Type slow-query-rds-slave-1
    Port 9200
    Match rds-slave-1
[OUTPUT]
    Index slow-query-rds-slave-2
    Name es
    Logstash_Format On
    Retry_Limit 1
    Host 1.0.25.216
    Logstash_Prefix slow-query-rds-slave-2
    Type slow-query-rds-slave-2
    Port 9200
    Match rds-slave-2
[OUTPUT]
    Index slow-query-rds-slave-master
    Name es
    Logstash_Format On
    Retry_Limit 1
    Host 1.0.25.IP
    Logstash_Prefix slow-query-rds-slave-master
    Type slow-query-rds-slave-master
    Port 9200
    Match rds-master
[OUTPUT]
    Index slow-query-rds-slave-mobile-master
    Name es
    Logstash_Format On
    Retry_Limit 1
    Host 1.0.25.IP
    Logstash_Prefix slow-query-rds-slave-mobile-master
    Type slow-query-rds-slave-mobile-master
    Port 9200
    Match mobile-verification-master
[OUTPUT]
    Index slow-query-rds-slave-mobile-slave
    Name es
    Logstash_Format On
    Retry_Limit 1
    Host 1.0.25.IP
    Logstash_Prefix slow-query-rds-slave-mobile-slave
    Type slow-query-rds-slave-mobile-slave
    Port 9200
    Match mobile-verification-slave
[INPUT]
    Mem_Buf_Limit 500MB
    Name tail
    DB fluent-bit-cluster.db
    Buffer_Max_Size 500MB
    Buffer_Chunk_Size 128k
    tag auroa-cluster-info
    Path /var/log/mysql/cardekho-info-cluster.log
    Parser rds-slow
    Refresh_Interval 5
[FILTER]
    Record hostname auroa-cluster-info
    Name record_modifier
    Match auroa-cluster-info
[OUTPUT]
    Index slow-query-rds-slave-info-cluster
    Name es
    Logstash_Format On
    Retry_Limit 1
    Host 1.0.25.IP
    Logstash_Prefix slow-query-rds-slave-info-cluster
    Type slow-query-rds-slave-info-cluster
    Port 9200
    Match auroa-cluster-info

```

```
bikedekho-master.conf

[INPUT]
    Mem_Buf_Limit 500MB
    Name tail
    DB fluent-bit.db
    Buffer_Max_Size 500MB
    Buffer_Chunk_Size 128k
    tag bikedekho-master
    Path /var/log/mysql/bikedekho-master.log
    Parser rds-slow
    Refresh_Interval 5
[FILTER]
    Record hostname bikedekho-master
    Name record_modifier
    Match bikedekho-master

[OUTPUT]
    Index slow-query-bikedekho-master
    Name es
    Logstash_Format On
    Retry_Limit 1
    Host 1.0.25.IP
    Logstash_Prefix slow-query-bikedekho-master
    Type slow-query-bikedekho-master
    Port 9200
    Match bikedekho-master 

```

```
parsers.conf

[PARSER]
    Name    mysql_slow0
    Format  regex
    Regex   #(\s)*Time:(\s)*(?<timestamp>[0-9\s:]*)
    Time_Key    time
    Time_Format %Y-%m-%dT%H:%M:%S.%L
    Time_Keep   Off
[PARSER]
    Name    mysql_slow1
    Format  regex
    Regex   ^(#) (?<USERandHOST>[A-Za-z].*@[A-Za-z].*): (?<USERNAMEandHOST>[A-Za-z\[\]].* @ *[A-Za-z0-9^ \n\t].*) (?<IP>\[.*\])
    Time_Key    time
    Time_Format %Y-%m-%dT%H:%M:%S.%L
    Time_Keep   Off

[PARSER]
    Name    mysql_slow2
    Format  regex
    Regex  (#)(\s)*Thread_id(\s)*:(\s)*(?<Thread_id>[0-9]*)(\s)*Schema:(\s)*(?<Schema>[^ ]*)(\s)*QC_hit:(\s)*(?<QC_hit>[a-zA-Z0-9]*)
    Time_Key    time
    Time_Format %Y-%m-%dT%H:%M:%S.%L
    Time_Keep   Off


[PARSER]
    Name    mysql_slow3
    Format  regex
    Regex  (#)(\s)*Query_time:(\s)*(?<QueryTime>[0-9.]{1,100})(\s)*Lock_time:(\s)*(?<LockKey>[0-9.]*)(\s)*Rows_sent:(\s)*(?<Rows_sent>[0-9]*)(\s)*Rows_examined:(\s)*(?<Rows_examined>[0-9]*)
    Time_Key    time
    Time_Format %Y-%m-%dT%H:%M:%S.%L
    Time_Keep   Off


[PARSER]
    Name    mysql_slow4
    Format  regex
    Regex  (#)(\s)*Rows_affected:(\s)*(?<Rows_affected>[0-9]*)
    Time_Key    time
    Time_Format %Y-%m-%dT%H:%M:%S.%L
    Time_Keep   Off


[PARSER]
    Name    mysql_slow5
    Format  regex
    Regex  SET(\s)*timestamp=(?<timestamp>[0-9;]*)
    Time_Key    time
    Time_Format %Y-%m-%dT%H:%M:%S.%L
    Time_Keep   Off


[PARSER]
    Name    mysql_slow6
    Format  regex
    Regex  (?<query>.*)
    Time_Key    time
    Time_Format %Y-%m-%dT%H:%M:%S.%L
    Time_Keep   Off

