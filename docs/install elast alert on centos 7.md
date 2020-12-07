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
elastalert-test-rule --config config.yaml example_rules/rule_name.yaml


### Testing a rule
```
# this will not send mail it is just for checking error if any 
```
python3 -m elastalert.elastalert --verbose --rule rule_name.yaml

# this will send mail 
#uuse python3 
```

```
#if you have multiple rules then create folder rules and put all yaml file inside it and change config.yaml ==> "rules_folder: rules"
python -m elastalert.elastalert --config ./config.yaml --verbose
will will load all the rules inside the rules folder 
```


