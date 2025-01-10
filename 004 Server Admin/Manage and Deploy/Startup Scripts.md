## [Startup Scripts](https://clickhouse.com/docs/en/operations/startup-scriptshttps://clickhouse.com/docs/en/operations/startup-scripts)
### Additional resources
[Replace delimited block of text in file with the contents of another file](https://stackoverflow.com/questions/2699666/replace-delimited-block-of-text-in-file-with-the-contents-of-another-file)
#### Question
Let's assume with have the following simplified files:

server.xml
```
<?xml version='1.0' encoding='UTF-8'?>
<Server port="8005" shutdown="SHUTDOWN">
  <Service name="Catalina">
    <Connector port="80" protocol="HTTP/1.1"/>
    <Engine name="Catalina" defaultHost="localhost">
      <!-- BEGIN realm -->
        <sometags/>
        <sometags/>
      <!-- END realm -->
      <Host name="localhost" appBase="webapps"/>
    </Engine>
  </Service>
</Server>
```
realm.xml
```
<Realm className="org.apache.catalina.realm.UserDatabaseRealm"
       resourceName="UserDatabase"/>
```
I want to run a script and have realm.xml replace the contents between the <!-- BEGIN realm --> and <!-- END realm --> lines.    
If realm.xml changes then whenever the script is run again it will replace the lines again with the new contents of realm.xml.    
This is intended to be run in /etc/init.d/tomcat on startup of the service on multiple installations on which the realm is going to be different.

#### Answer 1
```
sed -i -ne '/<!-- BEGIN realm -->/ {p; r realm.xml' -e ':a; n; /<!-- END realm -->/ {p; b}; ba}; p' server.xml
```
#### Answer 2
```
TOTAL_LINES=`cat server.xml | wc -l`
BEGIN_LINE=`grep -n -e '<!-- BEGIN realm -->' server.xml | cut -d : -f 1`
END_LINE=`grep -n -e '<!-- END realm -->' server.xml | cut -d : -f 1`
TAIL_LINES=$(($TOTAL_LINES-$END_LINE))

head -n $BEGIN_LINE server.xml > server2.xml
cat realm.xml > server2.xml
tail -n $TAIL_LINES server.xml > server2.xml
```
#### Answer 3
```
awk 'FNR==NR{ _[++d]=$0;next}
/BEGIN realm/{
  print
  for(i=1;i<=d;i++){ print _[i] }
  f=1;next
}
/END realm/{f=0}!f' realm.xml server.xml > temp && mv temp server.xml
```
#### Answer 4
```
sed -n \
  -e "1,/<\!-- BEGIN realm -->/ p" \
  -e"/<\!-- END realm -->/,$ p" \
  -e "/<\!-- BEGIN realm -->/ r realm.xml" \
  server.xml
```
#### Answer 4
```
cat <<-'EOF' | sed -e 's/^ *//' -e 's/ *$//' | ed -s server.xml
   H
   /BEGIN realm/i
   .
   /BEGIN realm/+1,/END realm/-1d
   .-1r realm.xml
   wq
EOF
```

