---
title:  Aplicativos móveis sem criptografia
description: >-
    Quantos usuários verificam se seus dados são criptografados quando usam um
    aplicativo de celular? Criptografar dados em trânsito é essencial para manter
    a privacidade e integridade dos dados, mas os usuários geralmente não pensam
    neste tipo de detalhes. Vamos aprender um pouco mais sobre o assunto?
image_url: /assets/posts_assets/2016-01-11-aplicativos-sem-criptografia.md_assets/mob-crypt.png
date: 2016-01-11 23:00:00 -0300
layout: blogpage
lang: pt_BR
tags: [apps, networking, tcpdump, wireshark]
comments: true
---

# Aplicativos móveis sem criptografia

Quando os primeiros sites com transações eletrônicas surgiram na Internet,
houve uma certa resistência até que as pessoas dessem seu voto de 
confiança. Bem nesta época surgiram vários relatos de ataques de 
hackers, levando prejuízos enormes a pessoas e empresas.

Para combater os problemas de segurança, vários mecanismos foram
criados, como o HTTPS. Por meio de de um conjunto de relações de confiança
e de criptografia na transmissão dos dados, o HTTPS trouxe mais
credibilidade às comunicações web.

Sempre nos disseram, por exemplo, que era uma boa prática ter o "cadeado"
na janela do navegador, indicando uma conexão HTTPS.

![Cadeado HTTPS](/assets/posts_assets/2016-01-11-aplicativos-sem-criptografia.md_assets/cadeado.png ) 

Só que a evolução não parou aí: os desktops foram quase que totalmente
substituídos pelos smartphones e tablets em tarefas de comunicação.
Um problema existente nessa nova abordagem é que fica mais difícil 
saber como o aplicativo foi implementado e se ele utiliza realmente
algum tipo de criptografia e boas práticas de segurança.

Quando o app não usa HTTPS (ou alguma outra forma de comunicação segura)
é relativamente fácil para outras pessoas com acesso à rede na qual seu
telefone está conectado interceptarem o tráfego. Vejamos como.

## Cenário Fictício

Vamos imaginar uma empresa fictícia; um escritório de 
contabilidade com cerca de 100 funcionários. Como a maioria
das empresas deste porte, há uma sala com equipamentos de
informática e um analista responsável pelo suporte.
O analista mantém um desenho da rede para facilitar a administração:

![Mapa da Rede](/assets/posts_assets/2016-01-11-aplicativos-sem-criptografia.md_assets/mapa-rede.png) 

Como é de praxe, nossa empresa de exemplo 
mantém um firewall caseiro muito rudimentar,
composto por uma boa máquina com várias placas de rede e 
executando uma distro GNU/Linux com IPTables e Squid.
O servidor Linux é o Gateway padrão da rede, logo todo o tráfego que 
vai para a Internet acaba passando por ele. 

É cada vez mais comum encontrarmos empresas que 
oferecem conexão wireless para os funcionários acessarem
redes sociais, banking, etc., já que os planos de dados 
do Brasil não são nenhuma maravilha. Neste caso não
é diferente.

Todos os dispositivos que se conectam em uma rede doméstica recebem
um número IP para identificação e diferenciação dos outros.

Certo. Munidos de todas estas informações, o que será que
um analista de TI com acesso ao firewall da rede consegue fazer?

## Sniffing com TCP Dump

Com um celular conectado à rede wireless, o primeiro passo é descobrir seu
IP. Quando temos um servidor Linux fornecendo IPs via  DHCP, 
essa é uma tarefa simples. Basta olhar o arquivo de **leases**.
No servidor que usamos como exemplo, o leases está em um ambiente
de chroot, por razões de segurança. Vejamos se há algum 
dispositivo Android ou iOS nesta rede:

`[root@firewall: /]# egrep -i 'lease|android|iphone' /var/dhcpd/var/db/dhcpd.leases`

Resultado:

![Leases](/assets/posts_assets/2016-01-11-aplicativos-sem-criptografia.md_assets/leases.png) 

Agora que já escolhemos o IP do alvo, utilizaremos o comando `tcpdump`, 
que captura todo o tráfego que passa pelo servidor. 
Para não gerar muito tráfego, utilizaremos o IP do celular alvo como filtro:

