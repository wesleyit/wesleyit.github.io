---
title: Criptografando diretórios no Linux
description: >-
    Existem várias formas de criar partições seguras no Linux.
    Mas e se você esqueceu de fazer isso durante a instalação?
    Aprenda a criar facilmente uma pasta criptografada para proteger
    seus arquivos.
image_url: /assets/posts_assets/2015-10-01-criptografando-diretorios-no-linux.md_assets/lock_dir.jpg
date: 2015-10-01 18:00:00 -0300
layout: blogpage
lang: pt_BR
tags: [linux, encrypt, disk]
comments: true
---



# Diretórios criptografados no Linux

Está trabalhando naquele "super projeto" e anda preocupado com a segurança dos seus dados?

Faz todo sentido, afinal a grande maioria dos desenvolvedores trabalha em um notebook.

Sexta-feira, voltando para casa, carro parado no semáforo e pimba! Perdeu, playboy! 
Além do incoveniente de perder os dados (podemos falar de estratégias de backup outro dia), imagina todo o seu código simplesmente "passeando" por aí?

Para evitar (ou pelo menos dificultar bastante) o acesso aos seus dados no HD, uma boa saída é criptografá-los.

No Mac OS X basta ativar o [FileVault](https://support.apple.com/pt-br/HT204837). No Windows também é fácil, é só ligar o [Bitlocker](http://windows.microsoft.com/pt-br/windows-8/bitlocker-drive-encryption). No Linux em um clique, durante a instalação do Ubuntu, Debian, Fedora, OpenSuSE e muitas outras distribuições, é possível ativar a criptografia no diretório `/home` ou até mesmo no disco todo.

Mas e se seu Linux já está instalado e você não usou criptografia? E se você quer criptografia apenas para uma pasta específica?
O Linux utiliza o LUKS como padrão para criptografia de disco. Por meio do módulo DM-Crypt, incluso no Kernel, e dos utilitários CryptSetup, é possível configurar uma infinidade de cenários. O problema é que estes programas são complicados, a documentação não é tão clara e amigável, e problemas na configuração podem fazer a máquina parar de inicializar, sendo necessário reformatar o computador. Assustador, não é?

Vamos mostrar hoje uma solução alternativa usando o [eCryptFS](http://ecryptfs.org/about.html) e o [EncFS](https://github.com/vgough/encfs). O eCryptFS é uma solução opensource e é utilizado como base para o sistema de criptografia de pasta de usuário no Ubuntu Linux e também no Google Chrome OS. O EncFS utiliza as capacidades de criptografia do eCryptFS para criar um filesystem criptografado acessível via FUSE.

Como queremos facilidade, vamos incluir uma ferramenta gráfica para gerenciar nossa pasta segura.

Vamos começar? Estou utilizando um Debian Jessie, mas os processos mostrados aqui são parecidos no Ubuntu e outras distros baseadas em Debian. Para outras distros, procure no seu gerenciador de pacotes.

O primeiro passo é instalar os aplicativos necessários:

```
$ sudo apt-get install -y encfs ecryptfs-utils
Reading package lists... Done
Building dependency tree       
...
```

Já instalamos os aplicativos que fazem a parte pesada, vamos agora instalar a interface gráfica de gerenciamento.
No Debian/Ubuntu, digite o comando abaixo para adicionar o repositório e instalar o software:
```
$ sudo add-apt-repository ppa:gencfsm
$ sudo apt-get update
$ sudo apt-get -y install gnome-encfs-manager
```

Pacotes para outras distros podem ser encontrados 
[aqui.](http://software.opensuse.org/download.html?project=home:moritzmolch:gencfsm&package=gnome-encfs-manager)

Isso é tudo que precisamos para criar a pasta segura.
Procure na sua lista de aplicativos pelo Gnome EncFS Manager:

![Gnome EncFS Manager](/assets/posts_assets/2015-10-01-criptografando-diretorios-no-linux.md_assets/snapshot1.png)

Ao executar o programa, um ícone (um desenho de uma pasta com uma chave dentro) aparece na área de notificação:

![Gnome EncFS Manager Icon](/assets/posts_assets/2015-10-01-criptografando-diretorios-no-linux.md_assets/snapshot2.png)

Se a janela principal do aplicativo não for exibida, clique no ícone e selecione a opção **Show Manager**.
Esta é a aparência do programa:

![Gnome EncFS Manager Window](/assets/posts_assets/2015-10-01-criptografando-diretorios-no-linux.md_assets/snapshot3.png)

Antes de criar a pasta segura, vamos entender como o app funciona.
O EncFS cria uma pasta na qual os arquivos criptografados são armazenados. 
Essa pasta está sempre disponível, mas nunca iremos acessá-la diretamente.
Ao abrir o programa, sua senha será solicitada e todos os arquivos aparecerão descriptografados na sua pasta de projeto.
Por isso, criar a pasta criptografada com um ponto no início do nome é uma boa ideia, assim ela fica oculta e evitamos acessá-la.

Clique no **+**. Na janela seguinte, o programa oferece algumas opções que podem ser utilizadas sem medo: 
um diretório `$HOME/Encfs/` será criado com duas subpastas, uma visível (`Private/`), na qual você colocará seus projetos e que chamaremos de pasta protegida, e outra invisível (`.Private/`), usada internamente pelo programa para colocar os arquivos criptografados, que chamaremos de pasta criptografada.

Aceite as opções default do programa, escolha uma senha forte - lembre-se de não anotá-la em um post-it colado na sua tela - e clique em create:

![Gnome EncFS Manager new folder](/assets/posts_assets/2015-10-01-criptografando-diretorios-no-linux.md_assets/snapshot4.png)

Pronto! Já temos a pasta configurada. Na janela principal ela aparece como "montada":

![Gnome EncFS Manager folder](/assets/posts_assets/2015-10-01-criptografando-diretorios-no-linux.md_assets/snapshot5.png)

**Nota:** Se você já tem uma pasta de projetos, não a selecione como ponto de montagem para a pasta protegida. Você deve criar essa pasta em um novo local e mover seus arquivos para dentro dela após criada e montada.

Com isso, encerramos a configuração da nossa pasta protegida. Porém, se você (assim como eu) é curioso, vai querer saber como as coisas funcionam por baixo do capô.
Para ver como o EncFS trata os arquivos, vou entrar na pasta protegida e criar um novo arquivo de texto:

![folder](/assets/posts_assets/2015-10-01-criptografando-diretorios-no-linux.md_assets/snapshot6.png)

Agora, por meio do gerenciador, vou desmontar a pasta protegida e tentar acessar os arquivos na pasta criptografada.
Basta desmarcar a opção "mounted":

![unmount](/assets/posts_assets/2015-10-01-criptografando-diretorios-no-linux.md_assets/snapshot7.png)

E tentar acessar a pasta `.Private/` ao invés da `Private/`:

![.Private](/assets/posts_assets/2015-10-01-criptografando-diretorios-no-linux.md_assets/snapshot8.png)

Maravilha! Tanto o nome do arquivo quanto seu conteúdo estão criptografados e protegidos dos terríveis hackers ladrões de HDs.

É isso, pessoal. 
Agora é só lembrar de montar a sua pasta na inicialização do PC (isso não pode ser automatizado porque requer a sua senha) e colocar seus arquivos importantes lá dentro.
Tire um tempo para ler a página de FAQ do [Gnome EncFS Manager](https://answers.launchpad.net/gencfsm) e aproveite os campos abaixo para deixar suas dúvidas.
Até a próxima!
