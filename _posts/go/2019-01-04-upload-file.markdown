---
layout: post
title: "upload file"
date: 2019-01-04 22:05:00 +0800
categories: go
---
##### 通过表单从浏览器上传文件到服务器
**upload.gtpl**
``` html
<html>
<head>
    <title>上传文件</title>
</head>
<body>
<form enctype="multipart/form-data" action="/upload" method="post">
  <input type="file" name="uploadfile" />
  <input type="hidden" name="token" value="{{.}}"/>
  <input type="submit" value="upload" />
</form>
</body>
</html>
```

**main.go**
``` go
package main

import (
	"crypto/md5"
	"fmt"
	"html/template"
	"io"
	"log"
	"net/http"
	"os"
	"strconv"
	"time"
)

func upload(w http.ResponseWriter, r *http.Request) {
	fmt.Println("method:", r.Method)
	if r.Method == "GET" {
		crutime := time.Now().Unix()
		h := md5.New()
		io.WriteString(h, strconv.FormatInt(crutime, 10))
		token := fmt.Sprintf("%x", h.Sum(nil))

		t, _ := template.ParseFiles("upload.gtpl")
		t.Execute(w, token)
	} else {
		r.ParseMultipartForm(32 << 20)
		file, handler, err := r.FormFile("uploadfile")
		if err != nil {
			fmt.Println(err)
			return
		}
		defer file.Close()

		fmt.Fprintf(w, "%v", handler.Header)
		f, err := os.OpenFile(handler.Filename, os.O_WRONLY|os.O_CREATE, 0666)
		if err != nil {
			fmt.Println(err)
			return
		}
		defer f.Close()
		io.Copy(f, file)
	}
}

func login(w http.ResponseWriter, r *http.Request) {
	fmt.Println("method:", r.Method) //获取请求的方法
	if r.Method == "GET" {
		t, _ := template.ParseFiles("login.gtpl")
		log.Println(t.Execute(w, nil))
	} else {
		fmt.Println("username:", r.Form["username"])
		fmt.Println("password:", r.Form["password"])
	}
}

func main() {
	http.HandleFunc("/login", login)
	http.HandleFunc("/upload", upload)

	log.Println("listening...")

	http.ListenAndServe(":8890", nil)
}
```
处理文件上传我们需要调用 r.ParseMultipartForm，里面的参数表示 maxMemory，调用 ParseMultipartForm 之后，上传的文件存储在 maxMemory 大小的内存里面，如果文件大小超过了maxMemory，那么剩下的部分将存储在系统的临时文件中。我们可以通过 r.FormFile获取上面的文件句柄，然后实例中使用了 io.Copy 来存储文件<br>
**上传文件步骤分析**
1. 表单中增加 enctype="multipart/form-data"
2. 服务端调用 r.ParseMultipartForm,把上传的文件存储在内存和临时文件中
3. 使用 r.FormFile 获取文件句柄，然后对文件进行存储等处理

##### 客户端上传文件
``` go
package main 

import (
	"io"
	"os"
	"fmt" 	
	"bytes"
	"net/http"
	"io/ioutil"
	"mime/multipart"
)

func postFile(file_name string, target_url string) error {
	bodyBuf := &bytes.Buffer{}
	bodyWriter := multipart.NewWriter(bodyBuf)//create a writer object

	//create a form
	fileWriter, err := bodyWriter.CreateFormFile("uploadfile", file_name)
	if err != nil {
		fmt.Println("error writing to buffer")
		return err
	}

	fh, err := os.Open(file_name)
	if err != nil {
		fmt.Println("error Opening file")
		return err
	}
	defer fh.Close()

	_, err = io.Copy(fileWriter, fh)
	if err != nil {
		return err
	}

	contentType := bodyWriter.FormDataContentType()
	bodyWriter.Close()

	resp, err := http.Post(target_url, contentType, bodyBuf)
	if err != nil {
		return err
	}
	defer resp.Body.Close()

	resp_body, err := ioutil.ReadAll(resp.Body)
	if err != nil {
		return err
	}

	fmt.Println(resp.Status)
	fmt.Println(string(resp_body))

	return nil
}

func main() {
	target_url := "http://localhost:8890/upload"
	file_name := "test.pdf"
	err := postFile(file_name, target_url)

	if err != nil {
		fmt.Println("upload filed!!!")
	}
}
```
客户端通过 multipart.Write 把文件的文本流写入一个缓存中，然后调用 http 的 Post 方法把缓存传到服务器
