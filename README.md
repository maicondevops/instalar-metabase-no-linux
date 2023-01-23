# instalar-metabase-no-linux

------------------------------------------------------------------------------------
--- 00 - MEU AMBIENTE E OBSERVAÇÕES
------------------------------------------------------------------------------------

* 1 VM com Ubuntu Server 20.04. </br>
* Uma partição secundária de Dados de 1 TB (/Dados). </br>
* Metabase 0.36.7. </br>
* Testado usando Vagrant e no Azure Virtual Machines. </br>

------------------------------------------------------------------------------------
--- 01 - CRIAR DIRETÓRIOS DADOS NO DISCO SECUNDÁRIO
------------------------------------------------------------------------------------

```
cd /Dados
sudo mkdir Metabase
cd Metabase
sudo mkdir log 
``` 
</br>

------------------------------------------------------------------------------------
--- 02 - AJUSTAR HOSTNAME E FILE HOSTS
------------------------------------------------------------------------------------
-- AJUSTAR HOSTNAME: </br>

```
sudo echo "metabase01.mydomain.com" > /etc/hostname
```  

-- AJUSTAR HOSTS: </br>

```
sudo echo "192.168.50.20 metabase01.mydomain.com" >> /etc/hosts
```

OBSERVAÇÕES: </br>
1 - SE O ACESSO AO METABASE FOR EXTERNO, VOCÊ PRECISA CRIAR OS REGISTROS NO SEU DNS EXTERNO. </br>

------------------------------------------------------------------------------------
--- 03 - INSTALAR O JAVA
------------------------------------------------------------------------------------

```
sudo apt update -y && sudo apt install openjdk-11-jdk openjdk-11-jre -y
```

------------------------------------------------------------------------------------
--- 04 - INSTALAR O MARIA DB
------------------------------------------------------------------------------------

```
sudo apt install mariadb-server && sudo systemctl start mariadb && sudo systemctl enable mariadb && sudo systemctl status mariadb
```


------------------------------------------------------------------------------------
--- 05 - CRIAR CREDENCIAL DE ROOT NO MARIA DB
------------------------------------------------------------------------------------

```
sudo mysql_secure_installation
```

||
|---|
|<img src="https://github.com/maicondevops/instalar-metabase-no-linux/blob/59cf29c9f29acadacd394e5103eb3c38f9184a2a/img/passo05.1.png" width="700" height="200"/>|

------------------------------------------------------------------------------------
--- 06 - CRIAR UM DATABASE PARA O METABASE NO MARIA DB
------------------------------------------------------------------------------------
--- LOGON MYSQL: </br>

```
sudo mysql -u root -p
```

--- CRIAR DATABASE: </br>

```
CREATE DATABASE metabase;
```

--- CRIAR USER METABASE NO MARIA DB: </br>

```
CREATE USER 'admin'@'localhost' IDENTIFIED BY '321321';
GRANT all privileges on *.* to 'admin'@'localhost' identified by '321321';
GRANT ALL ON *.* TO 'admin'@'localhost' WITH GRANT OPTION;
FLUSH PRIVILEGES;
EXIT;
```

Observação: Eu liberei acesso a todas as bases. Mas você DEVE restringir o acesso apenas a base do metabase.
</br>

------------------------------------------------------------------------------------
--- 07 - DOWNLOAD DO METABASE
------------------------------------------------------------------------------------

```
cd /Dados/Metabase/ && wget https://downloads.metabase.com/v0.36.7/metabase.jar
```

------------------------------------------------------------------------------------
--- 08 - AJUSTAR PERMISSÕES USER METABASE
------------------------------------------------------------------------------------

```
sudo groupadd -r metabase &&
sudo useradd -r -s /bin/false -g metabase metabase &&
sudo chown -R metabase:metabase /Dados/Metabase &&
sudo touch /Dados/Metabase/log/metabase.log && sudo chown metabase:metabase /Dados/Metabase/log/metabase.log
```

------------------------------------------------------------------------------------
--- 09 - CRIAR O ARQUIVO DE SERVIÇO DO METABASE
------------------------------------------------------------------------------------

```
sudo nano /etc/systemd/system/metabase.service
```

--- COLAR O SCRIPT ABAIXO E SALVAR O ARQUIVO: </br>

