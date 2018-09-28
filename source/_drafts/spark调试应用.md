# spark调试应用

在 YARN 术语中，executors（执行器）和 application masters 在 “containers（容器）” 中运行时。 YARN 有两种模式用于在应用程序完成后处理容器日志（container logs）。如果启用日志聚合（aggregation）（使用 `yarn.log-aggregation-enable` 配置），容器日志（container logs）将复制到 HDFS 并在本地计算机上删除。可以使用 `yarn logs` 命令从集群中的任何位置查看这些日志。

```
yarn logs -applicationId <app ID>
```

将打印来自给定的应用程序的所有容器（containers）的所有的日志文件的内容。你还可以使用 HDFS shell 或者 API 直接在 HDFS 中查看容器日志文件（container log files）。可以通过查看您的 YARN 配置（`yarn.nodemanager.remote-app-log-dir` 和 `yarn.nodemanager.remote-app-log-dir-suffix`）找到它们所在的目录。日志还可以在 Spark Web UI 的 “执行程序（Executors）”选项卡下找到。您需要同时运行 Spark 历史记录服务器（Spark history server） 和 MapReduce 历史记录服务器（MapReduce history server），并在 `yarn-site.xm`l 文件中正确配置 `yarn.log.server.url`。Spark 历史记录服务器 UI 上的日志将重定向您到 MapReduce 历史记录服务器以显示聚合日志（aggregated logs）。

当未启用日志聚合时，日志将在每台计算机上的本地保留在 `YARN_APP_LOGS_DIR 目录下`，通常配置为 `/tmp/logs` 或者 `$HADOOP_HOME/logs/userlogs`，具体取决于 Hadoop 版本和安装。查看容器（container）的日志需要转到包含它们的主机并在此目录中查看它们。子目录根据应用程序 ID （application ID）和 容器 ID （container ID）组织日志文件。日志还可以在 Spark Web UI 的 “执行程序（Executors）”选项卡下找到，并且不需要运行 MapReduce history server。

要查看每个 container（容器）的启动环境，请将 `yarn.nodemanager.delete.debug-delay-sec` 增加到一个较大的值（例如 `36000`），然后通过 `yarn.nodemanager.local-dirs` 访问应用程序缓存，在容器启动的节点上。此目录包含启动脚本（launch script）， JARs ，和用于启动每个容器的所有的环境变量。这个过程对于调试 classpath 问题特别有用。（请注意，启用此功能需要集群设置的管理员权限并且还需要重新启动所有的 node managers，因此这不适用于托管集群）。

要为 application master 或者 executors 使用自定义的 log4j 配置，请选择以下选项:

- 使用 `spark-submit` 上传一个自定义的 `log4j.properties` ，通过将 spark-submit 添加到要与应用程序一起上传的文件的 –files 列表中。
- add `-Dlog4j.configuration=<location of configuration file>` to `spark.driver.extraJavaOptions` (for the driver) or `spark.executor.extraJavaOptions` (for executors). Note that if using a file, the `file:` protocol should be explicitly provided, and the file needs to exist locally on all the nodes.
- 添加 `-Dlog4j.configuration=<配置文件的位置>` 到 `spark.driver.extraJavaOptions`（对于驱动程序）或者 containers （对于执行者）。请注意，如果使用文件，文件: 协议（protocol ）应该被显式提供，并且该文件需要在所有节点的本地存在。
- 更新 `$SPARK_CONF_DIR/log4j.properties` 文件，并且它将与其他配置一起自动上传。请注意，如果指定了多个选项，其他 2 个选项的优先级高于此选项。

请注意，对于第一个选项，executors 和 application master 将共享相同的 log4j 配置，这当它们在同一个节点上运行的时候，可能会导致问题（例如，试图写入相同的日志文件）。

如果你需要引用正确的位置将日志文件放在 YARN 中，以便 YARN 可以正确显示和聚合它们，请在您的 `log4j.properties` 中使用 `spark.yarn.app.container.log.dir`。例如，`log4j.appender.file_appender.File=${spark.yarn.app.container.log.dir}/spark.log`。对于流应用程序（streaming applications），配置 `RollingFileAppender` 并将文件位置设置为 YARN 的日志目录将避免由于大型日志文件导致的磁盘溢出，并且可以使用 YARN 的日志实用程序（YARN’s log utility）访问日志。

To use a custom metrics.properties for the application master and executors, update the `$SPARK_CONF_DIR/metrics.properties` file. It will automatically be uploaded with other configurations, so you don’t need to specify it manually with `--files`.

要为 application master 和 executors 使用一个自定义的 metrics.properties，请更新 `$SPARK_CONF_DIR/metrics.properties` 文件。它将自动与其他配置一起上传，因此您不需要使用 `--files` 手动指定它。



## 链接

http://spark.apachecn.org/docs/cn/2.2.0/running-on-yarn.html