# AndroidLibPacker
Pack All  Android Library dependency in One jar

## 问题起源

在Android Studio中开发Library A，不可避免要引用到别的Library，可能是线上的比如rxjava或者是本地的project。

当其他APP或者Library要引用这个Library A时，有几种方法：

  1. 将Library A打包上传到jcenter等，在Android Studio中直接引用对应的group name version即可。

  2. 将Library A打成aar包，在Android Studio中直接Import然后引用。

  3. 将Library A打包成jar包，其他工程直接引用jar包。

上述几种方法对应的问题：

  1. 上传到jcenter是比较推荐的方法，需要将依赖的本地project也打包上传到jcenter，其次jcenter是公开的，隐私不能保证。但是这个可以通过搭建私有nexus repo manager避免。

  2. 打成aar包，并不会将依赖打包，无论Lib A依赖的线上还是本地project，都需要在新工程中重新声明。

  3. 打出Lib A的jar不难，关键需要将Lib A的所有本地或者线上依赖全部打入jar包，搜索很久还没发现gradle有现成的task可以完成，这正是本文需要解决的问题。


## 解决方案

虽然gradle没有将所有依赖打入jar的task，但是我们在gradle生成apk时却会将APP的所有依赖打入apk，所以假如我们将apk解包，再把dex转成jar包就可以了。

gradle并没有dex转成jar的task，所以需要依赖dexpatcher，它包含了Dex2Jar的task，可以让我们把这件事完全自动化。

## 工程说明

JarPacker中包含三个工程：

    Root project '__LibraryPacker'
        +--- Project ':packer'
    Root project 'Example'
        +--- Project ':libA'
        +--- Project ':libB'
        +--- build.gradle

libA和libB假设是我们正常开发的Library Project，我们可以运行

     ./gradlew packLibB

就会在Example下output下生成libA.jar，libB.jar会包含生命的依赖的libA和rxjava，rxandroid。

## 如何使用

1.在需要打包的工程下加入如下task声明，修改其中的targetLibName，targetLibConfig和task名称，buildFile的目录修改为__LibraryPacker对应的目录:

       // 自动提取工程的所有依赖项
       def resolveDependencies(String moduleName) {
           def dependencies = [:]
           def dependencyPattern = ~/ project\('(.*?)'\)\s*/
           def matcher = project(moduleName).buildFile.text =~ dependencyPattern
           while(matcher.find()){
               dependencies.put(matcher.group(1), project(matcher.group(1)).projectDir)
           }
           return dependencies
       }
       
       task packLibB(type:GradleBuild) {
           buildFile = '../__LibraryPacker/build.gradle'
           tasks = ['clean', 'packLibJar']
       
           //指定需要pack的library工程名
           def targetLibName = ':libB'
           //指定需要pack的library的variant，可选（默认default)
           def targetLibConfig = 'default'
           startParameter.setProjectProperties(['libType'        : 'project',
                                                'libName'        : targetLibName,
                                                'libPath'        : project(targetLibName).projectDir,
                                                'libConfig'      : targetLibConfig,
                                                'libDependencies': resolveDependencies(targetLibName),
                                                'libArchiveName' : "${project(targetLibName).name}.jar",
                                                'libOutputDir'   : file('output/libs')])
       }

2. 运行如下命令：

		./gradlew packLibB

	packLibB对应build.gradle中声明的task名称

3. 在当前工程的output文件夹下就生成对应的jar包。