```
[Unit]
Description=Metabase server
After=syslog.target
After=network.target

[Service]
WorkingDirectory=/Dados/Metabase/
ExecStart=/usr/bin/java -jar /Dados/Metabase/metabase.jar
EnvironmentFile=/Dados/Metabase/default/metabase
User=metabase
Type=simple
StandardOutput=syslog
StandardError=syslog
SyslogIdentifier=metabase
SuccessExitStatus=143
TimeoutStopSec=120
Restart=always

[Install]
WantedBy=multi-user.target
```

------------------------------------------------------------------------------------
--- 10 - CRIAR SYSLOG DO METABASE
------------------------------------------------------------------------------------

```
sudo nano /etc/rsyslog.d/metabase.conf
```

--- COLAR O SCRIPT ABAIXO E SALVAR O ARQUIVO: </br>

```
if $programname == 'metabase' then /Dados/Metabase/log/metabase.log
& stop
```

```
sudo systemctl restart rsyslog.service
```

------------------------------------------------------------------------------------
--- 11 - CRIAR O ARQUIVO DE CONFIGURAÇÃO DO METABASE
------------------------------------------------------------------------------------

```
sudo nano /etc/default/metabase
```

--- COLAR O SCRIPT ABAIXO E SALVAR O ARQUIVO: </br>

```
MB_PASSWORD_COMPLEXITY=strong
MB_PASSWORD_LENGTH=10
MB_DB_TYPE=mysql
MB_DB_DBNAME=metabase
MB_DB_PORT=3306
MB_DB_USER=admin
MB_DB_PASS=321321
MB_DB_HOST=localhost
MB_EMOJI_IN_LOGS=true
```

```
sudo systemctl daemon-reload && sudo systemctl start metabase && sudo systemctl enable metabase && sudo systemctl status metabase
```


------------------------------------------------------------------------------------
--- 12 - INSTALAR O NGINX
------------------------------------------------------------------------------------

```
sudo apt install nginx -y
```

--- CRIAR FILE NGINX: </br>

```
sudo nano /etc/nginx/sites-available/metabase
```

--- COLAR O SCRIPT ABAIXO E SALVAR O ARQUIVO: </br>

```
server {
     listen [::]:80;
     listen 80;

     server_name metabase01.mydomain.com;

     }
}

  access_log /Dados/Metabase/log/access.log;
```

```
sudo ln -s /etc/nginx/sites-available/metabase /etc/nginx/sites-enabled/metabase
```

--- VERIFICAR STATUS NGINX:

```
sudo systemctl restart nginx && sudo nginx -t
```

------------------------------------------------------------------------------------
--- 13 - SETUP DO METABASE
------------------------------------------------------------------------------------

--- A PORTA PADRÃO É A 3000: </br>

```
http://metabase01.mydomain.com:3000/setup
```

--- PREENCHA O CADASTRO E PRONTO: </br>

||
|---|
|<img src="https://github.com/maicondevops/instalar-metabase-no-linux/blob/59cf29c9f29acadacd394e5103eb3c38f9184a2a/img/passo13.1.png" width="700" height="400"/>|

--- PRONTO !

||
|---|
|<img src="https://github.com/maicondevops/instalar-metabase-no-linux/blob/59cf29c9f29acadacd394e5103eb3c38f9184a2a/img/passo13.4.png" width="700" height="400"/>|

||
|---|
|<img src="https://github.com/maicondevops/instalar-metabase-no-linux/blob/59cf29c9f29acadacd394e5103eb3c38f9184a2a/img/passo13.5.png" width="800" height="500"/>|

------------------------------------------------------------------------------------
--- 14 - ALTERAR PORTA PADRÃO DO METABASE
------------------------------------------------------------------------------------

--- EDITAR O FILE NGINX: </br>

```
sudo nano /etc/nginx/sites-available/metabase
```

--- COLAR O SCRIPT ABAIXO E SALVAR O ARQUIVO: </br>

```
server {
     listen [::]:80;
     listen 80;

     server_name metabase01.mydomain.com;

     location / {
         proxy_set_header X-Forwarded-Host $host;
         proxy_set_header X-Forwarded-Server $host;
         proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
         proxy_pass http://localhost:3000;
         client_max_body_size 100M;
     }
}

  access_log /Dados/Metabase/log/access.log;
```

```
sudo ln -s /etc/nginx/sites-available/metabase /etc/nginx/sites-enabled/metabase
```

--- E ALTERE A URL NO PAINEL DE ADMINISTRAÇÃO: </br>

||
|---|
|<img src="https://github.com/maicondevops/instalar-metabase-no-linux/blob/59cf29c9f29acadacd394e5103eb3c38f9184a2a/img/passo14.1.png" width="700" height="200"/>|
