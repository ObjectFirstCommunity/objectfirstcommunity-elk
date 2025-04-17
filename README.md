# ELK Stack Installation Guide

This guide provides step-by-step instructions for installing and configuring the Elastic Stack (Elasticsearch, Logstash, and Kibana) on Ubuntu, with a specific focus on Veeam integration.

## Table of Contents
- [Prerequisites](#prerequisites)
- [Elasticsearch Installation](#elasticsearch-installation)
- [Kibana Installation](#kibana-installation)
- [Logstash Installation](#logstash-installation)
- [Veeam Integration](#veeam-integration)
- [Troubleshooting](#troubleshooting)

## Prerequisites

- Ubuntu Server (recommended: 20.04 LTS or newer)
- Sudo access
- At least 4GB RAM
- 10GB+ free disk space

## Elasticsearch Installation

1. Import the Elasticsearch public GPG key:

```bash
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg
```

2. Install the apt-transport-https package:

```bash
sudo apt-get install apt-transport-https
```

3. Add the Elasticsearch repository:

```bash
echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/8.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-8.x.list
```

4. Update packages and install Elasticsearch:

```bash
sudo apt-get update && sudo apt-get install elasticsearch
```

5. Configure system service:

```bash
sudo /bin/systemctl daemon-reload
sudo /bin/systemctl enable elasticsearch.service
```

### Elasticsearch Configuration

1. Edit the Elasticsearch configuration file:

```bash
sudo vi /etc/elasticsearch/elasticsearch.yml
```

2. Make the following changes:

```yaml
# Uncomment this line
node.name: node-1

# Change this to allow external connections
network.host: 0.0.0.0

# Configure discovery
discovery.seed_hosts: ["127.0.0.1"]

# Enable X-Pack security
xpack.security.enabled: true

# Configure initial master nodes
cluster.initial_master_nodes: ["node-1"]
```



3. Modify service timeout settings:

```bash
sudo nano /usr/lib/systemd/system/elasticsearch.service
```

4. Set the timeout to a higher value:

```
TimeoutStartSec=900
```

5. Change log viewing permissions:

```bash
sudo chmod 755 -R /var/log/elasticsearch/
```

6. Start Elasticsearch:

```bash
sudo systemctl start elasticsearch.service
```

7. Set the Elastic password:

```bash
# The auto-generated elastic password is displayed during installation
# To reset/change the password, use:
sudo /usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic
```

![alt text](images\image1.png)

8. Export the password as an environment variable for convenience:

```bash
export ELASTIC_PASSWORD="your_new_password"
```

9. Verify the installation (with security enabled):

```bash
curl -X GET "https://localhost:9200/_cluster/health?pretty" -u elastic:$ELASTIC_PASSWORD --insecure
```

![alt text](images\image2.png)

## Kibana Installation

1. Check available Kibana versions (ensure it matches your Elasticsearch version):

```bash
sudo apt list kibana -a
```

2. Install Kibana:

```bash
sudo apt install kibana
```

3. Configure Kibana:

```bash
sudo vi /etc/kibana/kibana.yml
```

4. Change the server host to allow external connections:

```yaml
server.host: "0.0.0.0"
```

![alt text](images\image2.png)

5. Start and enable Kibana:

```bash
sudo systemctl daemon-reload
sudo systemctl enable kibana.service
sudo systemctl start kibana.service
```

6. Verify Kibana is running (it will be available on port 5601):

```bash
curl http://localhost:5601/status -I
```

### Kibana Security Configuration

Since X-Pack security is enabled, you need to configure Kibana to connect securely:

1. Generate an enrollment token for Kibana:

```bash
cd /usr/share/elasticsearch/bin
sudo ./elasticsearch-create-enrollment-token --scope kibana
```

> ![alt text](images\image4.png)

2. Copy this token as you'll need it during Kibana's first start.

3. Start Kibana:

```bash
sudo systemctl start kibana.service
```

4. Check the Kibana logs to see the verification code prompt:

```bash
sudo tail -f /var/log/kibana/kibana.log
```

5. Generate a verification code:

```bash
cd /usr/share/kibana/bin
sudo ./kibana-verification-code
```



6. Copy the token and enter the verification code in the Kibana web interface (http://your_server_ip:5601).

![alt text](images\image5.png)
![alt text](images\image6.png)
![alt text](images\image7.png)
7. Generate encryption keys for alerts:

```bash
cd /usr/share/kibana/bin
sudo ./kibana-encryption-keys generate
```
![alt text](images\image8.png)

8. Add the generated keys to your kibana.yml file:

```bash
sudo vi /etc/kibana/kibana.yml
```

9. Add the generated encryption keys at the end of the file:

```yaml
xpack.encryptedSavedObjects.encryptionKey: "generated_encryption_key"
xpack.reporting.encryptionKey: "generated_reporting_key"
xpack.security.encryptionKey: "generated_security_key"
```

10. Restart Kibana to apply the changes:

```bash
sudo systemctl restart kibana.service
```


## Logstash Installation

1. Install dependencies and Logstash:

```bash
sudo apt install openjdk-8-jre-headless
sudo apt update
sudo apt install logstash
```

2. Create a dedicated user for Logstash:

```bash
sudo useradd -r -g logstash logstash
sudo chown -R logstash:logstash /usr/share/logstash /var/log/logstash
```

## OOTBI

### Create Elasticsearch Index Template

Importing Templates
When importing, you must follow the correct order - component templates first, then index templates.

change directory into the Templates folder and run these commands 

### Import component templates to the new Elasticsearch instance

curl -X PUT "new-host:9200/_component_template/syslog-hardware" -H "Content-Type: application/json" -d @component-hardware.json
curl -X PUT "new-host:9200/_component_template/syslog-security" -H "Content-Type: application/json" -d @component-security.json
curl -X PUT "new-host:9200/_component_template/syslog-services" -H "Content-Type: application/json" -d @component-services.json
curl -X PUT "new-host:9200/_component_template/syslog-settings" -H "Content-Type: application/json" -d @component-settings.json


### Import the index template after all component templates are in place:


curl -X PUT "new-host:9200/_index_template/syslog-template" -H "Content-Type: application/json" -d @syslog-template.json


### Configure Logstash Pipeline for Veeam

Copy the logstash.conf file from the Template folder to /etc/logstash/conf.d



restart both elasticsearch and logstash

sudo systemcl restart logstash
sudo systemctl restart elasticsearch


## Troubleshooting

### Common Issues

1. **Elasticsearch fails to start**
   - Check logs: `sudo journalctl -u elasticsearch.service`
   - Verify sufficient memory: `free -m`
   - Check disk space: `df -h`

2. **Kibana cannot connect to Elasticsearch**
   - Verify Elasticsearch is running: `curl http://localhost:9200`
   - Check network settings in kibana.yml

3. **Logstash parsing errors**
   - View Logstash logs: `sudo tail -f /var/log/logstash/logstash-plain.log`
   - Test your grok patterns at [Grok Debugger](https://grokdebug.herokuapp.com/)


## Additional Resources

- [Elasticsearch Documentation](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html)
- [Kibana Documentation](https://www.elastic.co/guide/en/kibana/current/index.html)
- [Logstash Documentation](https://www.elastic.co/guide/en/logstash/current/index.html)

