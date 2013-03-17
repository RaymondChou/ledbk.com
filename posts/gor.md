---
title: gor
date: '2013-03-17'
description:
categories:
---

	package main

	import (
	    "fmt"
    "net/http"
	)
	
	func main() {
	    resp, err := http.Get("http://mirrors.ustc.edu.cn/opensuse/distribution/12.3/iso/openSUSE-12.3-GNOME-Live-i686.iso")
	    if err != nil {
	        panic(err)
	    }
	    fmt.Println("Resp code", resp.StatusCode)
	    resp.Body.Close() // 注意,这里并不读取resp.Body, 而resp.Body有大概700mb未读取
	}