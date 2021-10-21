### Beats

Download the role

```shell
git clone https://github.com/mirkenstein/ansible-beats.git
cd ansible-beats
git checkout Feature-keystore
```

```shell
ansible-playbook -i inventory-elk/ playbook-beats-metricbeat.yml 
ansible-playbook -i inventory-elk/ playbook-beats-metricbeat-kibana.yml
ansible-playbook -i inventory-elk/ playbook-beats-filebeat.yml
```
Go to Kibana->Setting->Monitoring