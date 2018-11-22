# Conan
_Reading Time: ~20 minutes_

_Translated from Portuguese by: **Ramon Nunes (ramonnunesunb@gmail.com)**_
___

In this article, I will try to generate a brief notion of what is Conan, its basic uses and some problems found during my learning process with it. You may not find detailed or complete informations here, but I will always try to indicate the location (links) to find these answers. I ask of you to read with the intent of finding a direction of where to look up for answers instead of something complete and detailed, as the oficial Conan [documentation](https://docs.conan.io/en/latest/index.html), where I took most of the subject here aproached, covers almost all regular cases.

A bit of the following topics will be approached: 
 * What is _Conan_
 * What packages are provided by _Conan_
 * Its basic uses
 * How to utilize Conan in the project:
   * `conaninfo.txt` e `conaninfo.py`
   * Profiles
 * Creating packages to use in _Conan_
 * Conclusion

### What is _Conan_
___
[Conan](https://docs.conan.io/en/latest/introduction.html) is a package manager for C++ that is nescient to generators and compilers in order to favor the build and link of libraries utilized in the building of a C++ project. It is divided, basically, in two parts: The client, git styled, used to arrange the enviroment in order for its generator (like CMake) to find and link libraries, and also used to keep the cache of the already found libraries; and the server or servers where the client searches the recipes and binaries on each necessary library to build your project. 

[![N|Conan](https://docs.conan.io/en/latest/_images/systems.png)](https://docs.conan.io/en/latest/introduction.html)

The pack manager already has two "official" ways to recover and/or send the packages, those are the _JFrog Artifactory_ and the _Jfrog Bintray_ - other than these two it is also possible to instal the server to be used in any PC, only on the intranet[1], for example. Conan's main repository, the _conan_ center, is located in Jfrog Bintray. Besides the conan center, another community named _bincrafters_ is being very helpful in the process of adapting already existing and regularly used libraries such as _Qt_, _boost_ and many "not so popular" others. It is very likely that with these two servers all the main libraries will be available - and in case it doesn't, the process of creating a _Conan_ package, even though expensive, is not hard, especially if the library is already prepared and some of the tools that the manager already posses are utilized. 

> [1]: I won't explore the subject of how to create a  [_server_](https://docs.conan.io/en/latest/uploading_packages/running_your_server.html?highlight=server), as far as I know you just need to make a few port settings, but, once again, I am not very sure on the complexity.

### What packages are provided by _Conan_
___

A pack in the Conan Server is, usually, just a recipe file  (`conaninfo.py` described below)  with some mini project as a test, associated to its binaries. Each "binary package" (not necessarily a binary, could be a _header only_ library) is made based on its enviroment options and settings. In other words, to each different compiler, a new binary package will be made; to each different architecture (x84 or x64) a new binary package will be made; in case you change any options (enabling the `shared`, for example) a new package will be made; and etc. This happens because the ID of each binary package is the `hash` of the settings and options informations, changed any parameter within this information set, a new binary will be used, available or not in the repository where the conan recipes are located.

[![N|Solid](https://docs.conan.io/en/latest/_images/binary_mgmt.png)](https://docs.conan.io/en/latest/introduction.html#decentralized-package-manager)

In short words, if you are in a `Windows`, `x64` enviroment, in need of a library X in the `shared` (`dll`, in this particular case) format, when executing the `conan install`, the _Conan_ will search the  _server_ for a binary with the same settings of enviroment and with the option `shared`[2] in the binaries linked to the library X recipe. If found, it will be downloaded and allocated in cache on your machine; if not found you will be informed and the option of compiling the library X with its settings will be given.[3]

> [2]: Notice that, even though it is standard, the `shared` option may not exist in every library for it could be an option named differently.

> [3]: If any binary is found for a specific environment, an error will be thrown by _Conan_ informing you and explaining that there is an option `--build=missing` that tells _Conan_ to build from sources if a binary package is missing.

### Its Basic Uses
___
Using _Conan_ is very similar to _git_. When typing just _conan_ the following exit is shown:
```
Consumer commands
  install    Installs the requirements specified in a recipe (conanfile.py or conanfile.txt).
  config     Manages Conan configuration.
  get        Gets a file or list a directory of a given reference or package.
  info       Gets information about the dependency graph of a recipe.
  search     Searches package recipes and binaries in the local cache or in a remote.
...
```
The interface is very detailed and the  [documentation](https://docs.conan.io/en/latest/) is very helpful when in doubt. _Conan_'s main command (used by me) are:
* `conan remote {add, list, remove...}`: Configure _remotes_ where the packages will be researched. to add the  _remote_ of the  _bincrafters_, for example, just call `conan remote add bincrafters https://api.bintray.com/conan/bincrafters/public-conan`. The remaining options are for renaming, changing the url, removing and etc. 

* `conan search PACKAGE_OR_PATTERN [-r REMOTE]`: Searches a package installed in the local cache or in a remote (server) if  `-r`is past. Useful to know full names and the existance or available versions. If I need to know which packages of the `Qt` are available, I use `conan search Qt* -r all`, and it shall give me the following exit:
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
    
* `conan install DIR [-if DIR]`: Basically this command installs the necessary files for conan's utilizations in the actual directory, based on the setting file `conaninfo.txt|py` found in the directoty `DIR`. Usually its usages happens in the build directory, but it's possible to use `-if [dir]` as a parameter to direct the exits to a specific directory. Basically this command is the only one used[4] to _consume_ from _Conan_ after everything is already configured. Besides installing the the necessary files for building, this command performs the download of the binaries and/or package codes desired for the acomplishment of the build, saving them in _cache_ in case another project needs it, this way there is no need to request a second time to the server if the settings are the same.

* `conan source DIR [-sf DIR]`: This command access the file `conaninfo.py`[5] settings and recover the project's code. It's usually used as a test to check if the code is being correctly recovered without processing the other steps (setting, build, test, etc...). Just as the `-if` from the `install`, the `-sf` shows the directory in which the source code will be placed.

* `conan build DIR [-sf DIR | -if DIR]`: this command also works with the `conaninfo.py`, given the file's settings it tries to make the package build. It is possible to specify the directory of the source code using `-sf`and where are the installed files[6] (from _conan_) using `-if`

> [4]: Only aplies if you're consuming Conan's dependencies without "creating too many roots" with the package manager, but it is also possible to integrate the project even more directly compiling it by using `conan build`, for example.

> [5]: The `conanfile.txt` is not used to recover code or make the build, it only deals with the dependencies while the `conanfile.py`also deals with the informations on how to generate a package for conan itself. 

> [6]: These files normally is used by generators (`.cmake` is one of them) to retrieve all the informations needed from the library, like library's path, include folders and etc.

### Using _Conan_ in the project
___
In order to make _Conan_ know exactly what is the compatible binary with your project/machine, a file named `conanfile.txt`or `conanfile.py`dictating the options and settings of the needed packages for the project is mandatory. Besides these informations, another profile file is taken into consideration to dictate informations on the enviroment where the project is going to be compiled - operational system, compiler, architecture and etc.

#### `conanfile.txt` and `conanfile.py`
The files _conanfile_ are used to define settings on how the desired packages must be delivered. The diference between them is that `conanfile.txt.`is the most basic, it has informations on how the packages will be downloaded and linked to the application, while the `conanfile.py`, besides holding informations on the packages, can also hold informations in order to generate a new package and has the possibility to include routines using _python_ and some tools already provided by _Conan_.

 * [`conanfile.txt`](https://docs.conan.io/en/latest/reference/conanfile_txt.html): Is a simple file containing some informations such as `requires`, `generators`amd `options`. With these informations, the _Conan_ verifies the dependencies required in `requires`, provide the informations based on the chosen `generators` and the binaries according to the provided `options`. We have an example of application using `gtest`, `Qt`and `libzip` in the file below.
 
    _conanfile.txt_
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
 
     When executing a `conan install` with our file setting `conanfile.txt`, the _Conan_ will verify if the desired packages (`gtest`, `Qt`and `libzip`) exists if there is any, and in case there isn't any in my local _cache_, it will be downloaded. After resolving the "instalation" of the dependencies, the _Conan_ will generate the needed files for the project integration according to your/yours `generators`, in our case the `Cmake`. On `options`we have the `gtest:shared=True`and `Qt:shared=True`, this means we want the `Qt`and the `gtest`to be used as dynamic (shared) libraries.
 

 * [`conanfile.py`](https://docs.conan.io/en/latest/reference/conanfile.html): Is a file more complex than `conanfile.txt`, but it posses a bigger gama of controls that can be used with Conan. Usually it is more used for the packing of the project, but for the consumption it also has a few advantagens when compared to `conanfile.txt`, such as `python`'s functions. We are going to use the same dependencies above with a few more functions to make the build easier as an example.
 
   _conanfile.py_
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

    Our `conanfile.py` basically works the same way the `conanfile.txt` above does with two differences, found in the `imports`and `build`methods. The `imports` can also be used on the `txt` and in this case it copies the `*.dll` and `*.dylib` to the `bin` folder. The `build` method is speciic of the `.py` file, it makes possible for you to call the `conan build` to compile your project, without needing to call for the `cmake`. For example, in this case you would only need to call the `conan install .. && conan build ..` in order for my project to be compiled, without the need to call `cmake ..`.
    
#### Profile
Our [_profile_](https://docs.conan.io/en/latest/using_packages/using_profiles.html#) file can be used in two different ways: as the machine local profile, usually found in `~/.conan/profile/`; or as a profile located anywhere in the disk, like at the root of the project such as it is exemplified bellow. This profile file is a text file that will define some settings related to your compilation eviroment in order to recover or compile the correct binaries that are compatible with your project.

Conan's standard procedure is to create a default profile in the first use. In a windows enviroment with Visual Studio 12 installed, this file should look like this:
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

Each time the `conan install` is called, this `default` profile is used to know what are the settings that must be used to recover the correct binary. "Ah, but I don't want to compile in `Release`, and the `build_type` described in the file is `Release`", in this case there is the option to create different profile files for each setting. So, in case I want to compile in the same settings as the `default`, but in `debug`, for example, all I have to do is create a profile file that looks like this: 

 ```
 include(default) # uses the settings and options described in "default" profile.
 
 [settings]
 build_type=Debug
 ```
 
That way I can use the new profile using `conan install .. -pr debug-profile`- with `debug-profile` as the file name. It is also possible to save different profile files in the folder `~/.conan/profiles/`, that way the command `-pr` will recognize it without the need to know the full path.
 
### Creating packages to use in _Conan_
___
In order to make a library in Conan available we need to create a package that is, basically, described by the file `conanfile.py`. This file, as said above, descrive some attributes and methods that Conan will read to execute the actions. In this file it is possible to make some of the informations about the package (name, version, description, url...), some settings and options that will be used (such as the default flag `shared`), the needed dependencies and the methods that Conan will call to prepare the code to be compiled, generate the needed settings, the compilation and the test available.

#### Workflow
 In conan official documentation, the [method](https://bincrafters.github.io/2018/02/27/Updated-Conan-Package-Flow-1.1/) used by bincrafters for the package development is indicated. The method consists in testing parts of the recipe before proceding to the next step - baby steps style. As most of the package development process is "attempt and error", when a step such as the source code download is ready (which is a simple step), it doesn't need to be repeated everytime you're testing the `build` or `package`. To do so, conan provides some commands, already comented in "Basic Uses", such as `conan create RECIPE_PATH company/branch -k` (`-k`for `--keep-source`) and the same command as the `-kb`(`--keep-build`) parameter. These parameters in the `create` command makes the flux easier by reutilizing the same code already downloaded by the `source()`and the same `binary` after the `build()` is already working. This means that it basically calls the command `create` skipping a few steps that are already made.
 
 That way you just have to test using the same command (`conan create`), adding parameters such as the `-kb` to skip a few steps and try only the part you are developing. Finishing the development of the `source()`, `build()` and `package()`, when `conan create` is working well, all you have to do is focus on creating a test miniproject and test it using the command `conan test TEST_FOLDER package/1.0@user/testing`. To make it easier, there is also an [example project](https://github.com/memsharded/example_conan_flow) available on git that is usually used as a template on the creation of new packages.


#### What to do in `conanfile.py`
To make the workflow work, a functional `conanfile.py` is obviously needed. As an example I will explain a little about the recipe used to generate the [`gtest`](https://github.com/bincrafters/conan-gtest/blob/stable/1.8.1/conanfile.py) package. Even though it contains some details related to the `CMake`, that we are not exploring here, and contains all the needed methods for a functional `conanfile.py` it is not a complex recipe .

The `gtest` package works with a `CMakeLists` created as a [_wrapper_](https://github.com/bincrafters/conan-gtest/blob/stable/1.8.1/CMakeLists.txt) that adds the original `CMakeLists` on the library as a `subfolder`. This way it is possible to make all the necessary setting based on the main project without changing anything from the `CMakeLists.txt` of the library - for that reason there are the settings files `FindGTest.cmake.in` and `FindGMock.cmake.in` that will be ignored for now.

On the inicial part of the class `GtestConan` we have the basic package informations: `name`, `version`, `description`, etc, that will be used to describe the package. 
```
class GTestConan(ConanFile):
    name = "gtest"
    version = "1.8.1"
    description = "Google's C++ test framework"
    url = "http://github.com/bincrafters/conan-gtest"
    homepage = "https://github.com/google/googletest"
    ...
```
A little further below we have `export` and `export_sources`, both relate to files packed along with the recipe that will be used or during the build - such as the files `FindGTest.cmake.in` and `FindGMock.cmake.in` - or during the project instalation. The difference between them is that the `export` wil always be saved in cache when a package is installed while the `export_sources` will be exported only when the package compilation is necessary - as usually there is already a binary ready they won't be needed that much. 

And after that we have the [needed informations](https://docs.conan.io/en/latest/reference/conanfile/attributes.html) for the library build/link, such as `generators`, `settings`, `options` and `default_options`. In short words:
 * the `generators` as seen before, are the needed generators for the package build, in this case the `cmake` works because it is the same used by the `gtest`;
 * in `settings` we have the attributes that will be taken in consideration to differenciate one binary from another, that is, in this case we have the operational system (`os`), architecture (`arch`), compiler (`compiler`) or the build type (`build_type`) - that stands for release, debug and etc. If any of these attributes changes, a new binary must be generated.
 * the `options` stand for available options for the present library build. The `gtest` provides options such as `build_gmock`, if you don't need to use the `gmock` all you have to do is set the option as `false`. Just like the `settings`, changing any option a different binary will be generated/recovered;
 * the `default_options`define the initial values of each option. Using the `gmock` example, it is always build by default.
```
    exports = ["LICENSE.md"]
    exports_sources = ["CMakeLists.txt", "FindGTest.cmake.in", "FindGMock.cmake.in"]
    generators = "cmake"
    settings = "os", "arch", "compiler", "build_type"
    options = {"shared": [True, False], "build_gmock": [True, False], "fPIC": [True, False], "no_main": [True, False], "debug_postfix": "ANY"}
    default_options = {"shared": False, "build_gmock": True, "fPIC": True, "no_main": False, "debug_postfix": 'd'}
    _source_subfolder = "source_subfolder"
 ```
 Next we have the [methods](https://docs.conan.io/en/latest/reference/conanfile/methods.html). Here is define how the library code will be recovered, how the enviroment will be configured and how the binary will generated. As example, there are some important methods used for the `gtest` below:
 ```
    ...
    def config_options(self):
        if self.settings.os == "Windows":
            del self.options.fPIC
        if self.settings.build_type != "Debug":
            del self.options.debug_postfix
            
    def configure(self):
        if self.settings.os == "Windows":
            if self.settings.compiler == "Visual Studio" and Version(self.settings.compiler.version.value) <= "12":
                raise ConanInvalidConfiguration("Google Test {} does not support Visual Studio <= 12".format(self.version))

    def source(self):
        tools.get("{0}/archive/release-{1}.tar.gz".format(self.homepage, self.version))
        extracted_dir = "googletest-release-" + self.version
        os.rename(extracted_dir, self._source_subfolder)

    def build(self):
        cmake = CMake(self)
        ...
        cmake.definitions["BUILD_GMOCK"] = self.options.build_gmock
        cmake.definitions["GTEST_NO_MAIN"] = self.options.no_main
        if self.settings.os == "Windows" and self.settings.compiler == "gcc":
            cmake.definitions["gtest_disable_pthreads"] = True
        cmake.configure()
        cmake.build()
        
     def package(self):
        self.copy("LICENSE", dst="licenses", src=self._source_subfolder)

        self.copy("FindGTest.cmake", dst=".", src=".")
        gtest_include_dir = os.path.join(self._source_subfolder, "googletest", "include")
        self.copy(pattern="*.h", dst="include", src=gtest_include_dir, keep_path=True)
        ...
```
* `config_options`: Must be used with caution for they are called **before** the option value attribution. It is usually used to make the compatibility verification between `settings` and `options`. On `gtest`'s case, for example, the option `fPIC` is removed if windows is the plataform; and the option `debug_postix` is also removed if the `build)type` is `debug`. This means that if the user wants to install the `gtest` with the `fPIC` option on the `windows` an error will be returned informing that the present option is not available. If it is in `debug` the same happens with `debug_postfix`.
 
 * `configure`: The configure is very similar to the `config_options`, but it is called right after the options are attributed. This way it is possible to verify the `options` and `settings` and make any changes or cheking before proceding to the build. In the `gtest` example, if the builder is `Visual Studio` version 12 or above, an error is thrown for there is no support for those versions.
 
 * `source`: Used to recover the project code you wish to pack. It doesn't make much sense to pack the code alongside the conan recipe, since there is no version control. The code can be recovered by using `git` (conan has a git's specific [tool](https://docs.conan.io/en/latest/reference/tools.html#tools-git)) or by downloading a compressed file, as made in `gtest`. It's worth saying that conan already helps a lot in this aspect by using the [`tool.get()`](https://docs.conan.io/en/latest/reference/tools.html#tools-get), it downloads, validate (if you wish to), decompress and delete the temporary compressed file, all in one single call.
 
 * `build`: Build the code recovered by the `source()`, or the code already available in the package if existant. In this method is where the generator is configured with the building flags. In `gtest`'s case, the `cmake` is used. Conan provides a [tool](https://docs.conan.io/en/latest/reference/build_helpers/cmake.html) to make the manipulation of the generator easier, all you have to do is add the definitions such as `BUILD_GMOCK`, `GTEST_NO_MAIN` and other options related to the library.
 
 * `package`: Lastly, there is the method that will pack and make everything available for use when asked for conan to install the present package. This method is where the `build` directory copys are made to the package directory. Most of the copy is made using the [`self.copy()`](https://docs.conan.io/en/latest/reference/conanfile/methods.html#package), but there is also the option to use the [`cmake`'s install](https://docs.conan.io/en/latest/howtos/cmake_install.html#reuse-cmake-install) if it's already done. The `gtest` package has a long routine in `package` due to the possibility of different settings - if you don't build the `gmock`, for example, there is no need to copy it.
  
 Based on this example, we can conclude that even for a fairly complex package such as the `gtest`, it is possible to create a package with less than 100 lines. Maybe some of you may think 100 lines is a lot, but taking into account that 20 lines are destined to describe the library and its options; and it took 40 lines only to accomplish the files copy and instalation (in `package`) due to the diferent build options (disabling `gmock` or the `main`) available by the library, the rest is a farily simple process. Instead of making a similiar process on your terminal during the setting of your project, you just need to do it once with Conan and it will be available anytime you need to use the library again. This works if you know the library that you wish to pack and it is not as giant ans the `Qt` or `boost`'s library, of course.

### Conclusion
___
I've been studying Conan for two weeks, verifying how to use and how to make the packages and how things develop when we attempt to combine both process. During these studies I've noticed that most of the problems happen when the libraries are not ready to be packed.

Some libraries are created with several different options, some options may or may not rely on other libraries, some of them claim to be cross compile but in the end doesn't have 100% support... Anyways, the more variables exposed to the build of a library, the bigger the complexity to pack them and this makes the process for those that create the packages worst, be it for Conan or any other package manager. So, before trying to pack your library, it is recommended that it is ready for the process, regardless of the package manager you are going to use. There are a few talks[7] and papers[8] that helped me to understand a lot of that process. A lot of people don't care or just doesn't know that a good `CMakeLists.txt`can make a great difference when making your library available for third parties, thinking that if the code works, it doesn't matter what others have to go through trying to build your library. Don't be that person.

Still, by being very well used, be it by Conan's official repository or active communitys such as `bincrafters`, lots of improvements are being made to make the resolution to theses problems easier and the creation and usage of the packages faster. Last year, Conan improved some commands to make the creation flow of packages easier after feedbacks from bincrafters[9], who noticed it could be improved. Acts like that show that the software is concerned with the community reception and is open to suggestions and improvements so that the software molds itself according to the needs of the community as a unity.

Conan turned out to be very complete and ready for many different settings, having a learning cost a little above the expcted due to it's complexity related to the package creation. The community is incredibly welcoming and the documentation is captivating. Sometimes, even if you don't have all the details you need the documentation helps you on the transparecy of the process accomplished, making you solve the easier problems by better understanding of the software. In some cases, in simpler package managers, I had greater difficulties in usage because I couldn't find available documentation or the lack of comments about a specific problems in the community, even realizing that the usage of the manager is easier.

Of course there are some negative sides and complaints from users. The `Python`'s dependency is one of the greatest setbacks exposed by the community, even though many don't see that as a problem, not everybody likes to install `Python`**only** to realize the build of a C++ project. Another problem I stumbled upon is the lack of deep knowledge on `Cmake`, especially the modern functions, gave me a little headache. This is not a Conan problem, obviously, but it made me realize that the usage of the package manager (regardless of the manager) isn't just a "plug-n-play", a larger knowledge is needed in the dependancys organizations part, at least to organize the setup. Once ready, everything gets easier.

>[7]: [Webinar: Introduction to C/C++ Package Management with Conan](https://www.youtube.com/watch?v=xBLjXdyh3zs), [Don't package your libraries, write packagable libraries!](https://www.youtube.com/watch?v=sBP17HQAQjk), [Effective CMake](https://www.youtube.com/watch?v=bsXLMQ6WgIk)

>[8]: [The Pitchfork Layout](https://api.csswg.org/bikeshed/?force=1&url=https://raw.githubusercontent.com/vector-of-bool/pitchfork/develop/data/spec.bs), [It's time to do CMake right](https://pabloariasal.github.io/2018/02/19/its-time-to-do-cmake-right/)

>[9]: [Updated Conan Package Flow](https://bincrafters.github.io/2018/02/27/Updated-Conan-Package-Flow-1.1/)
