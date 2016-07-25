---
title: 【Play】热部署是如何工作的？
published: true
date: '27-12-2015 00:00'
visible: true
summary:
    enabled: '1'
    format: long
taxonomy:
    tag:
        - 原创
        - play
content:
    items: '@self.children'
    limit: 5
    order:
        by: date
        dir: desc
    pagination: true
    url_taxonomy_filters: true
---

#### 1.什么是热部署

> 所谓热部署，就是在应用正在运行的时候升级软件，却不需要重新启动应用。对于Java应用程序来说，热部署就是在运行时更新Java类文件。
> http://baike.baidu.com/view/5036687.htm

对于Java应用，有三种常见的实现热部署的方式：

- JPDA: 利用JVM原生的JPDA接口，参见[官方文档](https://docs.oracle.com/javase/8/docs/technotes/guides/jpda/enhancements1.4.html#hotswap)
- Classloader: 通过创建新的Classloader来加载新的Class文件。OSGi就是通过这种方式实现Bundle的动态加载。
- Agent: 通过自定义Java Agent实现Class动态加载。JRebel，hotswapagent使用的就是这种方式。

Play console自带的auto-reload功能正是基于上述第二种方式实现的。

#### 2.Auto-reload机制

Play console是Typesafe封装的一种特殊的的sbt console，主要增加了activator new和activator ui两个命令。其auto-reload功能是以sbt插件（"com.typesafe.play" % "sbt-plugin"）的形式提供的，sbt-plugin通过sbt-run-support类库连接到play开发模式下的启动类（play.core.server.DevServerStart）。每当应用收到请求时，play会通过sbt-plugin检查是否有源文件被修改，如果存在，则调用sbt命令进行编译，然后依次停止老的play应用，创建新的classloader，然后启动新的play应用，在此过程中运行sbt的JVM并没有被重启，只是play应用完成了重启。

#### 3.源码分析

以下分别从sbt-plugin,sbt-run-support和play-server挑选3个核心类对上述流程进行简单梳理。

##### play.sbt.run.PlayRun

定义play run task，通过Reloader传递sbt回调函数引用给DevServerStart。

```scala
[Line 73-93: PlayRun#playRunTask]
lazy val devModeServer = Reloader.startDevMode(
      runHooks.value,
      (javaOptions in Runtime).value,
      dependencyClasspath.value.files,
      dependencyClassLoader.value,
      reloadCompile,										# sbt回调函数引用
      reloaderClassLoader.value,
      assetsClassLoader.value,
      playCommonClassloader.value,
      playMonitoredFiles.value,
      fileWatchService.value,
      (managedClasspath in DocsApplication).value.files,
      playDocsJar.value,
      playDefaultPort.value,
      playDefaultAddress.value,
      baseDirectory.value,
      devSettings.value,
      args,
      runSbtTask,
      (mainClass in (Compile, Keys.run)).value.get
    )
```

##### play.runsupport.Reloader

通过反射启动play应用，将Reloader自身作为参数传入。

```scala
[Line 203-212: Reloader#startDevMode]
val server = {
    val mainClass = applicationLoader.loadClass(mainClassName)
    if (httpPort.isDefined) {
        val mainDev = mainClass.getMethod("mainDevHttpMode", classOf[BuildLink], classOf[BuildDocHandler], classOf[Int], classOf[String])
        mainDev.invoke(null, reloader, buildDocHandler, httpPort.get: java.lang.Integer, httpAddress).asInstanceOf[play.core.server.ServerWithStop]
    } else {
        val mainDev = mainClass.getMethod("mainDevOnlyHttpsMode", classOf[BuildLink], classOf[BuildDocHandler], classOf[Int], classOf[String])
        mainDev.invoke(null, reloader, buildDocHandler, httpsPort.get: java.lang.Integer, httpAddress).asInstanceOf[play.core.server.ServerWithStop]
    }
}
```

##### play.core.server.DevServerStart

从注释可以清楚的看到stop-and-start的重启逻辑。

```scala
[Line 113-180: DevServerStart#mainDev]
val reloaded = buildLink.reload match {
    case NonFatal(t) => Failure(t)
    case cl:
        ClassLoader => Success(Some(cl))
    case null => Success(None)
}

reloaded.flatMap {
    maybeClassLoader =>

        val maybeApplication: Option[Try[Application]] = maybeClassLoader.map {
            projectClassloader =>
                try {

                    if (lastState.isSuccess) {
                        println()
                        println(play.utils.Colors.magenta("--- (RELOAD) ---"))
                        println()
                    }

                    val reloadable = this

                    // First, stop the old application if it exists
                    lastState.foreach(Play.stop)

                    // Create the new environment
                    val environment = Environment(path, projectClassloader, Mode.Dev)
                    val sourceMapper = new SourceMapper {
                        def sourceOf(className: String, line: Option[Int]) = {
                            Option(buildLink.findSource(className, line.map(_.asInstanceOf[java.lang.Integer]).orNull)).flatMap {
                                case Array(file: java.io.File, null) => Some((file, None))
                                case Array(file: java.io.File, line: java.lang.Integer) => Some((file, Some(line)))
                                case _ => None
                            }
                        }
                    }

                    val webCommands = new DefaultWebCommands
                    currentWebCommands = Some(webCommands)

                    val newApplication = Threads.withContextClassLoader(projectClassloader) {
                        val context = ApplicationLoader.createContext(environment, dirAndDevSettings, Some(sourceMapper), webCommands)
                        val loader = ApplicationLoader(context)
                        loader.load(context)
                    }

                    Play.start(newApplication)

                    Success(newApplication)
                } catch {
                    case e:
                        PlayException => {
                            lastState = Failure(e)
                            lastState
                        }
                    case NonFatal(e) => {
                        lastState = Failure(UnexpectedException(unexpected = Some(e)))
                        lastState
                    }
                    case e:
                        LinkageError => {
                            lastState = Failure(UnexpectedException(unexpected = Some(e)))
                            lastState
                        }
                }
        }

    maybeApplication.flatMap(_.toOption).foreach {
        app =>
            lastState = Success(app)
    }

    maybeApplication.getOrElse(lastState)
}
```

#### 4. Gotcha

上述的实现看上去并不复杂，那为什么老牌的Tomcat，JBoss容器却始终没有提供类似的机制呢？原因很简单，Play是stateless的，而其余的不是。

#### 参考

- http://jto.github.io/articles/play_anatomy_part2_sbt/