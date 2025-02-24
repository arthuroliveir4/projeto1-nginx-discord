# PROJETO SERVIDOR WEB NGINX + AWS

## PRINCIPAIS TECNOLOGIAS UTILIZADAS NO PROJETO
- Amazon Web Services (AWS), para criação da instância EC2, Amazon Linux 2023 com t2.micro (com VPC e Security Groups personalizados), onde foi feita a configuração de um ambiente que servirá como máquina virtual para a configuração do servidor WEB e criação dos scripts.

- Software PuTTy, para realizar a conexão na instância, utilizando uma chave SSH .ppk e o IPv4 público da EC2.

- NGINX, instalado dentro do ambiente Linux da EC2, com a função de servir a página WEB.

- Python para realizar o script de monitoramento da página web e enviar notificações quando fora do ar.

- Crond (crontab), ferramenta do Linux, utilizada para programar a execução do script, verificando a situação da página web a cada minuto.

- Discord, para fazer a integração com o script e receber as notificações via chat (por meio de um bot webhook).
## CONFIGURAÇÃO DO AMBIENTE AWS
- O primeiro passo é a criação de uma VPC com 2 subnets públicas e duas privadas, utilizando a criação automática da AWS, o Internet Gateway e as Route Tables já vêm configuradas.

![Image](https://github.com/user-attachments/assets/6102c450-475d-4825-8d60-4cb78b1bfa21)


- Após isso, criei um Security Group vinculado à VPC e liberei regras de entrada e saída para a porta 80 (HTTP) e 22 (SSH). A regra de saída já vem liberando para All, mas para as regras de entrada precisa adicionar as portas 80 e 22.

![Image](https://github.com/user-attachments/assets/250ed674-1af9-43c4-9db6-afe319eb4640)

- Agora é hora de criar a instância EC2, utilizei Amazon Linux 2023 e t2.micro.

![Image](https://github.com/user-attachments/assets/e1fbf100-b820-46a2-b67a-6d8718f11fc6)

- Selecionei a VPC e o Security Group criados previamente.
- Selecionei uma das subnets públicas da VPC.
- Ainda nas configurações de rede da EC2, habilitei a atribuição automática de um IPv4 público à instância, para ser possível utilizar esse IP como endereço da página WEB.
- Criei uma chave SSH (.ppk) para conectar via PuTTy.
- Para conectar-se à instância com essa chave .ppk é necessária a instalação do programa PuTTy na sua máquina.
- Nesse programa, você atribuirá a chave SSH e colocará o IPv4 público da instância para conectar-se nela.

![Image](https://github.com/user-attachments/assets/1048957a-726d-4fc1-8bba-a871669b8cfb)

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
sudo systemctl restart nginx
sudo systemctl status nginx
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
    <title>Projeto com Nginx, AWS e Discord</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            background-color: #f4f4f4;
            margin: 0;
            padding: 0;
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
        }
        .container {
            background: white;
            padding: 20px;
            border-radius: 10px;
            box-shadow: 0px 4px 6px rgba(0, 0, 0, 0.1);
            text-align: center;
            max-width: 500px;
        }
        h1 {
            color: #2c3e50;
        }
        p {
            color: #555;
            line-height: 1.6;
        }
        .button {
            display: inline-block;
            margin-top: 15px;
            padding: 10px 20px;
            background-color: #3498db;
            color: white;
            text-decoration: none;
            border-radius: 5px;
            font-weight: bold;
            transition: 0.3s;
        }
        .button:hover {
            background-color: #2980b9;
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>Olá, EC2 com Nginx!</h1>
        <p>Este projeto demonstra a configuração de um servidor web usando <strong>Nginx</strong> hospedado em uma instância <strong>AWS EC2</strong>.</p>
        <p>Além disso, a aplicação está integrada com um bot no <strong>Discord</strong>, permitindo comunicação em tempo real e automações.</p>
        <p>O objetivo deste projeto é explorar a infraestrutura em nuvem e servidores web de forma prática.</p>
        <a href="#" class="button">Saiba mais</a>
    </div>
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
- PÁGINA WEB EM FUNCIONAMENTO (Acessada pelo http://IPv4_Público_da_Ec2)
![Image](https://github.com/user-attachments/assets/42970bdd-4fee-41c6-a623-73a3f328a6e3)

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
- Script Python no Linux
![Image](https://github.com/user-attachments/assets/7c2c4e9c-2311-4a07-9546-13e6024e986d)

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
- Script Crond no Linux
![Image](https://github.com/user-attachments/assets/52805ab9-c179-40da-9ffe-2cdd04fed39d)


## CRIAÇÃO DO SERVIDOR NO DISCORD E CANAL DE TEXTO CONFIGURADO COM WEBHOOK:

- Criar um servidor no DISCORD e criar um canal de texto
- Ir nas configurações desse canal e clicar em INTEGRAÇÕES(WEBHOOKS)
- Criar um novo Webhook e copiar a URL dele para colocar no monitoramento.py no início do script em
webhookdiscord_url =

- Notificações no Discord quando o NGINX para e a página WEB para de responder
![Image](https://github.com/user-attachments/assets/e0b18a21-5032-47c9-8c76-2b71a909717e)

## CONCLUSÃO DO PROJETO
- A instância EC2 quando iniciada e em funcionamento na AWS, fornece a conexão à máquina Amazon Linux, nessa máquina estão configurados o NGINX, o script em Python e o crontab. Além do Discord com o bot webhook para receber as notificações.
- O NGINX providencia o funcionamento da página WEB, o script em Python realiza a verificação do funcionamento da página, o crontab realiza a execução do script a cada minuto e o link do webhook do Discord dentro do script permite o envio de notificações diretamente para o chat do Discord.
- Portanto, com a instância EC2 em estado "running" e tendo a conexão estabelecida via PuTTy, o servidor WEB irá funcionar, o script será executado e em caso de falha na obtenção de respostas da página, serão enviadas notificações a cada 1 minuto para o Discord.

- Teste do NGINX em funcionamento e após ser parado
![Image](https://github.com/user-attachments/assets/e359dc4e-a1e9-4389-b47c-e6309fe7dc35)

- Teste da página WEB sem resposta após parar o NGINX
![Image](https://github.com/user-attachments/assets/672a501d-21e9-4203-8d02-53e480ecf31b)
__Detalhe que ao realizar esse teste de parar o NGINX, as notificações chegam no Discord, avisando que o site está sem resposta (como mostrado no print apresentado no tópico do Discord)__

- Arquivo monitoramento.log no diretório /var/log antes e após a parada do NGINX

  ![Image](https://github.com/user-attachments/assets/7ccbe34c-e818-40b5-8599-1b3b24e7dbeb)
