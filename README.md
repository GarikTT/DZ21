Занятие 21 - Prometheus
Цели занятия -
узнать, из чего состоит экосистема Prometheus;
собирать и смотреть метрики
Домашнее задание - 
Настроить дашборд с 4-мя графиками -
память;
процессор;
диск;
сеть.
Настроить на одной из систем:
zabbix (использовать screen (комплексный экран);
prometheus - grafana.

В качестве результата прислать скриншот экрана - дашборд должен содержать в названии имя приславшего.

Решение -

Выполняем команды по методичке -

# Устанавливаем вспомогательные пакеты и скачиваем Prometheus
$ yum update -y
$ yum install wget vim -y
$ wget https://github.com/prometheus/prometheus/releases/download/v2.44.0/prometheus-2.44.0.linux-amd64.tar.gz
# Создаем пользователя и нужные каталоги, настраиваем для них владельцев
$ useradd --no-create-home --shell /bin/false prometheus
$ mkdir /etc/prometheus
$ mkdir /var/lib/prometheus
$ chown prometheus:prometheus /etc/prometheus
$ chown prometheus:prometheus /var/lib/prometheus
# Распаковываем архив, для удобства переименовываем директорию и копируем бинарники в /usr/local/bin
$ tar -xvzf prometheus-2.44.0.linux-amd64.tar.gz
$ mv prometheus-2.44.0.linux-amd64 prometheuspackage
$ cp prometheuspackage/prometheus /usr/local/bin/
$ cp prometheuspackage/promtool /usr/local/bin/
# Меняем владельцев у бинарников
$ chown prometheus:prometheus /usr/local/bin/prometheus
$ chown prometheus:prometheus /usr/local/bin/promtool
# По аналогии копируем библиотеки
/*
Первоначальный вариант prometheuspackage/prometheus.yml -
cat /etc/prometheus/prometheus.yml
cat prometheuspackage/prometheus.yml
 my global config
global:
  scrape_interval: 15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).
                                                                                                                                                                             
# Alertmanager configuration                                                                                                                                                 
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          # - alertmanager:9093
                                                                                                                                                                             
# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.                                                                              
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"
                                                                                                                                                                             
# A scrape configuration containing exactly one endpoint to scrape:                                                                                                          
# Here it's Prometheus itself.                                                                                                                                               
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: "prometheus"
                                                                                                                                                                             
    # metrics_path defaults to '/metrics'                                                                                                                                    
    # scheme defaults to 'http'.                                                                                                                                             
                                                                                                                                                                             
    static_configs:                                                                                                                                                          
      - targets: ["localhost:9090"]
*/
$ cp -r prometheuspackage/consoles /etc/prometheus
$ cp -r prometheuspackage/console_libraries /etc/prometheus
$ chown -R prometheus:prometheus /etc/prometheus/consoles
$ chown -R prometheus:prometheus /etc/prometheus/console_libraries
# Создаем файл конфигурации
#$ vim /etc/prometheus/prometheus.yml
cat /etc/prometheus/prometheus.yml - пока такого файла нет, создадим.
cat > /etc/prometheus/prometheus.yml
global:
  scrape_interval: 10s
scrape_configs:
  - job_name: 'prometheus_master'
    scrape_interval: 5s
    static_configs:
      - targets: ['localhost:9090']
Выполняем "Ctrl+d"
$ chown prometheus:prometheus /etc/prometheus/prometheus.yml
# Настраиваем сервис
#$ vim /etc/systemd/system/prometheus.service
cat /etc/systemd/system/prometheus.service
cat > /etc/systemd/system/prometheus.service
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target
[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
--config.file /etc/prometheus/prometheus.yml \
--storage.tsdb.path /var/lib/prometheus/ \
--web.console.templates=/etc/prometheus/consoles \
--web.console.libraries=/etc/prometheus/console_libraries
[Install]WantedBy=multi-user.target
$ systemctl daemon-reload
$ systemctl start prometheus
$ systemctl status prometheus
#http://192.168.56.10:9090/graph
http://localhost:9090/graph
# Скачиваем и распаковываем Node Exporter
$ wget https://github.com/prometheus/node_exporter/releases/download/v1.5.0/node_exporter-1.5.0.linux-amd64.tar.gz
$ tar xzfv node_exporter-1.5.0.linux-amd64.tar.gz
# Создаем пользователя, перемещаем бинарник в /usr/local/bin
$ useradd -rs /bin/false nodeusr
$ mv node_exporter-1.5.0.linux-amd64/node_exporter /usr/local/bin/
# Создаем сервис
#$ vim /etc/systemd/system/node_exporter.service
cat > /etc/systemd/system/node_exporter.service
[Unit]
Description=Node Exporter
After=network.target
[Service]
User=nodeusr
Group=nodeusr
Type=simple
ExecStart=/usr/local/bin/node_exporter
[Install]
WantedBy=multi-user.target
# Запускаем сервис
$ systemctl daemon-reload
$ systemctl start node_exporter
$ systemctl enable node_exporter
systemctl status node_exporter
systemctl status prometheus

# Обновляем конфигурацию Prometheus
#$ vim /etc/prometheus/prometheus.yml
cat > /etc/prometheus/prometheus.yml
cat /etc/prometheus/prometheus.yml
global:
  scrape_interval: 10s
  scrape_configs:
    - job_name: 'prometheus_master'
      scrape_interval: 5s
      static_configs:
        - targets: ['localhost:9090']
    - job_name: 'node_exporter_centos'
      scrape_interval: 5s
      static_configs:
        - targets: ['localhost:9100']
# Перезапускаем сервис
$ systemctl restart prometheus
systemctl status prometheus

#http://192.168.56.10:9090/targets
#http://192.168.56.10:9100/metrics
http://localhost:9090/targets
http://localhost:9100/metrics

# Скачать grafana-enterprise-9.5.2-1.x86_64.rpm через VPN и поместить на сервер
$ cd /vagrant
$ yum -y install grafana_enterprise_9.5.2_1.x86_64-364648-163733.rpm
# Стартуем сервис
$ systemctl daemon-reload
$ systemctl start grafana-server
systemctl status grafana-server

http://192.168.56.10:3000 (admin/admin)
http://localhost:3000

Ставим URL http://192.168.56.10:9090 - Работает !!!
Выбираем максимально понравившийся Dashboar - ID: 1860
# Скачиваем и распаковываем AlertManager
$ wget https://github.com/prometheus/alertmanager/releases/download/v0.25.0/alertmanager-0.25.0.linux-amd64.tar.gz
$ tar zxf alertmanager-0.25.0.linux-amd64.tar.gz
# Создаем пользователя и нужные директории
$ useradd --no-create-home --shell /bin/false alertmanager
$ usermod --home /var/lib/alertmanager alertmanager
$ mkdir /etc/alertmanager
$ mkdir /var/lib/alertmanager
# Копируем бинарники из архива в /usr/local/bin и меняем владельца
$ cp alertmanager-0.25.0.linux-amd64/amtool /usr/local/bin/
$ cp alertmanager-0.25.0.linux-amd64/alertmanager /usr/local/bin/
$ cp alertmanager-0.25.0.linux-amd64/alertmanager.yml /etc/alertmanager/
$ chown -R alertmanager:alertmanager /etc/alertmanager /var/lib/alertmanager
$ chown alertmanager:alertmanager /usr/local/bin/{alertmanager,amtool}
$ echo "ALERTMANAGER_OPTS=\"\"" > /etc/default/alertmanager
$ chown alertmanager:alertmanager /etc/default/alertmanager
mkdir /var/lib/prometheus/alertmanager
$ chown -R alertmanager:alertmanager /var/lib/prometheus/alertmanager
# Настраиваем сервис
#$ vim /etc/systemd/system/alertmanager.service
cat > /etc/systemd/system/alertmanager.service
[Unit]
Description=Alertmanager Service
After=network.target prometheus.service
[Service]
EnvironmentFile=-/etc/default/alertmanager
User=alertmanager
Group=alertmanager
Type=simple
ExecStart=/usr/local/bin/alertmanager \
          --config.file=/etc/alertmanager/alertmanager.yml \
          --storage.path=/var/lib/prometheus/alertmanager \
          $ALERTMANAGER_OPTS
          ExecReload=/bin/kill -HUP $MAINPID
          Restart=on-failure
          Restart=always
          [Install]
          WantedBy=multi-user.target
# Запускаем сервис
$ systemctl daemon-reload
$ systemctl start alertmanager
# Настраиваем правила
#$ vim /etc/prometheus/rules.yml
cat /etc/prometheus/rules.yml
cat > /etc/prometheus/rules.yml
groups:
- name: alert.rules
  rules:
  - alert: InstanceDown
    expr: up == 0
    for: 1m
    labels:
      severity: critical
      annotations:
      description: '{{ $labels.instance }} of job {{ $labels.job }} has been down for more than 1 minute.'
        summary: Instance {{ $labels.instance }} down
# Проверяем валидность
$ /usr/local/bin/promtool check rules /etc/prometheus/rules.yml
# Обновляем конфиг prometheus
#$ vim /etc/prometheus/prometheus.yml
cat > /etc/prometheus/prometheus.yml
global:
  scrape_interval: 10s
scrape_configs:
  - job_name: 'prometheus_master'
    scrape_interval: 5s
    static_configs:
      - targets: ['localhost:9090']
  - job_name: 'node_exporter_centos'
    scrape_interval: 5s
    static_configs:
      - targets: ['localhost:9100']
rule_files:
 - "rules.yml"
alerting:
  alertmanagers:
    - static_configs:
      - targets:
        - localhost:9093
# Рестартуем сервисы
$ systemctl restart prometheus
$ systemctl restart alertmanager

#http://192.168.56.10:9093
#http://192.168.56.10:9090
http://localhost:9093
http://localhost:9090
Проверяем работу Dashboard - Результат в файле - "Снимок экрана от 2023-07-27 07-47-38.jpg"
Изменил несколько параметров в панелях и названии.
Все.
