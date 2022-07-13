---
title:  Seu Jenkins e seu Docker estão seguros?
description: >-
    Usar o Jenkins já não é nenhuma novidade. É comum vê-lo em uso até mesmo
    em projetos que não fazem integração contínua, funcionando como um executor
    de tarefas agendadas. Usar Docker também não é nenhuma novidade. Ele chegou
    para ficar, e já não dá mais para viver sem ele <3.
    Mas será que a segurança está em dia?
image_url: /assets/posts_assets/2017-02-28-jenkins-security.md_assets/oops.png
date: 2017-02-28 11:00:00 -0300
layout: blogpage
lang: pt_BR
tags: [jenkins, devops, docker, testing]
comments: true
---

# Seu Jenkins está seguro?

Usar o Jenkins já não é nenhuma novidade. É comum vê-lo em uso até mesmo
em projetos que não fazem integração contínua, funcionando como um executor
de tarefas agendadas. Usar Docker também não é nenhuma novidade. Ele chegou
para ficar, e já não dá mais para viver sem ele <3

E como este ambiente costuma ser? Normalmente vemos uma máquina média ou grande
com Docker instalado, na qual são executados ao mesmo tempo vários contêineres
de serviços como Jenkins, Nexus, Gitlab, Apiary, Swagger, SonarQube, etc.

O Jenkins disponibiliza uma matriz de usuários com permissões bem fáceis de
configurar e restringir. Mas você já parou para analisar a segurança da
sua instalação de Jenkins como um todo, desde a máquina na qual ele está instalado?


## Penetrando no Jenkins

Para nosso cenário fictício, vamos simular uma máquina Linux com Docker, na qual há vários contêineres (entre eles o Jenkins) executando.

Nosso Jenkins está protegido com uma senha bastante segura, que não possuímos.
A única coisa que temos no momento é acesso SSH à máquina Docker na qual
o contêiner Jenkins foi iniciado.

Vamos lá?

Acessando o nosso Jenkins, damos de cara com a tela pedindo login e senha:

![Home Jenkins](/assets/posts_assets/2017-02-28-jenkins-security.md_assets/Screenshot_20170228_110752.png)

Sem chance. Poderíamos fazer aqui algum tipo de ataque de força bruta baseado
em uma lista de usuários e senhas comuns, mas não vamos por este caminho.

Que tal acessar o servidor? Após fazer o login, vamos acessar a pasta que foi
configurada como persistência local do Jenkins, que geralmente é montada
no contêiner como `/var/jenkins_home`.

Para saber a localização desse diretório, temos que descobrir o nome do
contêiner do Jenkins e depois inspecioná-lo, buscando seus pontos de montagem:

![Jenkins Home](/assets/posts_assets/2017-02-28-jenkins-security.md_assets/Screenshot_20170228_112218.png)

Uau! O Jenkins cria vários arquivos e diretórios. Tem coisas muito importantes
nesta pasta. Vamos iniciar conseguindo o login do próprio Jenkins,
depois poderemos entrar na interface e obter outros logins e senhas.

Enumerar os usuários é o primeiro passo. Basta dar um `ls` na pasta `users` e
pronto, já sabemos os logins que vamos atacar. Inclusive, temos acesso
ao hash da senha acessando seu arquivo `config.xml`:

![Password](/assets/posts_assets/2017-02-28-jenkins-security.md_assets/Screenshot_20170228_113411.png)

Ahhh, que pena! A senha é armazenada em hash JBCrypt. Isso quer dizer que não é
reversível, mas não vamos desistir. Temos que entrar na interface web.
Já que não sabemos qual senha gerou este hash, podemos colocar no lugar
um hash conhecido, gerado por uma senha bem fácil.

Edite o arquivo, duplique e comente a linha da senha, para que tenhamos
um recovery depois de conseguir acesso.
Troque o hash da linha copiada para:

`#jbcrypt:$2a$10$razd3L1aXndFfBNHO95aj.IVrFydsxkcQCcLmujmFQzll3hcUrY7S`

Fica assim:

![Alteracoes](/assets/posts_assets/2017-02-28-jenkins-security.md_assets/Screenshot_20170228_165308.png)

Este hash corresponde à senha **test**.

Depois de trocar o hash é só reiniciar o contêiner para que a correção seja
aplicada. Podemos usar um simples `docker restart <nomedocontêiner>` e
logar com as credenciais `admin` e senha `test`.

![Logando](/assets/posts_assets/2017-02-28-jenkins-security.md_assets/Screenshot_20170228_165528.png)

![Logado](/assets/posts_assets/2017-02-28-jenkins-security.md_assets/Screenshot_20170228_165608.png)

Sucesso! Imaginando que somos crackers, para não levantar suspeitas agora
criaríamos uma conta com privilégios de administrador e voltaríamos a senha
de admin para o hash antigo, que salvamos no comentário do `config.xml`.

Pronto, já temos acesso ao Jenkins. O que mais podemos descobrir?

Passeando pelas configurações em [Jenkins/Manage Jenkins/Configure System],
vemos que este servidor utiliza o plugin SSH. Podemos ver o usuário e a
senha salvos para o servidor `SRV-PRODUCAO.acme.br`:

![SSH](/assets/posts_assets/2017-02-28-jenkins-security.md_assets/Screenshot_20170228_191311.png)

