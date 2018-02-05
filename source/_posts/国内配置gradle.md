---
title: 国内配置gradle
date: 2018-01-22 22:14:45
categories: gradle
tags: gradle配置
---
不知zf是怎么想的，为何要把Google developer给q了，导致Android开发很痛苦，如果不fq，那么gradle里面库就没法下载，不过感谢阿里云，在国内搭建了一个国内的gradle仓库。下面就配置一下国内的gradle仓库
### 1. 单个项目配置
用这种方式仅对单个项目生效，不会影响其他的项目
``` xml
buildscript {
    repositories {
        maven { url 'http://maven.aliyun.com/nexus/content/groups/public/' }
                maven{ url 'http://maven.aliyun.com/nexus/content/repositories/jcenter'}
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:2.2.3'

        // NOTE: Do not place your application dependencies here; they belong
        // in the individual module build.gradle files
    }        
}

allprojects {
    repositories {
        maven { url 'http://maven.aliyun.com/nexus/content/groups/public/' }
        maven{ url 'http://maven.aliyun.com/nexus/content/repositories/jcenter'}
    }
}
```

### 2. 配置对所有项目生效
在USER_HOME/.gradle/下创建init.gradle文件，填入以下内容：
``` xml
allprojects{
    repositories {
        def ALIYUN_REPOSITORY_URL = 'http://maven.aliyun.com/nexus/content/groups/public'
        def ALIYUN_JCENTER_URL = 'http://maven.aliyun.com/nexus/content/repositories/jcenter'
        all { ArtifactRepository repo ->
            if(repo instanceof MavenArtifactRepository){
                def url = repo.url.toString()
                if (url.startsWith('https://repo1.maven.org/maven2')) {
                    project.logger.lifecycle "Repository ${repo.url} replaced by $ALIYUN_REPOSITORY_URL."
                    remove repo
                }
                if (url.startsWith('https://jcenter.bintray.com/')) {
                    project.logger.lifecycle "Repository ${repo.url} replaced by $ALIYUN_JCENTER_URL."
                    remove repo
                }
            }
        }
        maven {
                url ALIYUN_REPOSITORY_URL
            url ALIYUN_JCENTER_URL
        }
    }
}
```
这样新建项目时用的gradle仓库也是阿里云的。