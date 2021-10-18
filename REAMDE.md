##Ansible Deployment 

We will be using Ansible playbooks to install and configure the ELK stack components.
Ansible is available in the package repositories fpr Liux and Mac.

For Windows OS use the Docker images from [WillHallOnline](https://www.willhallonline.co.uk/project/docker/docker-ansible/)

Ansible for Windows
```shell
PS C:\Users\user\Ansible> docker run --rm -it -v C:\Users\user\Ansible:/ansible  -v C:\Users\user\.ssh:/root/.ssh willhallonline/ansible:2.9-centos-7 /bin/sh
sh-4.2# 
```

### Ansible SSH Hostfile for Inventory
We already launched the EC2 nodes with Terraform.
In order to generate the `$HOME/.ssh/config` file entries for our infrastructure we can use the `aws cli` utiliy

```shell
aws ec2 describe-instances \
--filters Name=tag:Name,Values=tf*  "Name=instance-state-name, Values=running"  \
--query 'Reservations[*].Instances[*].{ PublicIpAddress:PublicIpAddress,Name:Tags[?Key==`Name`]|[0].Value }'  \
--region us-west-2 \
--output text \
--profile hgs |awk '{print "Host "tolower($1)"\nHostName "$2}'
```            
This would output the 3 node entries for our `.ssh/config`  fille
```shell
Host tf-elk-es02
HostName 35.111.22.243
Host tf-elk-es01
HostName 34.aa.bb.0
Host tf-elk-es03
HostName 34.xx.yy.10
```
#### Get the Ansible role for Elasticsearch 
We will be checking out the code for the roles instead of using Ansible galaxy  
 
```shell
cd Ansible/roles
git clone https://github.com/elastic/ansible-elasticsearch
git checkout tags/v7.15.1

```
Test Ansible Inventory 
```shell
 ansible -i inventory-elk/hosts.conf all  -m ping
```
returns 
```shell
...
tf-elk-es01 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": false,
    "ping": "pong"
}
...
```


### SSL Cert with acme.sh
We already set up our DNS server to point to the deployd EC2s. We will also need a valid SSL certificates for our ELK stack deployment.
In this example here we're using the Acme.sh API utility and the GoDaddy provider.
[acme.sh](https://github.com/acmesh-official/acme.sh)
[Use GoDaddy.com domain API to automatically issue cert](https://github.com/acmesh-official/acme.sh/wiki/dnsapi#4-use-godaddycom-domain-api-to-automatically-issue-cert)

```shell
export GD_Key="xxx"
export GD_Secret="PPPX123"
```
```shell
acme.sh --issue --dns dns_gd -d kibana.sciviz.co -d es2-sb.dfh.ai -d es3-sb.dfh.ai -d es4-sb.dfh.ai -d es5-sb.dfh.ai -d kibana-sb.dfh.ai  --force

[Tue Feb  2 07:42:03 PM CST 2021] Your cert is in  /home/user/.acme.sh/kibana.sciviz.co/kibana.sciviz.co.cer 
[Tue Feb  2 07:42:03 PM CST 2021] Your cert key is in  /home/user/.acme.sh/kibana.sciviz.co/kibana.sciviz.co.key 
[Tue Feb  2 07:42:03 PM CST 2021] The intermediate CA cert is in  /home/user/.acme.sh/kibana.sciviz.co/ca.cer 
[Tue Feb  2 07:42:03 PM CST 2021] And the full chain certs is there:  /home/user/.acme.sh/kibana.sciviz.co/fullchain.cer 
[user@localhost AWS]$ 
```



### Generate the PKCS12 (p12, aka pfx) file 
Go to the folder where the certificates were generated with `acme.sh`
```shell
$ openssl pkcs12 -export  -inkey kibana.sciviz.co.key  -in kibana.sciviz.co.cer -certfile fullchain.cer -out es-sb.dfh.ai.p12 
Enter Export Password:
Verifying - Enter Export Password:

export  SSL_DOMAIN=kibana.sciviz.co
export SSL_PASS=changme
openssl pkcs12 -export \
 -inkey ${SSL_DOMAIN}.key \
 -in ${SSL_DOMAIN}.cer \
 -certfile fullchain.cer -out ${SSL_DOMAIN}.pfx \
  env:$SSL_PASS
[user@localhost kibana.sciviz.co]$ 
```

            

### GoDaddy DNS Change

```shell
export GD_Key="xxx"
export GD_Secret="PPPX123"
``` 

##### DNS Update GoDaddy Nameservers
We set up with Terraform a Private and Public hosted zones. We would need to point our GoDaddy domain to the Custom Nameservers. We take these values
AWS Console>Route53>Public Zone>NS Type  Value/Route traffic to
These nameserver have the following format
```shell
ns-234.awsdns-33.com.
ns-999.awsdns-59.net.
ns-1234.awsdns-55.co.uk.
ns-123.awsdns-22.org.
```
We use the GoDaddy API for the custom DNS Nameserver update.
```shell

curl -X PATCH https://api.godaddy.com/api/v1/domains/sciviz.co \
-H "Authorization: sso-key ${GD_Key}:${GD_Secret}" \
-H "Content-Type: application/json" \
--data  '{	"nameServers": [
"ns-1143.awsdns-14.org",
"ns-154.awsdns-19.com",
"ns-1828.awsdns-36.co.uk",
"ns-576.awsdns-08.net"
]
  }'
  ```
##### Check if the change was propagated
```shell
curl -X GET  https://api.godaddy.com/api/v1/domains/sciviz.co \
-H "Authorization: sso-key ${GD_Key}:${GD_Secret}" \
  -H "Content-Type: application/json" 
```

##### DNS Set A record

For documentation purposes I am posting also how to change the DNS to the hostnames directly.
```shell
curl -X PATCH https://api.godaddy.com/v1/domains/sciviz.co/records \
-H "Authorization: sso-key ${GD_Key}:${GD_Secret}" \
  -H "Content-Type: application/json" \
  --data '[
  {"type": "A","name": "es1","data": "34.209.128.5","ttl": 3600},
   {"type": "A","name": "es2","data": "52.38.139.226","ttl": 3600},
   {"type": "A","name": "es3-sb","data": "34.209.124.156","ttl": 3600}
  ]'

```


##### Finaly Let's Install our 3 node elasticsearch cluster
```shell
 ansible-playbook -i inventory-elk/ playbook-elasticsearch-cluster.yml
```
 