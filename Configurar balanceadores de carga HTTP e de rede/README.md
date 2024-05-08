<h4>Tarefa 1: definir a região e a zona padrão para todos os recursos</h4>

Defina a região padrão:

`gcloud config set compute/region Region`

Defina a zona padrão no Cloud Shell:

`gcloud config set compute/zone Zone`

<h4>Tarefa 2: criar várias instâncias do servidor da Web</h4>

Crie uma máquina virtual www1 na zona padrão usando o código a seguir:

```  gcloud compute instances create www1 \
    --zone=Zone \
    --tags=network-lb-tag \
    --machine-type=e2-small \
    --image-family=debian-11 \
    --image-project=debian-cloud \
    --metadata=startup-script='#!/bin/bash
      apt-get update
      apt-get install apache2 -y
      service apache2 restart
      echo "
<h3>Web Server: www1</h3>" | tee /var/www/html/index.html'
```


Crie uma máquina virtual www2 na zona padrão usando o código a seguir:
```  gcloud compute instances create www2 \
    --zone=Zone \
    --tags=network-lb-tag \
    --machine-type=e2-small \
    --image-family=debian-11 \
    --image-project=debian-cloud \
    --metadata=startup-script='#!/bin/bash
      apt-get update
      apt-get install apache2 -y
      service apache2 restart
      echo "
<h3>Web Server: www2</h3>" | tee /var/www/html/index.html'
```

Crie uma máquina virtual www3 na zona padrão:
```  gcloud compute instances create www3 \
    --zone=Zone  \
    --tags=network-lb-tag \
    --machine-type=e2-small \
    --image-family=debian-11 \
    --image-project=debian-cloud \
    --metadata=startup-script='#!/bin/bash
      apt-get update
      apt-get install apache2 -y
      service apache2 restart
      echo "
<h3>Web Server: www3</h3>" | tee /var/www/html/index.html'
```

Crie uma regra de firewall para permitir o tráfego externo para as instâncias de VM:
```gcloud compute firewall-rules create www-firewall-network-lb \
    --target-tags network-lb-tag --allow tcp:80 ```


Execute o comando a seguir para listá-las. Os endereços IP aparecem na coluna EXTERNAL_IP:

gcloud compute instances list