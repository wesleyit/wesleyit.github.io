---
title:  Protegendo seus playbooks com Ansible Vault
description: >-
    Em algumas situações nos vemos forçados a colocar dados
    sensíveis - como credenciais, por exemplo - em um playbook.
    Em ambientes colaborativos, onde várias pessoas tem acesso
    aos arquivos, isso pode ser um problema. Vamos usar Ansible Vault
    para aumentar a segurança dos playbooks.
image_url: /assets/posts_assets/2016-06-01-ansible-vault.md_assets/lock.jpg
date: 2016-06-01 23:00:00 -0300
layout: blogpage
lang: pt_BR
tags: [ansible, devops, vault, playbook]
comments: true
---

Ansible - Mantendo seus playbooks em segurança
==============================================

O Ansible chegou chutando a porta e mostrando que não veio de brincadeira.
Em pouco tempo já é uma das ferramentas mais utilizadas por DevOps do mundo todo quando chega a hora de provisionar a infra como código.

Hoje vamos mostrar como utilizar o Ansible Vault, um recurso para criptografar arquivos de configuração.

Primeiro, vamos analisar o playbook a seguir:

``` yaml
---
- hosts: docker

  vars:
  - dbname: BANCOSECRETO
  - dbuser: superlogin
  - dbpass: impossible123

  tasks:
  - name: Instala o MySQL Server e outros pacotes úteis
    apt: name={{ item }} state=latest update_cache=yes cache_valid_time=3200
    with_items:
      - mysql-server
      - mysql-client
      - python-mysqldb

  - name: Habilita e inicia o serviço
    service: name=mysql enabled=yes state=started

  - name: Cria um usuário administrador do banco de dados
    mysql_user: name={{ dbuser }} password={{ dbpass }} priv=*.*:ALL state=present

  - name: Cria o banco de dados
    mysql_db: name={{ dbname }} state=present

```

Nada de muito avançado. Este playbook utiliza os módulos nativos do Ansible para instalar o MySQL Server,
iniciar o serviço e criar uma base de dados com um usuário e senha específicos.

Parece que tudo funciona como planejado:

```
[wesley@maclinux-01: ~/Ansible]$ ansible-playbook playbook.yml

PLAY ***************************************************************************

TASK [setup] *******************************************************************
ok: [172.17.0.1]

TASK [Instala o MySQL Server e outros pacotes úteis] ***************************
ok: [172.17.0.1] => (item=[u'mysql-server', u'mysql-client', u'python-mysqldb'])

TASK [Habilita e inicia o serviço] *********************************************
ok: [172.17.0.1]

TASK [Cria um usuário administrador do banco de dados] *************************
ok: [172.17.0.1]

TASK [Cria o banco de dados] ***************************************************
ok: [172.17.0.1]

PLAY RECAP *********************************************************************
172.17.0.1                 : ok=5    changed=0    unreachable=0    failed=0

```

O playbook rodou. Vamos ver no host se o banco foi criado?

```
root@11b56f5b4e1e:~# mysql -usuperlogin -pimpossible123
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 71
Server version: 5.5.47-0+deb8u1 (Debian)

Copyright (c) 2000, 2015, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| BANCOSECRETO       |
| mysql              |
| performance_schema |
+--------------------+
4 rows in set (0.00 sec)

mysql>
```

Ótimo, realmente funcionou. Só tem um problema:
Este playbook gerencia a nossa infra como código, e como sabemos, código deve ser testado, avaliado,
versionado e armazenado em um repositório remoto que garanta sua disponibilidade.
Várias pessoas poderão ter acesso a este código.
**Parece uma boa ideia salvar o usuário e a senha do banco de dados no arquivo de playbook?**

Vamos refazer nosso playbook de uma forma mais bacana.
Para não perder nosso primeiro playbook - que apesar de inseguro, funciona bem - vamos salvá-lo
com outro nome. Vamos também criar um arquivo no qual colocaremos nossas variáveis secretas:

```
[wesley@maclinux-01: ~/Ansible]$ ls
playbook.yml
[wesley@maclinux-01: ~/Ansible]$ cp playbook.yml unsafe_playbook.yml
[wesley@maclinux-01: ~/Ansible]$ touch my_secrets.yml
```

Agora precisamos extrair do `playbook.yml` a seção de variáveis e colocá-la no arquivo `my_secrets.yml`, sem o `vars:`.
No arquivo `playbook.yml` vamos colocar o `vars_files:` com o `my_secrets` exatamente onde ficava o bloco de variáveis.

Olha como fica:

```
[wesley@maclinux-01: ~/Ansible]$ cat playbook.yml
---
- name: Instalacao do Banco de Dados
  hosts: docker
  vars_files:
    - my_secrets.yml

  tasks:

  - name: Instala o MySQL Server e outros pacotes úteis
    apt: name={{ item }} state=latest update_cache=yes cache_valid_time=3200
    with_items:
      - mysql-server
      - mysql-client
      - python-mysqldb

  - name: Habilita e inicia o serviço
    service: name=mysql enabled=yes state=started

  - name: Cria um usuário administrador do banco de dados
    mysql_user: name={{ dbuser }} password={{ dbpass }} priv=*.*:ALL state=present

  - name: Cria o banco de dados
    mysql_db: name={{ dbname }} state=present

[wesley@maclinux-01: ~/Ansible]$ cat my_secrets.yml
- dbname: BANCOSECRETO
- dbuser: superlogin
- dbpass: impossible123

[wesley@maclinux-01: ~/Ansible]$

```

