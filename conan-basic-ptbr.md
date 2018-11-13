# Conan
_Tempo de leitura: X_
___

Nesse artigo, teremos uma noção do que é o _Conan_, a sua utilização básica e alguns problemas encontrados durante o meu aprendizado ao utilizá-lo. Talvez encontre informações aqui as quais não são detalhadas nem "completas", mas tento indicar sempre que possível o local (links) onde encontrar a resposta. Peço que leiam mais com o intuito de ter um "norte" de onde encontrar as respostas, do que algo detalhado e completo, afinal a [documentação](https://docs.conan.io/en/latest/) encontrada no próprio site do _Conan_ já é bem completa e de lá que tirei a maioria do assunto abordado aqui. Com base nisso, abordo um pouco sobre:

 * O que é o _Conan_
 * O que são os pacotes disponibilizados pelo _Conan_
 * Utilização básica
 * Utilizando _Conan_ no projeto
   * `conaninfo.txt` e `conaninfo.py`
   * Profiles
 * Criando pacotes para utilização no _Conan_
 * Problemas (e soluções) ao aplicar _Conan_ em soluções reais
 * Pros e Contras
 * Conclusão

### O que é o _Conan_
___
[Conan](https://docs.conan.io/en/latest/introduction.html) é um gerenciador de pacotes para C++ que é agnóstico a geradores e a compiladores com o intuito de facilitar o _build_ e _link_ de bibliotecas utilizadas para construção de um projeto C++. Ele é dividido em 2 partes, basicamente: O _client_ estilo _git_, usado para preparar o ambiente para que o seu _generator_, como o _CMake_, encontre e link as bibliotecas; e também para manter o _cache_ das bibliotecas já encontradas; e o(s) servidor(es) é onde o _client_ procura pelas receitas e binários de cada biblioteca que for necessária para o build do seu projeto.

[![N|Conan](https://docs.conan.io/en/latest/_images/systems.png)](https://docs.conan.io/en/latest/introduction.html)

O gerenciador de pacotes já possui duas formas "oficiais" onde os pacotes podem ser recuperados e/ou enviados, que são por meio do _JFrog Artifactory_ e do _JFrog Bintray_ - mas além dessas formas é possível fazer a instalação do _server_ em um computador qualquer para ser utilizado só na intranet[1], por exemplo.
No _JFrog Bintray_ é onde o principal respositório do conan está localizado, o [_conan center_](https://bintray.com/conan/conan-center). Além dele, outra comunidade chamada [_bincrafters_](https://bintray.com/bincrafters/public-conan) também está sendo de grande ajuda no processo de adaptação de bibliotecas já existentes e muito utilizadas como _Qt_, _boost_ e várias "não tão utilizadas" também. Já com esses dois servidores é muito provavel que as principais bibliotecas que utilizadas estarão disponíveis - e caso não esteja, o processo de criar um pacote para o _Conan_ não é muito difícil, principalmente se a biblioteca utilizar alguma ferramenta que o gerenciador de pacotes já possui.

___
[1]: Não me aprofundei no assunto sobre como criar um [_server_](https://docs.conan.io/en/latest/uploading_packages/running_your_server.html?highlight=server), ao que tudo indica basta fazer algumas configurações de porta, mas, de novo, não sei a complexidade.

### O que são os pacotes disponibilizados pelo _Conan_
___

Um pacote no _Conan server_ geralmente é apenas um arquivo de receita (`conaninfo.py` descrito mais abaixo) com algum mini projeto de teste, associado aos seus binários. Cada "pacote binário" (não é necessariamente um binário, pode ser um projeto _header only_) é criado baseado em suas opções e configurações de ambiente. Ou seja, para cada compilador diferente, um novo pacote binário será criado; para cada arquitetura diferente (x86 ou x64) um novo pacote binário será criado; caso você altere alguma configuração (habilitar o `shared`, por exemplo) um novo pacote será criado; e etc. Isso porque o ID de cada pacote binário é o `hash` das informações das configurações (`settings`) e opções (`options`), mudando qualquer parâmetro dentro desse conjunto de informações, um novo binário será construído, estando ou não disponível no repositório onde estão as receitas do _Conan_.

[![N|Solid](https://docs.conan.io/en/latest/_images/binary_mgmt.png)](https://docs.conan.io/en/latest/introduction.html#decentralized-package-manager)

Então, em resumo, se você está em um ambiente `Windows`, `x64`, querendo uma biblioteca X em formato `shared` (`dll`, nesse caso), ao rodar o `conan install`, o _Conan_ vai procurar no _server_ algum binário com as mesmas configurações de ambiente e com a opção `shared` nos binários ligados à receita da biblioteca X. Caso encontre, o mesmo será baixado e alocado em _cache_ na sua máquina; caso contrário, ele irá informar que não encontrou e te dará a opção de compilar a biblioteca X com suas configurações.


### Utilização básica
___
A utilização do _Conan_ é bem parecida com a do _git_. Ao digitar apenas `conan` a seguinte saída é exibida:
```
Consumer commands
  install    Installs the requirements specified in a recipe (conanfile.py or conanfile.txt).
  config     Manages Conan configuration.
  get        Gets a file or list a directory of a given reference or package.
  info       Gets information about the dependency graph of a recipe.
  search     Searches package recipes and binaries in the local cache or in a remote.
...
```
A interface é bem detalhada e a [documentação](https://docs.conan.io/en/latest/) ajuda bastante quando surgem dúvidas. Os principais comandos do _Conan_ (que eu precisei utilizar) são:
* `conan remote {add, list, remove...}`: Configura _remotes_ onde os pacotes serão pesquisados. Para adicionar o _remote_ do _bincrafters_, por exemplo, basta chamar `conan remote add bincrafters https://api.bintray.com/conan/bincrafters/public-conan`. As demais opções são para renomear, alterar url, remover e etc.

* `conan search PACKAGE_OR_PATTERN [-r REMOTE]`: Procura por um pacote instalado no cache local ou em um remote (server) se passado `-r`. Útil pra saber nomes completos, saber se existe ou as versões disponíveis. Por exemplo, preciso saber quais pacotes do `Qt` estão disponíveis, utilizo `conan search Qt* -r all`, assim ele me da a seguinte saída:
    ```
    Remote 'bincrafters':
    Qt/5.11.0@bincrafters/stable
    Qt/5.11.1@bincrafters/stable
    Qt/5.11.2@bincrafters/stable
    ...
    Remote 'osechet':
    Qt/5.6.2@osechet/stable
    Qt/5.7.1@osechet/stable
    Qt/5.7.1@osechet/testing
    Qt/5.8.0@osechet/stable
    Qt/5.8.0@osechet/testing
    ```
    
* `conan install DIR [-if DIR]`: O que ele faz, basicamente, é instalar os arquivos necessários para a utilização do conan no diretório atual, baseando-se no arquivo de configuração `conaninfo.txt|py` encontrado no diretório `DIR`. Geralmente a utilização acontece no diretório de build, mas é possível passar `-if [dir]` como parâmetro para direcionar as saídas para um diretório específico. Esse comando é, basicamente, o único utilizado[2] pra _consumir_ do _Conan_ depois que tudo está configurado. Esse comando, além de instalar os arquivos necessários para o build, realiza o download dos binários e/ou o código do pacote desejado para que o build seja realizado, salvando-os em _cache_ para caso algum outro projeto também precise, assim não há necessidade de solicitar pro servidor uma segunda vez se as configurações forem as mesmas.

* `conan source DIR [-sf DIR]`: Esse comando acessa as configurações do arquivo `conaninfo.py`[^3] e recupera o código do projeto. É utilizado mais como teste para verificar se o código está sendo recuperado corretamente sem que os outros passos (configuração, build, teste, etc...) sejam processados. Assim como o `-if` do `install`, o `-sf` indica o diretório em que o código será colocado.

* `conan build DIR [-sf DIR | -if DIR]`: Esse comando também funciona com o `conaninfo.py`, dado as configurações do arquivo ele tenta fazer o build do pacote. É possível especificar o diretório onde está o código utilizando `-sf` e onde estão os arquivos instalados (do próprio _conan_) utilizando `-if`.

___
[2]: Não que tenha utilizado **muito**...
[3]: O `conaninfo.txt` não é utilizado para recuperar código ou fazer o build, ele trata apenas das dependências enquanto o `conaninfo.py` trata também das informações de _como_ gerar um pacote para o próprio _conan_.

### Utilizando _Conan_ no projeto
___
Para que o _Conan_ saiba exatamente qual o binário compatível com o seu projeto/máquina, é necessário que exista um arquivo chamado `conaninfo.txt` ou `conaninfo.py` ditando as opções e configurações dos pacotes necessários para o projeto. Além dessas informações, um outro arquivo de _profile_ é levado em consideração para ditar informações sobre o ambiente ao qual o projeto será compilado - sistema operacional, compilador, gerador, etc.

##### `conaninfo.txt` e `conaninfo.py`
Os arquivos _conaninfo_ são utilizados pra definir configurações de como os pacotes desejados devem ser entregues. A diferença entre os dois é que o `conaninfo.txt` é mais básico, possui informações apenas sobre como os pacotes serão baixados e _linkados_ à aplicação. Já o `conaninfo.py`, além de conter informações dos pacotes, também pode conter informações para **gerar** um pacote, além da possibilidade de incluir rotinas usando _python_ e as ferramentas já fornecidas pelo _Conan_.

 * [`conaninfo.txt`](https://docs.conan.io/en/latest/reference/conanfile_txt.html): É um arquivo simples contendo algumas informações como `requires`, `generators` e `options`. Com essas informações, o _Conan_ verifica as dependências exigidas em `requires`, fornece as informações com base nos `generators` escolhidos e os binários de acordo com as `options` fornecidas. No arquivo abaixo temos um exemplo de uma aplicação utilizando `gtest`, `Qt` e `libzip`.
 
    _conaninfo.txt_
    ```
    [requires]
    gtest/1.7.0@bincrafters/stable
    Qt/5.8.0@osechet/stable
    libzip/1.5.1@bincrafters/stable

    [generators]
    cmake
    
    [options]
    gtest:shared=True
    Qt:shared=True

    ```
 
     Ao executar um `conan install` com as configurações do nosso arquivo `conaninfo.txt`, o _Conan_ vai verificar se os pacotes desejados (`gtest`, `Qt` e `libzip`) já existem com as mesmas configurações do meu projeto e caso não encontre fará o download para o cache. Depois de resolver a "instalação" das dependências, o _Conan_ vai gerar os arquivos necessários para a integração no projeto de acordo com o(s) seu(s) `generators`, no nosso caso o `CMake`. Nas `options` temos `gtest:shared=True` e `Qt:shared=True`, o que quer dizer que queremos que o `Qt` e o `gtest` sejam utilizados como _dynamic (shared) libraries_.
 

 * [`conaninfo.py`](https://docs.conan.io/en/latest/reference/conanfile.html): É um arquivo mais complexo que o `conaninfo.txt`, mas que possui maior gama de controles que podem serem utilizados com o _Conan_. Geralmente é mais utilizado para o empacotamento do projeto, mas para o consumo também possui algumas vantagens em comparação ao `conaninfo.txt`, como próprias funções do `python`. Como exemplo, vamos utilizar as mesmas dependências acima e com algumas funções a mais para facilitar o build.
 
   _conaninfo.py_
   ```
   from conans import ConanFile, CMake

   class TestingConan(ConanFile):
       settings = "os", "compiler", "build_type", "arch"
       requires = "gtest/1.7.0@bincrafters/stable", 
                  "Qt/5.8.0@osechet/stable", 
                  "libzip/1.5.1@bincrafters/stable"
       generators = "cmake"
       default_options = {"Qt:shared": True, "gtest:shared": True}

       def imports(self):
           self.copy("*.dll", dst="bin", src="bin") # From bin to bin
           self.copy("*.dylib*", dst="bin", src="lib") # From lib to bin
           
       def build(self):
           cmake = CMake(self)
           cmake.configure()
           cmake.build()
   ```

    O nosso `conaninfo.py` funciona basicamente da mesma forma que o `conaninfo.txt` acima, com 2 diferenças encontradas nos métodos `imports` e `build`. O `imports` também é possível ser utilizado no `txt`, o que ele faz é copiar todas as `*.dll` e `*.dylib` para a pasta `bin`. Já o método `build` é específico do arquivo `.py`, ele possibilita que você possa chamar o `conan build` para compilar o seu projeto, sem a necessidade de chamar o `cmake`. Por exemplo, nesse caso, dado que meu `conaninfo.py` esteja na pasta `my_project` e se estiver em uma pasta `my_project/build/`, bastaria chamar `conan install .. && conan build ..` para que meu projeto seja compilado.
    
##### Profile
O arquivo de [_profile_](https://docs.conan.io/en/latest/using_packages/using_profiles.html#) pode ser utilizado de duas formas: como _profile_ local da máquina, se encontra geralmente em `~/.conan/profile/`; ou como um _profile_ localizado em qualquer lugar do disco, como na raiz do projeto como exemplificado abaixo. Esse arquivo de _profile_ é um arquivo texto que irá definir algumas configurações para recuperar os binários ou para compilar as bibliotecas necessárias para o projeto.

Por padrão, o _Conan_ cria um _profile_ _Default_ na primeira utilização. Em um ambiente _Windows_ com _Visual Studio 12_ instalado, esse arquivo deve ficar assim:
 ```
 [settings]
os=Windows
os_build=Windows
arch=x86_64
arch_build=x86_64
compiler=Visual Studio
compiler.version=12
build_type=Release

[options]

[build_requires]

[env]
 ```

Cada vez que o comando `conan install` é chamado, esse profile `default` é utilizado para saber quais as configurações devem ser usadas para recuperar o binário correto. _"Ah, mas eu não quero compilar em `Release`, e o `build_type` descrito no arquivo é `Release`"_, nesse caso existe a opção de criar arquivos de _profile_ diferente para cada configuração. Então caso eu queira compilar nas mesmas configurações que o `default`, mas em `debug`, por exemplo, basta criar um arquivo de profile como esse:

 ```
 include(default) # utiliza configurações do profile "default"
 
 [settings]
 build_type=Debug
 ```
 
 Assim, eu posso utilizar o novo profile criado utilizando `conan install .. -pr debug-profile` - sendo `debug-profile` o nome do arquivo. Também é possível salvar arquivos de profiles diferentes na pasta `~/.conan/profiles/`, assim o comando `-pr` irá reconhecê-lo sem necessidade de saber o path completo.
 
### Criando pacotes para a utilização no _Conan_
___
TODO

### Problemas (e soluções) ao aplicar _Conan_ em soluções reais
___
TODO

### Pros e Contras
___
TODO

### Conclusão
___
TODO
