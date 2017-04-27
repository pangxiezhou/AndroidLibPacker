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

    Root project 'JarPacker'
        +--- Project ':libA'
        +--- Project ':libB'
        \--- Project ':packer'
        
libA和libB假设是我们正常开发的Library Project，我们可以运行

    gradlew.bat :packer:packJar -PtargetLib=libB
    
就会在packer下release_jar下生成libB.jar，libB.jar会包含生命的依赖的libA和rxjava，rxandroid。

## 如何使用

1. 将packer和dexpatcher-gradle-tools放到需要生成的project下， setting.gradle生命加入packer:

        include ':packer'

2. 在根目录下build.gradle中buildscript加入dexpatcher依赖：
        
        buildscript {
            repositories {
                jcenter()
        
                maven {
                    url "https://plugins.gradle.org/m2/"
                }
            }
            dependencies {
                classpath 'com.android.tools.build:gradle:2.3.1'
                classpath "com.github.lanchon.dexpatcher:dexpatcher-gradle:0.4.2"
            }
        }


3. 运行如下命令：

        gradlew.bat :packer:packJar -PtargetLib=xxx
    
    xxx代表需要生成的Library Project名称。

4. 在packer的release_jar文件夹下就能看到打包所有依赖生成的jar了。

