# prometheus-target-self-signed-Certificate
This repo demonstrate the basic steps for  setting up self-signed Certificate between Prometheus and targets

## prerequisites

* Make sure prometheus is installed and running.
* You should have any exporter on other machine running. (in my case, it's node_exporter for linux)
* Prometheus can pull metrics from the exporter.

## Steps

* Generate self-signed certificates on the target node.
  ```bash
  sudo openssl req -new -newkey rsa:2048 -days 365 -nodes -x509 -keyout node_exporter.key -out node_exporter.crt -subj "/C=EG/ST=Egypt/L=Alex/O=MyOrg/CN=localhost"
  ```
  You can change the content of the ```subj``` as you like.
  
  Now You should have 2 files
  * ```node_exporter.crt```
  * ```node_exporter.key```
    
* Create a folder to store these files.

  ```bash
  sudo mkdir /etc/node_exporter
  sudo mv node_exporter.* /etc/node_exporter/
  ```

* Create a config.ymlfile with the below content in the same directory
  ```yml
  tls_server_config:
    cert_file: node_exporter.crt
    key_file: node_exporter.key
  ```

  Now you should have the following
  ```bash
  ls /etc/node_exporter/
  config.yml  node_exporter.crt  node_exporter.key
  ```
  
* These files should have the same permissions as the node-exporter service

* Update the node_exporter service with TLS config

  ```bash
  sudo vi /etc/systemd/system/node_exporter.service
  ```
  ```service
  [Unit]
  Description=Node Exporter
  Wants=network-online.target
  After=network-online.target
  [Service]
  User=node_exporter
  Group=node_exporter
  Type=simple
  ExecStart=/usr/local/bin/node_exporter --web.config.file="/etc/node_exporter/config.yml"
  [Install]
  WantedBy=multi-user.target
  ```

* Reload the daemon and restart the node_exporter

  ```bash
  sudo systemctl daemon-reload
  sudo systemctl restart node_exporter
  ```

* Test the tls locally

  ```bash
  curl https://localhost:9100/metrics
  ```
  This should give an error.

  ```bash
  curl -k https://localhost:9100/metrics
  ```
  This should work because ```-k``` is for insecure connection.
  
* Now we need to configure prometheus server.

* Copy ```node_exporter.crt``` file from the node exporter server to the Prometheus server at ```/etc/prometheus``` (You can use ```scp```)

* Give the CRT the right permission

  ```bash
  sudo chown prometheus:prometheus node_exporter.crt
  ```

* Edit the job part in the prometheus configuration ```/etc/prometheus/prometheus.yml```

  ```yml
  scrape_configs:
    - job_name: 'prometheus_master'
      scrape_interval: 5s
      static_configs:
        - targets: ['localhost:9090']
    - job_name: 'Linux Server'
      scheme: https
      tls_config:
        ca_file: /etc/prometheus/node_exporter.crt
        insecure_skip_verify: true
      scrape_interval: 5s
      static_configs:
        - targets: ['192.168.216.129:9100']
  ```
  
* Restart prometheus

  ```bash
  sudo systemctl restart prometheus.service
  ```

* VOILA, You are ready to go.
  