`[root@firewall: /]# tcpdump -i em0 -A host 10.200.1.14`

Resultado:

![Tcpdump](/assets/posts_assets/2016-01-11-aplicativos-sem-criptografia.md_assets/tcpdump.png) 

Lembrando que 10.200.1.14 é o IP do telefone alvo.
A opção `-i` especifica a placa de rede utilizada e a opção `-A` decodifica
os pacotes de uma forma humanamente legível.

Enquanto a sequência `CTRL+C` não for pressionada, todo o tráfego
com origem ou destinado a este telefone será exibido na tela.

OK, mas qual o conteúdo deste dump? A resposta é **DEPENDE**.
Se o aplicativo executado durante a captura utilizou criptografia,
então o conteúdo será ilegível (ao menos para pobres mortais sem 
os conhecimentos dos hackers). 

Vejamos como fica o arquivo capturado quando utilizamos o Facebook:

![Facebook](/assets/posts_assets/2016-01-11-aplicativos-sem-criptografia.md_assets/facebook.png) 

Fica claro que o Facebook utiliza criptografia na comunicação por meio
do seu aplicativo. Temos duas evidências disso:

 - A porta HTTPS (443/TCP) está sendo utilizada;
 - O payload (conteúdo dos pacotes) é ilegível.

Para demonstrar um exemplo de aplicativo que não utiliza estes recursos,
vamos utilizar o app de uma empresa de planos de saúde, 
disponível para Android na Play store.
(aplicativo baixado em janeiro de 2016)

![Unimed](/assets/posts_assets/2016-01-11-aplicativos-sem-criptografia.md_assets/unimed.png)

:O

Este app utiliza uma porta personalizada (777/TCP) sem HTTPS.
Conseguimos ver dados como número do cliente e senha de acesso,
trafegados livremente no payload dos pacotes.
Qualquer informação visível no app seria também visível
para o interceptador na rede.

Sabe o que é mais assustador? 
Existem muitos, muitos apps que não cuidam adequadamente dos dados dos usuários.
E o pior é que enquanto alguém analisa o tráfego da rede, 
o usuário não percebe nada!

## Análise com Wireshark

Olhar os dumps no modo texto pode ser um pouco chato.
Existe uma ferramenta chamada Wireshark que pode tornar
esta tarefa mais simples.

É muito fácil: o hacker acessa o servidor via SSH e dispara um 
comando para capturar uma série de arquivos de vários celulares dos
funcionários. No final do expediente, ele copia todos os arquivos e 
leva para casa. Em casa ele instala o Wireshark em seu PC e abre os
arquivos `.pcap`. Olha que bonito!

Capturando no servidor e copiando para o notebook: 
![PCAP](/assets/posts_assets/2016-01-11-aplicativos-sem-criptografia.md_assets/pcap.png) 

Abrindo no Wireshark:
![Wireshark](/assets/posts_assets/2016-01-11-aplicativos-sem-criptografia.md_assets/wireshark1.png) 

## Como se proteger?

É bem difícil. Não tem como o usuário saber imediatamente 
se o aplicativo utiliza ou não HTTPS. 
Diferentemente dos sites, que exibem o protocolo na barra de 
endereços do navegador, um aplicativo pode ocultar todos os detalhes
de sua implementação. 

Por isso é importante utilizar aplicativos de fornecedores confiáveis,
e mesmo assim evitar digitar dados sigilosos quando
estiver em redes desconhecidas.
Evite apps feitos por empresas desconhecidas ou instalados de fontes 
estranhas.

Prefira apps das lojas oficiais, como Google Play e 
Apple Store, e não coloque informações como número de documento ou
cartão de crédito em aplicativos nos quais não confia.

Leia os comentários sobre o aplicativo. Muitos apps problemáticos
foram votados negativamente em suas respectivas lojas, é bem
comum encontrar os relatos dos usuários que foram vítimas.

É isso aí. Já foi espiado por algum bisbilhoteiro? Já desconfiou que
alguma rede na qual estava não era segura? Deixe seu relato nos
comentários. Abraço e até a próxima!
