# Utilizando docker-elk sebp/elk, sistema linux

### Passo a passo de como instalar o docker no sistema linux & windows

[docker no sistema linux 20.4](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-ubuntu-20-04-pt)

[docker no sistema Windows 10](https://docs.docker.com/docker-for-windows/install/)

```bash
# Ambiente linux
sudo update
sudo upgrade
sudo apt install curl vim tree git wget
sudo apt install -qqy --no-install-recommends ca-certificates gosu tzdata openjdk-11-jdk-headless

# Criar o diretorio de producao
mkdir producao
cd producao
git clone git@github.com:neviim/_logstakelk_cris.git

cd _logstakelk_cris
cat /proc/sys/vm/max_map_count
sudo sysctl -w vm.max_map_count=262144

sudo docker pull sebp/elk

# A instalação esta pre configurada com:
user: elastic
password: changeme

# Por padrão, o ambiente expõe as seguintes portas:
5000: Logstash TCP input
9200: Elasticsearch HTTP
9300: Elasticsearch TCP transport
5601: Kibana
```


## Docker
```bash
# Container elk - sepb/elk
sudo docker run -p 5601:5601 -p 9200:9200 -p 5044:5044 -it \
-e ES_HEAP_SIZE="2g" -e LS_HEAP_SIZE="1g" --name elk sebp/elk

# OU

# Builder
docker-compose build elk

#  Up, Down, Log
docker-compose up elk
docker-compose up elk -d

docker-compose down elk
docker-compose logs elk

# Restartar serviços
docker-compose restart kibana logstash

# referencias de alguns comandos uteis
Contêineres:  docker container rm -f $(docker container ls -a -q)   
Imagens....:  docker image rm $(docker image ls -a -q)
Volumes....:  docker volume rm $(docker volume ls -q)     
Networks...:  docker network rm $(docker network ls -q)
```


# Valida e testa o acesso ao elasticsearch/kibana
```bash
apt-get update
apt-get install curl mlocate jq

# testa conectividade com o elasticsearch, retorna json de name: elk
curl --user elastic:changeme -X GET "http://192.168.0.44:9200"
curl --user elastic:changeme -X GET "http://192.168.0.44:9200/?pretty"
curl --user elastic:changeme -X GET "http://192.168.0.44:9200/_cat/indices?v"

# Habilite a licença de teste no servidor ElasticSearch.
curl --user elastic:changeme -X POST "http://192.168.0.44:9200/_license/start_trial?acknowledge=true&pretty"
curl --user elastic -X GET "http://192.168.0.44:9200/?pretty"

# Ao iniciar o contêiner ELK em um host, que atuará como o primeiro mestre.
curl http://192.168.0.44:9200/_cluster/health?pretty
{
  "cluster_name" : "elasticsearch",
  "status" : "yellow",
  "timed_out" : false,
  "number_of_nodes" : 1,
  "number_of_data_nodes" : 1,
  "active_primary_shards" : 6,
  "active_shards" : 6,
  ...
}
```


## Com logstash usando do csv-aba...conf
```bash
# ler um arquivo csv, colocando-o no elastic.
cd ~/producao/_logstakelk_cris/beats/logstash-7.9.1

./bin/logstash -f ../../src/confs/csv-aba_diario.conf
./bin/logstash -f ../../src/confs/csv-aba_expert.conf
```


# Slave host
```bash
# Em outro host, crie um arquivo denominado elasticsearch-slave.yml (digamos que esteja em ~/producao/elk), com o conteúdo
network.host: 0.0.0.0
network.publish_host: <reachable IP address or FQDN>
discovery.zen.ping.unicast.hosts: ["elk-master.example.com"]

# inicie o contêiner ELK que usa esse arquivo de configuração, usando o seguinte comando:
sudo docker run -it --rm=true -p 9200:9200 -p 9300:9300 \
-v /home/elk/elasticsearch-slave.yml:/etc/elasticsearch/elasticsearch.yml \
sebp/elk

# Depois que o Elasticsearch estiver ativo, a exibição da
# integridade do cluster no host original agora mostra 2 nodes:
curl http://192.168.0.44:9200/_cluster/health?pretty
{
  "cluster_name" : "elasticsearch",
  "status" : "green",
  "timed_out" : false,
  "number_of_nodes" : 2,
  "number_of_data_nodes" : 2,
  "active_primary_shards" : 6,
  "active_shards" : 12,
...
}
```


## Alguns atalhos do tmux
```bash
tmux ls
tmux list-windows -a
# não é possível mover a janela para uma janela já criada.
tmux move-window -s 0:1 -t 1:2   

tmux list-panes -a
# Para mover para um painel específico neste caso 2
tmux move-pane -s 0:1.1 -t 1:1.2

tmux list-panes -t 0:1

# Lista e abre session
tmux list-sessions
tmux attach-session -t <session_number>
tmux attach -t n
```


## Linux comando
```bash
# liberar porta 80 iptables
sudo iptables -A FORWARD -p tcp --dport 80   -j ACCEPT
sudo iptables -A FORWARD -p tcp --dport 5601 -j ACCEPT
sudo iptables -A FORWARD -p tcp --dport 9200 -j ACCEPT

sudo ufw allow 22
sudo ufw allow 5601
sudo ufw allow 9200
sudo ufw allow 9600
sudo ufw allow 5000

sudo ufw allow 'Apache'
sudo ufw allow 'Nginx Full'
# permite acesso a um rost TRUSTED_IP confiavel 
sudo ufw allow from TRUSTED_IP to any port 9200

sudo ufw delete allow 'Nginx HTTP'
sudo ufw status
# mostra seu ip local
hostname -I
# mostra seu ip publico
curl -4 icanhazip.com
# listar porta abertas
nmap -v localhost
# lista regras ufw
sudo ufw app list
# systemctl
sudo systemctl daemon-reload
```


## Coletar logs do apache com o filebeat
```bash
sudo filebeat modules enable apache system
sudo filebeat -e -c <path/filebeat.yml>
```


## Usando instalando filebeat
```bash
# em /etc/filebeat/filebeat.yml
sudo vim /etc/filebeat/filebeat.yml
  output.elasticsearch:
    hosts: ["192.168.0.44:9200"]
    username: "elastic"
    password: "changeme"

# instala dashboards (kibana)
systemctl stop filebeat.service
filebeat sudo filebeat setup

systemctl start filebeat.service
systemctl status filebeat.service

ps -aux | grep filebeat
```


## Config filebeat ==> enviando para ==> logstash
```bash
# workspace
cd /home/neviim/workspace/elastic

# config filebeat
cd /etc/filebeat
sudo vim /etc/filebeat/filebeat.yml
  # comenta a conecção com elasticsearch
    # output.elasticsearch:
    #   # Array of hosts to connect to.
    #   #hosts: ["localhost:9200"]
    #   hosts: ["192.168.0.44:9200"]

    #   # Protocol - either `http` (default) or `https`.
    #   #protocol: "https"

    #   # Authentication credentials - either API key or username/password.
    #   #api_key: "id:api_key"
    #   username: "elastic"
    #   password: "changeme"

  # descomenta a saida para logstash porta 5044
    output.logstash:
      # The Logstash hosts
      #hosts: ["192.168.0.44:5044"]
      hosts: ["localhost:5044"]

      # Optional SSL. By default is off.
      # List of root certificates for HTTPS server verifications
      #ssl.certificate_authorities: ["/etc/pki/root/ca.pem"]

      # Certificate for SSL client authentication
      #ssl.certificate: "/etc/pki/client/cert.pem"

      # Client Certificate Key
      #ssl.key: "/etc/pki/client/cert.key"

ps -aux | grep filebeat
sudo systemctl stop filebeat.service 
sudo filebeat modules disable system
sudo service filebeat restart
sudo systemctl status filebeat.service

# lista serviço filebeat
sudo filebeat modules list
```


## Usando modules Packetbeat
```bash
sudo ./packetbeat -e -c packetbeat.yml
```


## Usando modules Metricbeat
```bash
# https://www.elastic.co/pt/downloads/beats/metricbeat

# cria o diretorio log
./metricbeat 

# carrega dashboard no elasticsearch
./metricbeat setup -e

./metricbeat modules enable linux
```


## Usando modules Packetbeat
```bash
# Cria pastas logs e baixa os deshboards
cd packetbeat-7.9.1

chown -R root *
./packetbeat
./packetbeat setup -e 

./packetbeat -c packetbeat.yml -configtest
./packetbeat start

# lista os nomes dos devices de rede
./packetbeat devices

# Para linux configurar
vim packetbeat.yml
  packetbeat.interfaces.device: enp3s0f0
  packetbeat.interfaces.type: af_packet

# inicializa packetbeat com o gestor de app pm2
pm2 start "sudo ./packetbeat -e" --name packetbeat
pm2 start "./packetbeat -e"
pm2 list 

#
curl --user elastic:changeme -X GET 'http://192.168.0.44:9200/packetbeat-*/_search?pretty'
curl --user elastic:changeme -X GET 'http://192.168.0.44:9200/aba_expert/_search?pretty'
curl --user elastic:changeme -X GET 'http://192.168.0.44:9200/aba_diario/_search?pretty'
```


## Gerado no Elasticsearch para cada mapeamento cliente com o filebeat
```bash
sudo filebeat enroll http://192.168.0.44:5601 eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJjcmVhdGVkIjoiMjAyMC0wOC0zMVQyMjo0NjozOC43NTFaIiwiZXhwaXJlcyI6IjIwMjAtMDgtMzFUMjI6NTY6MzguNzUxWiIsInJhbmRvbUhhc2giOiLvv71P77-977-9YXTvv73vv73vv71o77-977-977-977-9XHLvv70277-9fVx1MDAxOO-_ve-_ve-_vVxuOCIsImlhdCI6MTU5ODkxMzk5OH0.tMC_FAW8sUoEXoDJUcoW4t7VzUvp01BZ3QF4wnJLHDA
```


## Kibana query - Manager (Dev Tools)
```bash
DELETE aba_diario
GET aba_diario/_count
GET aba_diario

{
  "query": {
    "multi_match" : {
      "query" : "guide",
      "fields" : ["title", "authors", "summary", "publish_date", "num_reviews", "publisher"]
    }
  }
}
```


## Config referencias
```bash
vim docker-compose.yml
elk:
  image: sebp/elk
  ports:
    - "5601:5601"
    - "9200:9200"
    - "5044:5044"
  
  environment:
    - ES_HEAP_SIZE=2g
    - LS_HEAP_SIZE=1g
```


## Ajustando a memoria
```bash
# verifica tamanho memoria virtual
cat /proc/sys/vm/max_map_count
65530

# Set a memoria para 262144
sudo sysctl -w vm.max_map_count=262144
config sysctl
vim /etc/sysctl.conf
```