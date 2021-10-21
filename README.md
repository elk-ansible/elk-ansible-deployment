### Deploy and Configure  ELK with Terraform and Ansible
This is a complete end-to-end tutorial on provision and deployment of ELK stack.
We are using Terraform for the infrastructure deployment amd then we install and configure all the elasticsearch components via Ansible.

The steps are
#Terraform 
1. Run terraform to provision the infrastructure
2. Preparatory steps
   1. Configure your DNS provider to use the AWS Nameservers 
   2. Generate the SSL certificates for our domain with Lets Encrypt acme.sh script
   3. Create the entries for our ssh config files that we will use for the ansible inventory
   
3. Adjust the inventory file settings and run the Ansible playbooks to deploy the following components
  a. 3 Node Elasticsearch Cluster
  b. Kibana
  c. Enterprise Search
  d. Logstash
  e. APM server 


```shell
 ansible-playbook -i inventory-elk/ playbook-elasticsearch-cluster.yml
```
 