Isso que fizemos é uma boa prática: separamos as variáveis da lógica principal do playbook,
facilitando e encorajando a sua reutilização. Vamos ver se ainda funciona?

```
[wesley@maclinux-01: ~/Ansible]$ ansible-playbook playbook.yml

PLAY [Instalacao do Banco de Dados] ********************************************

TASK [setup] *******************************************************************
ok: [172.17.0.1]

TASK [Instala o MySQL Server e outros pacotes úteis] ***************************
ok: [172.17.0.1] => (item=[u'mysql-server', u'mysql-client', u'python-mysqldb'])

TASK [Habilita e inicia o serviço] *********************************************
ok: [172.17.0.1]

TASK [Cria um usuário administrador do banco de dados] *************************
ok: [172.17.0.1]

TASK [Cria o banco de dados] ***************************************************
ok: [172.17.0.1]

PLAY RECAP *********************************************************************
172.17.0.1                 : ok=5    changed=0    unreachable=0    failed=0

[wesley@maclinux-01: ~/Ansible]$

```

Funciona! Mas espera aí... não adiantou nada! Ainda conseguimos ver as senhas no
arquivo `my_secrets.yml`. É neste momento que o **Ansible Vault** entra em cena.

Vamos criptografar o arquivo `my_secrets.yml`:

```
[wesley@maclinux-01: ~/Ansible]$ ansible-vault encrypt my_secrets.yml
New Vault password:
Confirm New Vault password:
Encryption successful

[wesley@maclinux-01: ~/Ansible]$ cat my_secrets.yml
$ANSIBLE_VAULT;1.1;AES256
34613061383235393130326238386132393762356635363538316166623663666434623731356161
3039373663383536373732306437643036386563366132640a383832386234383535633935336530
32313332363039313834663433366337313861626232363861363763623264336335336137633430
3439616361336462370a356531353530373730663465373333383961653933363965326662363336
64386466663131333336643762613337393565626462613364356166663664393064323339666532
37646135343064333631643164653237366162643539343939313938366565353861616563363664
39323532303566653734346337373531376331383366633532366530333161663032363335373438
64363237303433356339
[wesley@maclinux-01: ~/Ansible]$

```

Olha que show! Depois de colocar uma senha de criptografia (por favor, que não seja 12345, OK?) temos
um arquivo ilegível.

Resolvemos nosso problema, mas será que o Ansible ainda consegue executá-lo?
Vejamos:

```
[wesley@maclinux-01: ~/Ansible]$ ansible-playbook playbook.yml
ERROR! Decryption failed
[wesley@maclinux-01: ~/Ansible]$

```

Por mil ouriços! De que adianta criptografar um arquivo se não posso mais utilizá-lo?
Isso não faz sentido. Vamos olhar o help do Ansible:

```
[wesley@maclinux-01: ~/Ansible]$ ansible-playbook --help | grep vault
  --ask-vault-pass      ask for vault password
  --new-vault-password-file=NEW_VAULT_PASSWORD_FILE
                        new vault password file for rekey
  --vault-password-file=VAULT_PASSWORD_FILE
                        vault password file
[wesley@maclinux-01: ~/Ansible]$

```

Mexilhões! Agora tudo faz sentido! Como o Ansible poderia abrir o arquivo se só nós sabemos a senha?
Vamos passar a senha para o Ansible na hora da execução:

```
[wesley@maclinux-01: ~/Ansible]$ ansible-playbook playbook.yml --ask-vault-pass
Vault password:

PLAY [Instalacao do Banco de Dados] ********************************************

TASK [setup] *******************************************************************
ok: [172.17.0.1]

TASK [Instala o MySQL Server e outros pacotes úteis] ***************************
ok: [172.17.0.1] => (item=[u'mysql-server', u'mysql-client', u'python-mysqldb'])

TASK [Habilita e inicia o serviço] *********************************************
ok: [172.17.0.1]

TASK [Cria um usuário administrador do banco de dados] *************************
ok: [172.17.0.1]

TASK [Cria o banco de dados] ***************************************************
ok: [172.17.0.1]

PLAY RECAP *********************************************************************
172.17.0.1                 : ok=5    changed=0    unreachable=0    failed=0

[wesley@maclinux-01: ~/Ansible]$

```

Para editar o arquivo depois de criado também é muito fácil.
Basta exportar a variável `EDITOR` com o seu editor de texto favorito.
Depois disso é só executar o `ansible-vault` com o parâmetro `edit` e o nome do arquivo,
colocar a senha e fazer as alterações necessárias. Veja:

```
[wesley@maclinux-01: ~/Ansible]$ ansible-vault edit my_secrets.yml
Vault password:

```

![Kate editando o arquivo criptografado](/assets/posts_assets/2016-06-01-ansible-vault.md_assets/kate.png)

Sucesso! Traz lá uma cerveja gelada porque agora nossos playbooks estão em segurança \o/\o/\o/
Mas lembre-se: utilize sempre uma senha segura e avalie se há realmente a necessidade de
colocar algo muito secreto em um arquivo público.

Dúvidas? Sugestões? Deixe seu comentário ;)
