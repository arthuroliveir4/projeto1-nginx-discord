# PROJETO SERVIDOR WEB NGINX + AWS

## CONFIGURAÇÃO DO AMBIENTE AWS
- O primeiro passo é a criação de uma VPC com 2 subnets públicas e duas privadas, utilizando a criação automática da AWS, o Internet Gateway e as Route Tables já vêm configuradas.

- Após isso, criei um Security Group vinculado à VPC e liberei regras de entrada e saída para a porta 80 (HTTP) e 22 (SSH). A regra de saída já vem liberando para All, mas para as regras de entrada precisa adicionar as portas 80 e 22.

- Agora é hora de criar a instância EC2, utilizei Amazon Linux e t2.micro.
- Selecionei a VPC e o Security Group criados previamente.
- Selecionei uma das subnets públicas da VPC.
- Ainda nas configurações de rede da EC2, habilitei a atribuição automática de um IPv4 público à instância, para ser possível utilizar esse IP como endereço da página WEB.
- Criei uma chave SSH (.ppk) para conectar via PuTTy.
- Para conectar-se à instância com essa chave .ppk é necessária a instalação do programa PuTTy na sua máquina.
- Nesse programa, você atribuirá a chave SSH e colocará o IPv4 público da instância para conectar-se nela.
- Após realizada a conexão, você terá acesso ao terminal da sua EC2 Amazon Linux.
- No terminal, deve-se conectar com o usuário ec2-user (padrão para Amazon Linux).
- Depois de logado, é a hora de iniciar a configuração do ambiente Linux com NGINX para servir a página WEB.


## CONFIGURAÇÃO DO AMBIENTE AMAZON LINUX
- Instalar o NGINX e INICIÁ-LO
```bash 
sudo yum update -y
sudo amazon-linux-extras enable nginx1
sudo yum install -y nginx
sudo systemctl start nginx
sudo systemctl enable nginx
systemctl status nginx
```
Primeiro deve realizar os updates dos arquivos e pacotes do sistema, e seguir com a instalação do NGINX, os comandos podem variar de acordo com a sua distribuição caso estiver utilizando outra.

- Criar a página HTML que o NGINX irá servir
```bash
cd /usr/share/nginx/html
sudo nano index.html


<!DOCTYPE html>
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Página de Teste</title>
</head>
<body>
    <h1>Olá, EC2 com Nginx!</h1>
    <p>Esta é uma página simples servida pelo Nginx no Amazon Linux.</p>
</body>
</html>
```

É necessário acessar o sub-diretório HTML, que vem dentro do diretório do NGINX.
Nesse sub-diretório você irá criar seu index.html, que conterá o código html da página WEB.

- No diretório nginx.conf, adicionar a linha index index.html para o nginx servir a página
```bash
sudo nano /etc/nginx/nginx.conf

server {
    listen 80;
    server_name _;

    root /usr/share/nginx/html;
    index index.html; (ADICIONAR ESSA LINHA)
}

sudo nano /usr/lib/systemd/system/nginx.service

[Service]
Restart=always
RestartSec=5

'(ADICIONAR ESSES RESTARTS PARA O NGINX REINICAR CASO FECHE)'

sudo systemctl daemon-reload
sudo systemctl restart nginx

'PARA REINICIAR O SERVIÇO E APLICAR AS MUDANÇAS'
```

## INSTALAR OS RECURSOS DO PYTHON PARA FAZER O SCRIPT DE MONITORAMENTO

```bash
sudo yum install python3 -y
sudo yum install python3-pip
sudo pip3 install requests
sudo yum install python3-requests -y '(Se o de cima não funcionar)'
```
Esses comandos também podem variar de acordo com a distribuição utilizada.

