# Arquitetura *Plug-in based* em C++

----

Há um tempo trabalho com softwares que adiciom novas funcionalidades utilizando bibliotecas dinâmicas. Sem necessidade de recompilar a aplicação e nem mesmo linkar a biblioteca na hora do build, basta inserir um arquivo `.so` ou `.dll` em uma determinada pasta e pronto, a funcionalidade está lá quando a aplicação encontrar a biblioteca.

Depois de alguns projetos, comecei a me interessar mais e notar que esse tipo de arquitetura é muito utilizada em outras aplicações, principalmente se o intuito for extensibilidade e modularidade. Durante minhas pesquisas, notei que era complicado achar um artigo bom onde explicava o que eram essas bibliotecas dinâmicas e como elas eram construídas e inseridas em determinada aplicação. Então, decidi compartilhar um pouco do que aprendi sobre essas bibliotecas dinâmicas, chamadas `Plugins`, e como criar uma arquitetura básica utilizando a linguagem C++.

Vamos abordar um pouco sobre:

* O que é um plugin;
* *Dynamic Libraries* e *Static Libraries*;
* O que é um *plugin* em C++?;
* Como carregar um *plugin* em C++?;
* Quais as vantagens de utilizar a arquitetura baseada em *plugins*?


# O que são Plugins?
see [Wikipedia](https://en.wikipedia.org/wiki/Plug-in_(computing))

Plugins são extensões para um determinado software. São rotinas feitas em cima de um software para extendê-lo, sem prejudicar seu código interno e sem necessidade de recompilar o código principal.

Os plugins que vamos fazer ao longo desse paper extendem funcionalidade a executaveis, são chamados de [bibliotecas](https://pt.wikipedia.org/wiki/Biblioteca_(computa%C3%A7%C3%A3o)). Essas bibliotecas podem ser *linkadas* com o software para que você não precise ficar copiando e colando código por aí, ou, no nosso caso, pra extender o seu software sem necessidade de modificá-lo.


## *Dynamic Libraries* vs *Static Libraries*
see [*Difference between static and shared libraries?*](https://stackoverflow.com/questions/2649334/difference-between-static-and-shared-libraries)

As bibliotecas são diferenciadas entre *Dynamic Libraries* (Bibliotecas Dinâmicas) e [*Static Libraries* (Biliotecas Estáticas)](https://en.wikipedia.org/wiki/Static_library). As *Static Libraries* são arquivos *.a* (Unix) ou *.lib* (Windows). São códigos prontos para serem imbutidos em um executável e o processo acontece em tempo de compilação. Aumentam consideravelmente o tamanho do executável, já que toda a biblioteca é copiada na hora da compilação. Não é a que a gente tá procurando no momento.

O que a gente quer são as [*Dynamic Libraries*](https://en.wikipedia.org/wiki/Library_(computing)#Shared_libraries). São os arquivos *.so* (Unix), *.dll* (Windows) ou *.dylib* (OSX). São códigos compilados e prontos para serem utilizados em *run-time*. Melhoram o reuso de código e/ou extende a aplicação sem aumentar o seu tamanho, já que as informações são carregadas em tempo de execução. Esse tipo de biblioteca é a qual vamos utilizar na nossa arquitetura.

# Criando plugins em C++
## O que é um plugin em C++?

Como vimos acima, um plugin é uma bilioteca dinamica. Essa biblioteca foi compilada utilizando interfaces do programa ao qual queremos extender para que o programa possa carregá-las e plugá-las em seu devido lugar. Ou seja, digamos que temos um software que imprime as informações de cada plugin na tela. Para isso, precisaríamos de uma interface que representa o `Plugin`:

```
class Plugin
{
public:
    virtual std::string getName() const = 0;

    virtual std::string data() const = 0;
};
```

Note que essa é uma interface abstrata. Ela não possui nenhuma implementação, logo não é possível criar um objeto utilizando apenas essa interface, precisamos extendê-la para isso:

```
class CustomPlugin : public Plugin
{
public:
    CustomPlugin();

    std::string getName() const override;

    std::string data() const override;
}
```

Agora nós temos a classe que representa o nosso plugin customizado. Com essa classe, a nossa aplicação poderá ler seu nome e imprimir seus dados na tela, como veremos mais a frente.

TODO: Adicionar passo de build

## Carregando o plugin

Mas ainda tem um problema, como iremos criar o objeto dessa classe? A aplicação não pode criar usando `new CustomPlugin()` porque ela não conhece `CustomPlugin`, então a aplicação precisa fornecer um modo que o plugin consiga retornar sua referencia como `Plugin` e não como `CustomPlugin`. Esse é o maior desafio para carregar uma biblioteca, principalmente se o seu software for multiplataforma. 

Aqui nós vamos trabalhar apenas com a [`dlopen API`](http://www.tldp.org/HOWTO/html_single/C++-dlopen/), uma biblioteca que fornece funções para leitura, link e acesso a uma biblioteca dinâmica. Com ela conseguimos fazer tudo que precisamos em um sistema linux.

Não entraremos em detalhes de como criar em outros SOs, mas deixaremos uma estrutura pronta para quando tivermos a solução só acrescentar à nossa estrutura sem a necessidade de modificar o código base.


### dlopen API

Para fornecer a criação do objeto utilizando a `dlopen API`, vamos precisar criar uma interface contendo um método para criação de `Plugin` e outro para deleção de `Plugin`, que os plugins terão acesso:

```
typedef Plugin* create_t();
typedef void delete_t(Plugin*);
```

Agora nós temos a interface para criar e destruir o `Plugin`. Então, devemos acrescentar em nossa biblioteca suas implementações:

```
extern "C" Plugin* create_t()
{
    return new CustomPlugin(); // Automaticamente convertido para `Plugin`
}

extern "C" void delete_t(Plugin* plugin)
{
    delete plugin;
}
```

Esse é o nosso criador do `CustomPlugin`. Podemos colocar ele no mesmo arquivo do nosso `CustomPlugin` para facilitar. Agora nossa aplicação ja consegue criar e destruir qualquer `Plugin`.
Note que usamos a *keyword* [`typedef`](http://en.cppreference.com/w/c/language/typedef). Ela é necessária para que a aplicação possa encontrar a função dentro da biblioteca. E como usamos `typedef`, precisamos dizer que a nossa função é em C, por isso acrescentamos `extern "C"` ao inicio da função.

Agora basta compilar nosso plugin como uma biblioteca dinâmica. Utilizando `CMake` seria com o comando: `add_library(${PROJECT_NAME} SHARED \"CustomPlugin.cpp\")`, onde `CustomPlugin.cpp` representa a implementação da nossa classe `CustomPlugin` e a implementação das funções para criação do plugin. Feito isso, conseguimos gerar nosso `libcustomplugin.so`, que é o arquivo que vamos carregar para a criação do objeto `CustomPlugin`.

### Carregando utilizando `dlopen`

Nós criamos a biblioteca, agora precisamos carregá-la. Para isso, precisamos utilizar a biblioteca `dlopen` e para termos acesso às funções `dlopen()`, `dlsym()` e `dlclose()`. A função `dlopen()` tenta carregar a biblioteca dinâmica, retornando um `handle` para nosso plugin, caso consiga. Com esse `handle`, iremos procurar pela função `create`, que é a qual criamos lá em cima, utilizando a função `dlsym()`. Se tudo der certo, teremos uma referencia para a função, que poderá ser chamada a qualquer momento enquanto a biblioteca estiver aberta. Quando terminarmos de usar nosso plugin, basta chamar `dlclose()`.

Então, seguindo essa lógica criamos o método para carregar a biblioteca:

```
#include <dlfcn.h>

bool DlLoader::openLib()
{
    void *handle;
    char *error;

    handle = dlopen("libcustomplugin.so", RTLD_LAZY);
    if (!handle) {
        std::cout << "Failed to load plugin: " << dlerror() << std::endl;
        return false;
    }
    dlerror();    // Limpa qualquer erro para verificarmos o link depois.

    // Criamos um ponteiro para a nossa função `create_t`, assim podemos utiliza-lá depois
    create_t *create_plugin = (create_t*) dlsym(plugin_h, "create")

    if ((error = dlerror()) != NULL)  {
        std::cout << "Failed to load plugin: " << dlerror() << std::endl;
        return false;
    }
    // Tentamos criar o nosso `CustomPlugin`.
    Plugin* customPlugin = create_plugin();
    if (customPlugin) {
        std::cout << "Plugin: " << customPlugin->getName() << std::endl;
        std::cout << "Plugin data:" << customPlugin->getData() << std::endl;
    }
        
    dlclose(handle);
    return true;
}
```

E é isto, com isso já é possível carregar uma biblioteca dinâmica. Mas até então não temos uma arquitetura, e é aí que entra a mágica. Para deixar mais bem estruturado e modular, precisamos criar uma estrutura basica de criação dos plugis para que eles fiquem disponíveis facilmente e em um tipo funcional, não como `IPlugin`.

## Modelo básico

Então pra criar uma estrutura modular e simples, vamos criar algumas classes/interfaces. Primeiro, vamos criar o `PluginLoader`, que irá fazer o trabalho que fizemos acima com o `DLLoader`. Depois, vamos criar a `PluginFactory`, onde os plugins serão criados, carregados com o `PluginLoader` e transformados em um objeto "externo" usando o `IPluginProxy`. Vamos passar por cada uma detalhadamente.

### _`PluginLoader`_
Então primeiro vamos definir o `PluginLoader`. O objetivo dessa classe é transformar uma lib fornecida em um objeto utilizável, que será o `IPlugin`. Então teremos:

```
class PluginLoader
{
public:
    PluginLoader();

    int loadPlugin(const std::string &pluginPath, std::shared_ptr<IPlugin> &plugin);
```

A implementação nós já fizemos acima em _Carregando utilizando `dlopen`_, se ainda tiver dúvida releia o tópico.

### `PluginFactory`

Para criarmos a `PluginFactory`, precisamos entender como um objeto do tipo `IPlugin` vai se transformar em algo útil para nós, senão todo esse trabalho será em vão já que até então ele serve apenas para ser carregado corretamente.
Como estamos utilizando C++, vamos fazer todo o trabalho utilizando subclasses e `casts`. A única coisa que precisamos fazer, é tornar a nossa classe `CustomPlugin` subclasse de uma interface que será útil para nossa aplicação. E quem vai saber como fazer essa transformação é o `IPluginProxy`, que será registrado na `PluginFactory`.
Vamos definí-lo como:
```
IPluginProxy
{
public:
    virtual ~IPluginProxy() = 0;

    virtual bool addPlugin(IPlugin *plugin) = 0;

    virtual std::string id() = 0;
};
```

>Se não entendeu, não se preocupe. O importante até então é entender que o `IPluginProxy` é uma interface fornecida por nós, que quando implementada e registrada na `PluginFactory`, vira o responsável por transformar esse nosso `IPlugin` um objeto funcional e armazená-lo para uso futuro. Os detalhes de como isso acontece vamos detalhar mais à frente.

Então, o objetivo da `PluginFactory` classe vai ser:
 * Registrar e manter referencias para `IPluginProxy`;
 * Carregar um plugin utilizando o `PluginLoader`;
 * Registrar o objeto `IPlugin` criado pelo `PluginLoader` em seu determinado `IPluginProxy`;

`PluginFactory.hpp`:
```
PluginFactory
{
private:
    std::vector<IPluginProxy> _proxies;

    PluginLoader _loader;

public:
    int createPlugins(const std::vector<std::string> &pluginsPaths);

    std::vector<std::shared_ptr<IPlugin>> plugins() const;

    bool registerProxy(std::shared_ptr<IPluginProxy> proxy);

}

```

`PluginFactory.cpp`
```
int PluginFactory::createPlugins(const std::vector<std::string> &pluginsPath)
{
    // Go through each directory and try to load. The path should be something
    // like "foolibpath/libfoo.lib" or "foolibpath/foo.dll" for windows
    for(const std::string &libPath : pluginsPath) {
        std::shared_ptr<IPlugin> plugin = nullptr;
        int rc = _loader.loadPlugin(libPath, plugin);
        if (rc == 0) {
            // Now we check if any proxy registered match with the plugin's proxy and
            // if so, add it to proxy, where the cast will be done and the object will
            // become available for anyone who has that proxy as a functional object.
            foreach (IPluginProxy *proxy : _proxies) {
                If (plugin->proxyId() == proxy->id()) {
                    proxy.addPlugin(plugin);
                }
            }
        } else {
            // Throw error
        }
    }
    return rerr;
}

bool PluginFactory::registerProxy(IPluginProxy *proxy)
{
    _proxies.push_back(proxy);
}
```

> Note que estamos utilizando o `PluginLoader` como membro direto da `PluginFactory`. Assim, ficamos preso a apenas um tipo de loader. No nosso caso temos apenas o loader para Linux, então caso fossemos acrescentar a funcionalidade para o windows, teríamos que mudar o nosso `PluginLoader` ou alterar a estrutura do `PluginFactory` para suportar mais de um tipo de `PluginLoader`. Para isso, basta fazer o mesmo que fizemos com os `PluginProxy`.

### Criando o Proxy para nossa funcionalidade.

Agora precisamos criar a estrutura que vai transformar nosso `IPlugin` em alguma funcionalidade. Para isso vamos precisar criar uma nova classe, que vai ser uma subclasse de `IPluginProxy`; e uma interface, que será a interface "funcional" da nossa aplicação a qual nosso `CustomPlugin` irá herdar para poder ser transformado de `IPlugin` para essa nova interface.
Nosso proxy vai se chamar `BarProxy`. O seu papel é transformar objetos do tipo `IPlugin` para o tipo `Bar`(nossa nova interface) e guardar suas referencias para acesso futuro.

Então nosso proxy será definido como:
```
#include "Bar.hpp"

class BarProxy : public IPluginProxy
{
private:
    std::vector<Bar*> _loadedPlugins;

public:
    BarProxy() {}
    ~BarProxy() {}

    bool addPlugin(Plugin* plugin) override {
        Bar *barPlugin = dynamic_cast<Bar*>(plugin);
        if (barPlugin != nullptr) {
            _loadedPlugins.push_back(barPlugin);
            return true;
        }
        return false;
    }
};
```

e nossa interface `Bar`:
```
class Bar
{
public:
    virtual ~Bar() = 0;

    // Implementação obrigatória na subclasse
    virtual void drawBar() = 0;

   // Implementação opcional.
   virtual void fooBar() {}
}
```

### Adaptando o `CustomPlugin` para a nova estrutura

Agora que criamos a estrutura completa, precisamos adaptar e `CustomPlugin` para a nova estrutura. Basta subclassear de `Bar` e adicionar a implementação dos métodos virtuais.
```
class CustomPlugin : public IPlugin, public Bar
{
    ...

    void drawBar() override {
         // ... implementation goes here
    }
}
```

### Buildando

Agora temos tudo pronto. Basta estruturar e ligar tudo pra fazermos o teste:

```
#include "PluginFactory.hpp"
#include "BarProxy.hpp"

void main()
{
    BarProxy *proxy = new BarProxy();
    PluginFactory factory;
    factory.registerProxy(proxy);
    factory.createPlugins(pluginsPath);

    std::vector<Bar*> bars = proxy.getBars(); // Gets a vector of plugins casted to Bar
    foreach (Bar *bar: bars) {
            bar->drawBar();
    }
}
```

TODO: Adicionar passos de build`}``}`
