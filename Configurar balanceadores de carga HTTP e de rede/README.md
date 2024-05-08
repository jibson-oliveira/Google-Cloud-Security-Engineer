<h4>Tarefa 1: definir a região e a zona padrão para todos os recursos</h4>

1. Defina a região padrão:

`gcloud config set compute/region Region`

2. Defina a zona padrão no Cloud Shell:

`gcloud config set compute/zone Zone`

<h4>Tarefa 2: criar várias instâncias do servidor da Web</h4>

1. Crie uma máquina virtual www1 na zona padrão usando o código a seguir:

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


2. Crie uma máquina virtual www2 na zona padrão usando o código a seguir:
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

3. Crie uma máquina virtual www3 na zona padrão:
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

4. Crie uma regra de firewall para permitir o tráfego externo para as instâncias de VM:
```gcloud compute firewall-rules create www-firewall-network-lb \
    --target-tags network-lb-tag --allow tcp:80
```
    

5. Execute o comando a seguir para listá-las. Os endereços IP aparecem na coluna EXTERNAL_IP:

`gcloud compute instances list`

6. Verifique se cada instância está sendo executada com curl e substitua [IP_ADDRESS] pelo endereço IP de cada uma das VMs:

`curl http://[IP_ADDRESS]`


<h4>Tarefa 3: configurar o serviço de balanceamento de carga</h4>

1. Crie um endereço IP externo estático para o balanceador de carga.
`gcloud compute addresses create network-lb-ip-1 --region Region`

2. Adicione um recurso legado de verificação de integridade HTTP.

`gcloud compute http-health-checks create basic-check`

3. Adicione um pool de destino na mesma região de suas instâncias. Execute o comando a seguir para criar o pool de destino e usar a verificação de integridade necessária para o funcionamento do serviço:
```gcloud compute target-pools create www-pool \
  --region Region --http-health-check basic-check
```


4. Adicione as instâncias ao pool:
```gcloud compute target-pools add-instances www-pool \
    --instances www1,www2,www3
```

5. Adicione uma regra de encaminhamento:
``` gcloud compute forwarding-rules create www-rule \
    --region  Region \
    --ports 80 \
    --address network-lb-ip-1 \
    --target-pool www-pool
```