- Criar o diretório onde será criado o script e o arquivo do script
```bash
mkdir -p /home/ec2-user/monitoramento
cd /home/ec2-user/monitoramento
nano monitoramento.py
```
- SCRIPT em PYTHON que realizará o monitoramento do servidor WEB servido pelo NGINX e enviará notificações no discord quando não obter respostas do servidor:
```bash
import requests
import time

URL = "http://54.160.46.98"

webhookdiscord_url = "https://discord.com/api/webhooks/1341039695844348004/4iM1jjMIdqcY7dNJ00F5xbolmt4sgvRmXcoEdDsPoB3OwKNHbah1VBO85l0LBhppwqM1"

def notificar_discord(mensagem):
        payload = {"content": mensagem}
        requests.post(webhookdiscord_url, json=payload)


def monitorar_site():
        try:
                resposta = requests.get(URL)
                codigo_status = resposta.status_code
                print(f"Status: {codigo_status}")

        except requests.exceptions.RequestException as erro:
                codigo_status = None
                print(f"Erro no acesso ao site: {erro}")

        timestamp = time.strftime("%d-%m-%Y %H:%M:%S")

        if codigo_status != 200:
                mensagem_log = f"{timestamp} - ALERTA: O site se encontra fora do ar!!"
                print(mensagem_log)
                with open("/var/log/monitoramento.log", "a") as arquivo_log:
                        arquivo_log.write(mensagem_log + "\n")
                notificar_discord(mensagem_log)
        else:
                mensagem_log = f"{timestamp} - OK: O site está funcionando de forma correta"
                print(mensagem_log)
                with open("/var/log/monitoramento.log", "a") as arquivo_log:
                        arquivo_log.write(mensagem_log + "\n")

if __name__ == "__main__":
        monitorar_site()
```
- **EXPLICAÇÃO DO CÓDIGO MONITORAMENTO.PY**
   
   Primeiramente, foram importadas as bibliotecas: *requests*, para fazer as requisições HTTP (verificar o site) e a *time* para poder adicionar data e hora no log de monitoramento.

   Foram definidas as variáveis: URL (endereço do site, no caso IP, que será monitorado) e webhookdiscord_url.

   A função **discord_notif(message)** recebe uma mensagem como parâmetro e envia essa mensagem para o canal do discord usando o webhook.

   A função **monitorar_site()**, faz uma requisição HTTP para o site com requests.get(URL) e verifica o status da resposta, se o status for 200 (código de resposta HTTP que indica que a requisição foi bem-sucedida), o código imprime que o site está em funcionamento e grava isso em /var/log/monitoramento.log.
   Caso o status não for 200, o código grava no LOG que o site está fora do ar e envia mensagem para o Discord.

   Além disso, tem o tratamento de erros, no try/except, no try ele verifica as possíveis chances de erro, se houver algum, ao invés de parar de rodar, ele vai para o except e o status_code recebe erro.

   A última parte do código (if __name__ == __main__), permite que o script possa ser executado tanto diretamente, quanto importado em outros programas.




**IMPORTANTE**: No início do código em URL, deve conter a URL com o IPv4 público da sua instância EC2.
E na URL do webhook, deve conter a url correspondente ao webhook do discord que criarei nos próximos passos.
Além disso, use o comando **sudo timedatectl set-timezone America/Sao_Paulo** para o horário das notifições aparecerem corretamente.

- Configuração e criação do script crond (crontab) para a automatização do script Python

```bash
sudo yum install cronie -y
sudo systemctl start crond
sudo systemctl enable crond
sudo systemctl status crond

crontab -e
'ADICIONE A SEGUINTE LINHA'
* * * * * /usr/bin/python3 /home/ec2-user/monitoramento/monitoramento.py

'ISSO FARÁ O monitoramento.py executar de 1 em 1 minuto'
'No script monitoramento.py, caso não haja resposta do servidor, uma mensagem será enviada via Discord a cada 1 minuto.'

chmod +x /home/ec2-user/monitoramento/monitoramento.py 
'PERMISSÃO PARA EXECUÇÃO DO SCRIPT'

'CASO QUEIRA TESTAR O SCRIPT MANUALMENTE'
python3 /home/ec2-user/monitoramento/monitoramento.py
```

## CRIAÇÃO DO SERVIDOR NO DISCORD E CANAL DE TEXTO CONFIGURADO COM WEBHOOK:

- Criar um servidor no DISCORD e criar um canal de texto
- Ir nas configurações desse canal e clicar em INTEGRAÇÕES(WEBHOOKS)
- Criar um novo Webhook e copiar a URL dele para colocar no monitoramento.py no início do script em
webhookdiscord_url =



## IMAGENS DO PROJETO

- Configurações da EC2

  https://imgur.com/a/mhQBciP

- Servidor WEB funcionando, servido pelo NGINX

  https://imgur.com/a/Y8ejBxD

- Script monitoramento.py

  https://imgur.com/a/ceOBx5M

- Script Crontab para automação do script python

  https://imgur.com/a/46vG4OU

- NGINX em funcionamento e após ser parado

  https://imgur.com/a/O1xSOlO

- Notificações via Discord quando o serviço NGINX foi parado

  https://imgur.com/a/lbSWXJ9

- Arquivo monitoramento.log no diretório /var/log antes e após a parada do NGINX

  https://imgur.com/a/yexNAad


