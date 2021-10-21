### App-search/Enterprise Search configuration
Some AWS instances do not have java installed by default. One way to install java is via the userdata script 
```shell
sudo amazon-linux-extras install -y  java-openjdk11

```
Generate dedicated certificates for our Enterprise Search, APM server and logstash hosts
```shell
acme.sh --issue --dns dns_aws -d ent-search.sciviz.co -d apm-server.sciviz.co -d logstash.sciviz.co
```
Create the PKCS12 file
```shell
cd ~/.acme.sh/ent-search.sciviz.co/ 
export  SSL_DOMAIN=ent-search.sciviz.co
export SSL_PASS=changeme
openssl pkcs12 -export \
-inkey ${SSL_DOMAIN}.key \
-in ${SSL_DOMAIN}.cer \
 -certfile fullchain.cer  \
-out ${SSL_DOMAIN}.pfx   \
-password env:SSL_PASS
```
Copy the pfx file to the ansible deployment folder `./files/certs`
```shell
cd role
git clone https://github.com/mirkenstein/ansible-role-elastic-enterprise-search.git
ln -s ansible-ent-search
```
In the current playbook the variables for the certificates to connect to elastic are the same as the ones we used for the kibana installation.
Run the playbook
```shell
ansible-playbook -i inventory-elk/ playbook-ent-search.yml 
```

#### Debugging the installation.
SSH to the host where enterprise search is installed and look at the logfiles `/var/log/messages` and `/var/log/enterprise-search/app-server.log`

It takes some time (couple of minutes ) for the enterprise search to start after service restart. 
Check the status in the main log file
`/var/log/enterprise-search/app-server.log`
If this one is empty or does not give you much information on the current issue then look at `/var/log/enterprise-search/app-server.log`