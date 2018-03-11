---
layout: post
title: 使用artifacory部署私有仓库
excerpt: "由于业务规模的扩展，很多app都有拥有一组相同功能或基础服务类代码，每个app维护这样一份代码不仅不易于管理，每次更新的过程更是痛苦，通过部署私有的远程仓库，我们可以统一管理代码，进行版本管理，实现公用组件功能"
categories: [other]
comments: true
---

## 安装artifactory服务
1. 下载对应系统的安装文件，下载地址(https://jfrog.com/open-source/#solutions)
2. 由于我的是linux系统，所以下载了rpm安装包
3. 执行 rpm -ivh 安装包名称.rpm 执行安装（注意，需要有jdk1.8以上版本）
4. 启动artifactory服务，执行命令 systemctl start artifactory.service
5. 检测你artifactory服务是否开启，lsof -i:8081,或者通过浏览器，打开网址 http://ip:8081/artifactory，初次进入会要求你设置管理员账户密码
6. 以上步骤完成，artifactory服务已经在运行

## 配置artifactory服务

* 首先介绍下仓库的分类，在Art中，repo有三种。本地Local型，远程Remote型，以及虚拟型。
* 本地私有仓库：用于内部使用，上传的组件不会向外部进行同步。
* 远程仓库：用于代理及缓存公共仓库，不能向此类型的仓库上传私有组件。
* 虚拟仓库：不是真实在存储上的仓库，它用于组织本地仓库和远程仓库

* 配置私有仓库
1. 点击账户头像，新建一个Local Repository仓库
2. 选择仓库类型，可选gradle，maven等等
3. 填写Repository Key，这里的Repository Key必须为独一无二的，类似于仓库名
4. finish it，仓库创建完成
* 发布公有组件到仓库中
1.在项目全局目录中配置依赖

```ruby
buildscript {
    repositories {
        google()
        jcenter()
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:3.0.1'
        classpath "org.jfrog.buildinfo:build-info-extractor-gradle:4.4.7"

        // NOTE: Do not place your application dependencies here; they belong
        // in the individual module build.gradle files
    }
}

allprojects {
    repositories {
        google()
        jcenter()
        apply plugin: 'com.jfrog.artifactory'
    }
}

task clean(type: Delete) {
    delete rootProject.buildDir
}
```

2.在你需要上传的组件module中新建gradle.properties文件，写入配置信息


```ruby
GROUP_ID = com.intersection.testlibrary
VERSION_NAME = 1.0.0

MAVEN_LOCAL_PATH=http://ip/artifactory/
ARTIFACT_ID=test

USER=你的artifactory账户
PASSWORD=你的artifactory密码
```

3.在组件对应的gradle中引入相应工具

```ruby
apply plugin: 'com.jfrog.artifactory'
apply plugin: 'maven-publish'
```

4.编写gradle上传代码

```ruby
publishing {
    publications {
        aar(MavenPublication) {
            groupId GROUP_ID
            version = VERSION_NAME
            artifactId ARTIFACT_ID

            // Tell maven to prepare the generated "*.aar" file for publishing
            artifact("$buildDir/outputs/aar/${project.getName()}-release.aar")

            pom.withXml {
                def dependencies = asNode().appendNode('dependencies')
                configurations.compile.allDependencies.each{
                    def depGroup = it.group
                    def depName = it.name
                    def depVersion = it.version

                    //如果当前库是以project形式引用的，重写依赖，replace project_name with your project name
                    if(it.name == "project_name") {
                        for (proj in project.parent.subprojects) {
                            if(proj.name == "project_name") {
                                //这些信息是在annotation-lib的build.gradle中定义的
                                depGroup = proj.ext.group
                                depName = proj.ext.artifactId
                                depVersion = proj.ext.version
                            }
                        }
                    }

                    if(it.group != null) {
                        def dependency = dependencies.appendNode('dependency')
                        dependency.appendNode('groupId', depGroup)
                        dependency.appendNode('artifactId', depName)
                        dependency.appendNode('version', depVersion)
                    }
                }
            }
        }
    }
}
artifactory {
    contextUrl = MAVEN_LOCAL_PATH
    publish {
        repository {
            // The Artifactory repository key to publish to
            //对应配置中的Repository Key
            repoKey = 'lib-local'

            username = USER
            password = PASSWORD
        }
        defaults {
            // Tell the Artifactory Plugin which artifacts should be published to Artifactory.
            publications('aar')
            publishArtifacts = true

            // Properties to be attached to the published artifacts.
            properties = ['qa.level': 'basic', 'dev.team': 'core']
            // Publish generated POM files to Artifactory (true by default)
            publishPom = true
        }
    }
}
```

5.在项目中执行,发布代码

```ruby
./gradlew clean assembleRelease
./gradlew artifactoryPublish  
```

6.在项目中依赖maven 仓库，并使用相应组件

```ruby
allprojects {
    repositories {
        maven { url "http://ip:8081/artifactory/lib-local" }
    }
}

dependencies {
    compile 'com.intersection.testlibrary:1.0.0'
}
```
