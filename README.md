# PPT1-PB-AWS-IFMS-ESTACIO-DE-SA
Primeira atividade solicitada para o estágio de DevSecOps na compasso.uol <br>

SEGUE ABAIXO A DOCUMENTAÇÃO DO PASSO A PASSO REALIZADO PARA A ATIVIDADE SOLICITADA <br>

-----------------------------------------------------------------------------------

CONTEXT: o cenário proposto foi a criação de uma instancia com um servidor nfs e um apache, ademais, era necessário criar um script que alimentasse arquivos de log a cada 5 minutos com o status do servidor apache/httpd

-----------------------------------------------------------------------------------

INSTÂNCIA UTILIZADA <br>
foi utilizada uma máquina ec2 t3.small com 16Gb com o SO Amazon Linux 2 <br>
outras configs: <br>
a instancia estava associada a um elastic ip, além de ter sido configurado um VPC com um gateway para permitir a conexão da instacia <br>

-----------------------------------------------------------------------------------

CONFIGURAÇÕES DO NFS <br>
Bom, o SO selecionado já vem com o nfs-server adicionado por padrão então não foi preciso fazer a instalação do mesmo, mas caso fosse necessário, esse problema seria resolvido com um simples ```yum install nfs-utils```.  <br>
Fora isso, segue abaixo o passo a passo realizado: <br>
  1) CRIANDO O DIRETÓRIO <br>
    - Foi criado um diretório específico que seria compartilhado, para a criação de um diretório, basta executar o comando: <br>
      - ```sudo mkdir {path}/nome-diretorio``` <br>
        O `{path}` deve ser substituido pelo caminho até onde pretende-se criar a pasta compartilhada e claro, o nome do diretório pode ser escolhido respeitando-se as regras de nomenclatura <br>
      - No meu caso, o comando ficou ```sudo mkdir /roberto-junior``` <br>
  2) COMPARTILHANDO O DIRETÓRIO
    - Para compartilhar o diretório criado, basta editar o arquivo em `/etc` chamado `exports`, eu utilizei o nano para isso: <br>
      - ```sudo nano /etc/exports``` -> Esse arquivo contem os diretórios que serão compartilhados <br>
      - Adicione o seguinte ao arquivo: <br>
        ```{path}/nome-diretorio {first_client_ip}(options) {second_client_ip}(options)...``` <br>
          (claro, substitua o path e os ips) <br>
        - No meu caso <br>
        ```/roberto-junior ip_privado_da_minha_intancia(rw,sync,no_subtree_check)``` <br>
          As opções entre os parênteses representam <br>
            rw: acesso de leitura e gravação ao computador cliente. <br>
            sync: faz com que o NFS a grave alterações no disco antes de responder. <br>
            no_subtree_check: impede a verificação de se o arquivo está de fato disponível na árvore exportada. <br>
  3) Ainda precisamos disponibilizar esse diretório para o cliente do NFS montar <br>
    - ```exportfs -rv``` <br>
      Esse comando usa as informaçõe do arquivo `/etc/exports` para exportar os diretórios/arquivos configurados. <br>
  4) Podemos agora restartar o nfs-server apenas para garantir, e após isso, em tese os diretórios estarão compartilhados <br>
    - ```systemctl restart nfs-server``` <br>
    - ```systemctl enable nfs-server``` <br>
-----------------------------------------------------------------------------------
CONFIGURAÇÕES DO APACHE
  1) Instalando o apache <br>
	  - ```yum update``` (só pra desencargo de consciência né...) <br>
      - Atualiza todos os pacotes e as suas dependências <br>
	  - ```yum install httpd``` <br>
		  - Instala os repositórios do servidor apache.
	2) Iniciando o apache <br>
    - ```systemctl start httpd``` <br>
		  - Inicia o servidor apache. <br>
    - ```systemctl enable httpd``` <br>
	    - Iniciar o servidor apache no boot do sistema.
-----------------------------------------------------------------------------------
CRIANDO O SCRIPT PARA VALIDAR O STATUS DO SERVIDOR HTTPD

```bash
  #!/bin/bash  // indica que o bash será usado para esse arquivo
  LOG_FILE="/roberto-junior/apache_server_status.log"  // arquivo em que o log de status online será salvo
  MSG="Servico Httpd - ONLINE - OK, servico httpd ativo"  // msg de status online
  ACTIVE_STATUS="active"  
  while true; do  // vai ficar executando pra sempre, até o script ser interrompido
      APACHE_STATUS=$(systemctl is-active httpd)  // requisita ao sistema o status do servico httpd
      if [ "$APACHE_STATUS" != "$ACTIVE_STATUS" ];then  // ser for um status diferente do status de ativo ele mudará a mensagem para uma mensagem de erro e tambem alterará o arquivo em que o log será salvo
          MSG="Servico Httpd - OFFLINE - Falha, servico Httpd offline"
          LOG_FILE="/roberto-junior/apache_server_status__errors.log"
      fi
      echo "$(date '+%Y-%m-%d %H:%M:%S') - $MSG" >> "$LOG_FILE"
      sleep 300  // tempo de repetição
  done
```
obs: esse script foi salvo em um arquivo `apache_status_monitor.sh`, eu tambem mudei as permissões do arquivo com o comando: <br>
  `chmod +x apache_status_monitor.sh`

-----------------------------------------------------------------------------------
AGENDANDO EXECUÇÃO DO SCRIPT <br>
utilizei um service para executar o script acima
  1) Crie um arquivo de serviço em /etc/systemd/system com uma extensão `.service`. <br>
    - no meu caso <br>
      `sudo nano /etc/systemd/system/monitor-apache.service` <br>
        isso irá abrir um arquivo de texto em branco
  2) Adicione as seguintes informações <br>
  
```unit
  [Unit]
  Description=Monitor Apache Service  // pode ser mudado
  
  [Service]
  ExecStart=/apache_status_monitor.sh  // especifique o caminho para o arquivo que contem o script
  
  [Install]
  WantedBy=default.target
```

  4) Recarregue o systemd para que ele reconheça o novo arquivo de serviço: <br>
    ```
      sudo systemctl daemon-reload
    ```
  5) Habilite o serviço para que ele seja iniciado automaticamente na inicialização: <br>
    ```
      sudo systemctl enable monitor-apache.service
    ```
  6) Inicie o serviço agora: <br>
    ```
      sudo systemctl start monitor-apache.service
    ```
  <br>Você também pode verificar o status do serviço: <br>
    ```
      sudo systemctl status monitor-apache.service
    ```
