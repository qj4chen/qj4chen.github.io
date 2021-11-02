---
layout: post
title: 使用 Scala & Spring Boot 搭建小🐴图床站
categories:
  - Java
  - Scala
  - Spring Boot
  - Aliyun
  - 图床
---

> 本文介绍了使用 Spring Boot 搭建的一个小型图床站的项目。这个项目的亮点在于 —— 没有亮点... 但是有很多独特之处，比如采用了 Scala + Java 的混合设计，Scala 在 Spring IOC 中可以很好的适应 AutoWired 特性，简直水乳交融 —— 这是我见过的唯一的两门能够如此紧密结合的语言，因此我决然的抛弃了 Python，起码在 Scala 能够调用的Java 百万类库的能力范围内。

# 前端

前端项目发布在 Github 上：[项目地址](https://github.com/corkine/cmBed_Vue)

技术组成如下所示：

![cm_image 2019-02-11 21.03.50](/media/cm_image%202019-02-11%2021.03.50.png)

72%的 JavaScript，26% 的 Vue，1.1% 的 HTML。具体而言，前端使用了 Vue View FrameWork，Element UI Library（饿了么），以及一个叫做 Clipboard 的用于自动粘贴内容到剪贴板的小工具。

前端的界面如下：

![cm_image 2019-02-11 19.58.50](/media/cm_image%202019-02-11%2019.58.50.png)
主要功能包括拖拽上传，通知提醒，点击粘贴到剪贴板，进度条展示上传进度。这个前端看着简单，但是实际很耗费时间，比如我之前一直用 Bootstrap，第一次接触 Element，发现这货大体堪用，但是小细节显然没有 BS 做得好，但是各种通知真的符合中国人的习性...不愧是饿了么前端团队开源的作品。

前端主要逻辑就是调用 api POST 文件到后端即可。因为是公有读，所以前端也没有加权限限制，不过有大小限制，5MB。

# 后端

后段部分的项目也托管在 Github 上：[项目地址](https://github.com/corkine/cmBed_Spring)

这是项目组成：

![cm_image 2019-02-11 21.08.48](/media/cm_image%202019-02-11%2021.08.48.png)

66% 的 Java，28% 的 Scala，5% 的 HTML。主要技术是 Aliyun OSS SDK 的一个简单应用，从前端接受到的 File，Java 再过一遍，再通过 Scala 上传到阿里云的 OSS 服务器中。

操作 OSS SDK，我是通过 Scala 进行的封装，之后通过 Spring Boot 进行调用。Spring 的各种配置类 —— 没有作死，还是用的 Java 进行配置（当然，**主要原因是 Scala 的智能提示做不到像 Java 那样，毕竟这蛋疼的山路十八弯的语法，还有神奇的绝地武士力场范围 implicit**，即便是 IDEA 这样对 Scala 支持最好的 IDE 也无能为力）。

Scala 用起来感觉确实很好，但是我还是挺遗憾的，没用到什么 match、Option 判断、FP 集合处理、柯里化和控制结构。下次搞个大点的项目试试看。

下面是主要的业务逻辑：

```scala
package com.mazhangjing.cloud.tuchuang.oss

import java.io.{File, PrintWriter, StringWriter}
import java.time.LocalDate
import java.time.format.DateTimeFormatter
import java.util

import com.aliyun.oss.model.OSSObjectSummary
import org.slf4j.LoggerFactory
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.stereotype.Component


@Component
class OSSUtils {

  @Autowired var config:OSSConfig = _
  private val logger = LoggerFactory.getLogger(classOf[OSSUtils])

  import java.util.UUID

  import com.aliyun.oss.model.{CannedAccessControlList, CreateBucketRequest, PutObjectRequest}

  def upload(file: File):String = {

    val client = config.getClientInstance
    val time = LocalDate.now().format(DateTimeFormatter.BASIC_ISO_DATE)
    val filePath = s"$time/${UUID.randomUUID().toString.replace("-","").substring(0,7)}"
    val fileUrl = filePath + s"_${file.getName.replace(" ","")}"

    try {
      logger.info(s"OSS 文件上传开始， File:${file.getName}")

      if (!client.doesBucketExist(config.getBucketName)) {
        client.createBucket(config.getBucketName)
        val request = new CreateBucketRequest(config.getBucketName)
        request.setCannedACL(CannedAccessControlList.PublicRead)
        client.createBucket(request)
      }
      val putRequest = new PutObjectRequest(config.getBucketName, fileUrl, file)
      val putResult = client.putObject(putRequest)
      client.setBucketAcl(config.getBucketName, CannedAccessControlList.PublicRead)
      putResult match {
        case null => logger.info(s"$file - $fileUrl - 文件上传失败 - $putResult")
        case _ =>
          logger.info(s"$file - $fileUrl - 文件上传成功")
          return config.getFileHost + "/" + fileUrl
      }
    } catch {
        case ex : Exception =>
          val w = new StringWriter()
          ex.printStackTrace(new PrintWriter(w))
          logger.info(ex.getMessage + ", Details: \n" + w.toString)
    } finally {
      if (client != null) client.shutdown()
    }
    null
  }

  import com.aliyun.oss.model.GetObjectRequest

  def downloadFile(objectName: String, localFileName: String): Boolean = {
    try {
      val ossClient = config.getClientInstance
      ossClient.getObject(new GetObjectRequest(config.getBucketName, objectName), new File(localFileName))
      ossClient.shutdown()
      return true
    } catch {
      case ex :Exception =>
        logger.info(s"DownLoad File Error: ${ex.getMessage}")
    }
    false
  }

  import com.aliyun.oss.model.ListObjectsRequest

  def listFile(prefix:String = ""): util.List[OSSObjectSummary] = {
    val ossClient = config.getClientInstance
    val listObjectsRequest = new ListObjectsRequest(config.getBucketName)
    if (prefix != "") listObjectsRequest.setPrefix(prefix)
    val listing = ossClient.listObjects(listObjectsRequest)

    val listWithOutFolders = listing.getObjectSummaries
    ossClient.shutdown()
    listWithOutFolders
  }

  def listFolder(prefix:String = ""): util.List[String] = {
    val ossClient = config.getClientInstance
    val listObjectsRequest = new ListObjectsRequest(config.getBucketName)
    if (prefix != "") listObjectsRequest.setPrefix(prefix)
    val listing = ossClient.listObjects(listObjectsRequest)

    val listJustFolders = listing.getCommonPrefixes
    ossClient.shutdown()
    listJustFolders
  }
}
```

对于配置而言，因为没有权限控制，所以不用 Shiro，仅仅是 Web 的配置，包括默认 Servlet、静态资源路径设置、单页程序 Resolver 设置、一个错误处理类的 Resolver 设置，Cros 跨域访问设置，最后简单的几行日志配置，就没了，后端起的很快。

.... 但是，这主要的原因是 Spring Boot 封装的好，比如直接从 Controller 获取到了 File，而根本不用自己处理 IO，各种日志，模板都是自动配置的。

总的来说，除了 Scala 的 Object 伴生对象无法很好的和 Spring 兼容以外，其余部分协作的非常好，Spring 可以直接注入 Bean 属性、字段到 Scala 的类中，这非常方便。

这是我第一次尝试混合 Scala 和 Java 程序，虽然做的不多，但是还是花了整整一天时间，包括文档、部署、前端、后端、仓库、博客。话说回来，这个阿里云的 OSS 只是一个备份，我目前主要的图床部署在七牛云，看七牛什么时候收钱收的太过分后，再转向阿里云。如果什么时候能搞到一个高带宽的服务器，可能我就会自己用文件存储替代 OSS 服务，毕竟自家的才安心。

# 部署

后端仓库直接 `java -jar xxx.jar` 运行即可，在 `resources/application.yml` 中有 OSS 的配置，拿出来，放到 jar 同目录下，修改并覆盖即可。后端仓库包含了前端打包好的 Js 和 HTML，直接使用即可，如果想要修改前端，且熟悉 Vue 的话，见此 [cmBed_Vue](https://github.com/corkine/cmBed_Vue)



