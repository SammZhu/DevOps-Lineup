# 将传统Spring Boot和MySQL应用迁移部署到OpenShift

本文介绍如何迁移现有的 [Spring Boot ](http://projects.spring.io/spring-boot/)加MySQL独立项目，将其部署在 [Red Hat OpenShift](https://developers.redhat.com/products/openshift/overview/)上。在此过程中，我们将创建可以部署到大多数容器/云平台的Docker镜像。我将讨论创建Dockerfile，将容器映像推送到OpenShift注册表，最后创建运行的Pod，并部署Spring Boot应用程序。

开发人员在本地机器上使用OpenShift进行开发和测试，通常推荐使用  [*红帽容器开发套件（CDK）*](https://developers.redhat.com/products/cdk/overview/)，它提供了基于minishift的可在在Red Hat Enterprise Linux VM（红帽的虚拟机产品）中运行的单节点OpenShift集群。这些产品都是开源产品，您可以在Windows、macOS或Red Hat Enterprise Linux之上运行CDK。本次测试使用了Red Hat Enterprise Linux Workstation 7.3版，它同样也适用于macOS。

如何创建Spring Boot应用程序，可以将  [这篇文章](https://spring.io/guides/gs/accessing-data-mysql/)  作为指南。同时使用现有的[openshift/mysql-55-centos7](https://docs.openshift.com/container-platform/3.11/using_images/db_images/mysql.html)镜像将MySQL部署到OpenShift。

在本文将在本地构建容器映像，因此您需要能够使用Maven工具，并可以从[*github仓库*](https://github.com/1984shekhar/POC/tree/master/mysql-springboot-docker-openshift)下载引用到的原代码进行编译。此示例使用*com.sample.app.MainController.java*类公开REST服务。

在代码库的`src/main/docker-files/`路径下可以找到一个Dockerfile名为*dockerfile_springboot_mysql*。这个文件负责创建一个基于 *java8 Docker*镜像的*Spring Boot*应用Docker镜像。这可以用于测试，但对于生产部署，出于对您的生产系统安全性和业务连续性的保证，还是需要使用基于Red Hat Enterprise Linux的镜像。

第一步，构建应用程序：

1. 使用 `mvn clean install` 来构建项目。
2. 将生成的jar复制到目标文件夹中  `src/main/docker-files`。创建docker镜像时，需要在这个位置找到应用程序jar包。
3. 在`src/main/resources/application.properties`中设置数据库用户名、密码和URL地址 。注意：对于OpenShift，建议将这些参数作为环境变量传递到容器中。

第二步，启动运行环境。如果已经存在OpenShift集群可以忽略此步骤。

1. 启动CDK VM以运行本地OpenShift集群，使用*minishift start启动* CDK VM  ：

   ```shell
   $ minishift start 
   ```

2. 为Docker和oc CLI设置本地环境：

   ```shell
   $ eval $(minishift oc-env) 
   $ eval $(minishift docker-env) 
   ```

   注意：上述eval命令在Windows上不起作用。有关更多信息，请参阅CDK文档。

3. 使用*开发者*帐户登录OpenShift和Docker  ：

   ```shell
   $ oc login
   $ docker login -u developer -p $(oc whoami -t) $(minishift openshift registry)
   ```

第三步，构建容器镜像

1. 将目录更改为工程内的`src/main/docker-files`位置。然后，执行以下命令以构建 应用*容器镜像*。注意：docker build命令结束时需要句点（.）来指示当前目录：

```shell
$ docker build -t springboot_mysql -f ./dockerfile_springboot_mysql .
```

​	使用以下命令查看已创建的容器镜像：

```shell
$ docker images
```

2. 将标记springboot_mysql添加到镜像，并将其推送到OpenShift注册表：

```shell
$ docker tag springboot_mysql $(minishift openshift registry)/myproject/springboot_mysql
$ docker push $(minishift openshift registry)/myproject/springboot_mysql
```

非minishift环境使用如下命令：

```shell
$ oc login -u admin
$ docker login -u admin -p $(oc whoami -t) $(oc registry info)
$ docker tag springboot_mysql $(oc registry info)/myproject/springboot_mysql
$ docker push $(oc registry info)/myproject/springboot_mysql
```

3. 接下来，拉出OpenShift  *MySQL*映像，并将其创建为OpenShift应用程序，该应用程序将初始化并运行它。有关更多信息，请参阅 [文档](https://docs.openshift.com/container-platform/3.11/using_images/db_images/mysql.html)：

```shell
$ docker pull openshift/mysql-55-centos7
$ oc new-app -e MYSQL_USER=root -e MYSQL_PASSWORD=root -e MYSQL_DATABASE=test openshift/mysql-55-centos7
```

4. 稍后，运行MySQL的pod会准备就绪。您可以使用以下方式检查状态`oc get pods`：

```shell
$ oc get pods
NAME READY STATUS RESTARTS AGE 
mysql-56-centos7-1-nvth9 1/1 Running 0 3m
```

5. 接下来，ssh到  *mysql* pod并创建一个具有完全权限的MySQL root用户：

```shell
$ oc rsh mysql-56-centos7-1-nvth9
sh-4.2$ mysql -u root
CREATE USER 'root'@'%' IDENTIFIED BY 'root';
Query OK, 0 rows affected (0.00 sec)
 
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' WITH GRANT OPTION;
Query OK, 0 rows affected (0.00 sec)
 
FLUSH PRIVILEGES;
Query OK, 0 rows affected (0.00 sec)
 
exit
```

6. 最后，使用*imagestream* springboot_mysql 初始化Spring Boot应用程序：

```shell
$ oc get svc
NAME CLUSTER-IP EXTERNAL-IP PORT(S) AGE
mysql-56-centos7 172.30.145.88 none 3306/TCP 8m
 
$ oc new-app -e spring_datasource_url=jdbc:mysql://172.30.145.88:3306/test springboot_mysql
$ oc get pods
NAME READY STATUS RESTARTS AGE
mysql-56-centos7-1-nvth9 1/1 Running 0 12m
springbootmysql-1-5ngv4 1/1 Running 0 9m
```

7. 检查pod日志：

```shell
$ oc logs -f springbootmysql-1-5ngv4
```

8. 接下来，将服务公开为路由：

```shell
$ oc get svc
NAME CLUSTER-IP EXTERNAL-IP PORT(S) AGE
mysql-56-centos7 172.30.242.225 none 3306/TCP 14m
springbootmysql 172.30.207.116 none 8080/TCP 1m
 
$ oc expose svc springbootmysql
route "springbootmysql" exposed
 
$ oc get route
NAME HOST/PORT PATH SERVICES PORT TERMINATION WILDCARD
springbootmysql springbootmysql-myproject.192.168.42.182.nip.io springbootmysql 8080-tcp None
```

9. 使用`curl`测试应用程序。您应该看到数据库表中所有条目的列表：

```shell
$ curl -v http://springbootmysql-myproject.192.168.42.182.nip.io/demo/all
```

10. 接下来，使用`curl`在db中创建一个条目：

```shell
$ curl http://springbootmysql-myproject.192.168.42.182.nip.io/demo/add?name=SpringBootMysqlTest
Saved
```

11. 查看数据库中更新的条目列表：

```shell
$ curl http://springbootmysql-myproject.192.168.42.182.nip.io/demo/all
 
[{"name":"UBUNTU 17.10 LTS","lastaudit":1502409600000,"id":1},{"name":"RHEL 7","lastaudit":1500595200000,"id":2},{"name":"Solaris 11","lastaudit":1502582400000,"id":3},{"name":"SpringBootTest","lastaudit":1519603200000,"id":4},{"name":"SpringBootMysqlTest","lastaudit":1519603200000,"id":5}
```



搞定！

希望本文对您将现有的*spring-boot*应用程序迁移到OpenShift 有所帮助。再次提醒注意，为了您的业务顺利开展，在生产环境中务必使用Red Hat支持的安全和可维护镜像。本文档仅用于开发目的，它应该可以帮助您创建在容器中运行的*spring-boot*应用程序，以及如何在OpenShift中为*spring-boot*设置*MySQL*连接。



感谢原文作者：Chandra Shekhar Pandey，我在文章基础上进行了维持原意的翻译和细节上的调整。

原文参考地址：https://developers.redhat.com/blog/2018/03/27/spring-boot-mysql-openshift/
