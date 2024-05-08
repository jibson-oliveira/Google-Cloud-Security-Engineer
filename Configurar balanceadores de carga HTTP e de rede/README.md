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

<h4>Tarefa 4: como enviar tráfego às instâncias</h4>

1. Execute o comando a seguir para visualizar o endereço IP externo da regra de encaminhamento www-rule usada pelo balanceador de carga:
`gcloud compute forwarding-rules describe www-rule --region Region `

2. Acesse o endereço IP externo:

`IPADDRESS=$(gcloud compute forwarding-rules describe www-rule --region Region --format="json" | jq -r .IPAddress)`

3. Mostre o endereço IP externo:
`echo $IPADDRESS`

4. Use o comando curl para acessar o endereço IP externo e substitua IP_ADDRESS por um endereço IP externo do comando anterior:
`while true; do curl -m1 $IPADDRESS; done`

A resposta do comando curl alterna de forma aleatória entre as três instâncias. Se ocorrer uma falha, aguarde cerca de 30 segundos até que a configuração esteja totalmente carregada e suas instâncias marcadas como íntegras antes de tentar novamente.

5. Use Ctrl + C para interromper a execução do comando.

<h4>Tarefa 5: criar um balanceador de carga HTTP</h4>

1. Crie primeiro o modelo do balanceador de carga:
```gcloud compute instance-templates create lb-backend-template \
   --region=Region \
   --network=default \
   --subnet=default \
   --tags=allow-health-check \
   --machine-type=e2-medium \
   --image-family=debian-11 \
   --image-project=debian-cloud \
   --metadata=startup-script='#!/bin/bash
     apt-get update
     apt-get install apache2 -y
     a2ensite default-ssl
     a2enmod ssl
     vm_hostname="$(curl -H "Metadata-Flavor:Google" \
     http://169.254.169.254/computeMetadata/v1/instance/name)"
     echo "Page served from: $vm_hostname" | \
     tee /var/www/html/index.html
     systemctl restart apache2'
```

2. Crie o grupo gerenciado de instâncias com base no modelo:
```gcloud compute instance-groups managed create lb-backend-group \
   --template=lb-backend-template --size=2 --zone=Zone
```
3. Crie a regra de firewall fw-allow-health-check.
```gcloud compute firewall-rules create fw-allow-health-check \
  --network=default \
  --action=allow \
  --direction=ingress \
  --source-ranges=130.211.0.0/22,35.191.0.0/16 \
  --target-tags=allow-health-check \
  --rules=tcp:80
```

4. Agora que as instâncias estão funcionando, configure um endereço IP externo, estático e global que seus clientes podem usar para acessar o balanceador de carga:
```gcloud compute addresses create lb-ipv4-1 \
  --ip-version=IPV4 \
  --global
```

**Anote o endereço IPv4 que foi reservado:**

```
gcloud compute addresses describe lb-ipv4-1 \
  --format="get(address)" \
  --global
```

5. Crie uma verificação de integridade para o balanceador de carga:
```
gcloud compute health-checks create http http-basic-check \
  --port 80
```

6. Crie um serviço de back-end:
```
gcloud compute backend-services create web-backend-service \
  --protocol=HTTP \
  --port-name=http \
  --health-checks=http-basic-check \
  --global
```

7. Adicione seu grupo de instâncias como back-end do serviço de back-end:

```
gcloud compute backend-services add-backend web-backend-service \
  --instance-group=lb-backend-group \
  --instance-group-zone=Zone \
  --global
```

8. Crie um mapa de URLs para encaminhar as solicitações de entrada ao serviço de back-end padrão:

  ```
    gcloud compute url-maps create web-map-http \
    --default-service web-backend-service
  ```

> **Observação:** o mapa de URL é um recurso de configuração do Google Cloud usado para rotear solicitações para serviços de back-end ou buckets de back-end. Por exemplo, com um balanceador de carga HTTP(S) externo, é possível usar um único mapa de URL para rotear solicitações para destinos diferentes com base nas regras configuradas no mapa de URL:
>
> Solicitações para https://example.com/video vão para um serviço de back-end.
> Solicitações para https://example.com/audio vão para um serviço de back-end diferente.
> Solicitações para https://example.com/images vão para um bucket de back-end do Cloud Storage.
> As solicitações de qualquer outra combinação de host e caminho vão para um serviço de back-end padrão.

9. Crie um proxy HTTP de destino para encaminhar as solicitações ao mapa de URLs:
```
gcloud compute target-http-proxies create http-lb-proxy \
    --url-map web-map-http
```

10. Crie uma regra de encaminhamento global para encaminhar as solicitações recebidas para o proxy:
```
gcloud compute forwarding-rules create http-content-rule \
   --address=lb-ipv4-1\
   --global \
   --target-http-proxy=http-lb-proxy \
   --ports=80
```

> **Observação:** A regra de encaminhamento e o endereço IP correspondente dela representam a configuração de front-end de um balanceador de carga do Google Cloud. Saiba mais sobre as noções básicas com o Guia de informações gerais das regras de encaminhamento. 