Podemos selecionar o campo da senha e clicar em **inspecionar elemento**, para
ver no código fonte as propriedades desse campo.

![SSH Senha](/assets/posts_assets/2017-02-28-jenkins-security.md_assets/Screenshot_20170228_191503.png)

Que beleza! Acabamos de descobrir que a senha de root do servidor é
**Acapulco1983** por meio do texto em **value** no código fonte.

Este Jenkins também possui configurado o plugin **E-mail Notification**.
Será que conseguimos pegar algo aqui também?

![SMTP Senha](/assets/posts_assets/2017-02-28-jenkins-security.md_assets/Screenshot_20170228_191935.png)

Pegamos, mas parece que tem algo errado:

`xKFNT3SYxVyFKqpXpYBZWBcKrAfRbLmWOdUE9XsCuZI=`

Essa é a senha do usuário SMTP, mas está criptografada. Não vai ser tão
simples quanto a senha do SSH.

Sabemos que o Jenkins usa uma chave master para criptografar uma chave local,
e que usa essa chave local para criptografar todos os seus recursos.
O que vamos fazer é algo bem simples: dentro do console do próprio Jenkins,
vamos escrever um script groovy que vai usar as chaves para decodificar
a senha.

Vamos acessar o console groovy em [Jenkins/Manage Jenkins/Script Console] e
solicitar nossa senha. Coloque seu hash na função
`hudson.util.Secret.fromString` e deixe o titio Jenkins cuidar da
criptografia para você:

`hudson.util.Secret.fromString('xKFNT3SYxVyFKqpXpYBZWBcKrAfRbLmWOdUE9XsCuZI=')`

O resultado aparece no campo Result:

![Resultado Decrypt](/assets/posts_assets/2017-02-28-jenkins-security.md_assets/Screenshot_20170228_192629.png)

Agora sabemos que a senha do SMTP é `AguaDeJamaica`.

Pois é, pessoal. Não dá para confiar nos mecanismos de proteção de senha do
Jenkins. Imagina só um cenário corporativo com acesso a ambientes de produção...
É grave. Mas os problemas não se restringem ao Jenkins. O Docker pode ser
também um vetor de vários tipos de problemas de segurança.

Vamos imaginar um cenário com um servidor Docker com vários contêineres em
execução, nos quais os usuários acessam com contas limitadas, mas **sem acesso
como root**. O simples fato de permitir acesso ao Jenkins via Docker com
um ponto de montagem habilitado pode ser criativamente explorado.

Logado como `wesley`, um usuário limitado no host, mas membro do grupo
`docker`, podendo executar contêineres livremente, vou iniciar uma sessão
como `root` no Jenkins. Vou criar um shell usando o bash como base e
dar `SUID` para ele, lembrando de deixá-lo na pasta acessível no
Docker host.

![Shell Capiroto](/assets/posts_assets/2017-02-28-jenkins-security.md_assets/Screenshot_20170228_195003.png)

O que acontece quando executamos este shell com a opção `-p` no Docker
host, logados com uma conta limitada?

![Dominado](/assets/posts_assets/2017-02-28-jenkins-security.md_assets/Screenshot_20170228_195257.png)

Uaaaaaahahahahahahahahahahahahahahaha!!! Essa máquina foi conquistada!

E você, vai deixar seus servidores dando sopa?


## Protegendo seu CI

O Jenkins é um CI Server bastante competente, e o Docker é o queridinho do
momento. Já que não vamos deixá-los, como podemos torná-los mais seguros?

A notícia ruim é que não dá. Simplesmente não dá para ficar tranquilo.
Se alguém quiser mesmo entrar no Jenkins (ou em qualquer outro servidor),
tiver tempo, dinheiro e vontade, vai conseguir. Mas não estamos totalmente
indefesos, podemos nos precaver de ataques mais básicos. Algumas dicas:

 - Controle quem tem acesso via SSH ao Docker. Como vimos, com um console na
 máquina do Docker é possível fazer muita bagunça;
 - Fique de olho em arquivos com SUID. Com eles é possível elevar os privilégios
 sem ter o trabalho de achar as senhas de root;
 - Não permita que o Jenkins Master tenha executores em ambientes de produção.
 A melhor estratégia é manter o Jenkins Master com permissões limitadas e deixar
 os executores nos Slaves. Com Executores no Master, um job poderia acessar o
 sistema de arquivos e exibir o conteúdo dos arquivos secretos;
 - Cuidado com os acessos armazenados em plugins. Se precisar salvar login e
 senha, melhor reconsiderar;
 - Mantenha seu stack atualizado. Se fosse uma versão mais antiga do Jenkins,
 daquelas que tem todas as senhas no arquivo `credentials.xml`, seria possível
 descriptografar e obter todas elas!

E lembre-se: seu ambiente é tão seguro quanto seu elo mais frágil. Quanto mais
ferramentas e passos de complexidade forem adicionados ao CI, mais ampla será
a cobertura que você precisará analisar em busca de falhas de segurança.
Neste caso, falhas no SonarQube, Nexus e outros contêineres podem dar acesso
aos servidores de produção. Em outras palavras, mantenha as coisas simples.

PS: A senha do admin Jenkins era a palavra cabalística
**Parangaricutirimirruaro**.

Até o próximo post =)
