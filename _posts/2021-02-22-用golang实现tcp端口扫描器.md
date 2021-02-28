---
layout:     post               
title:      用golang实现tcp端口扫描器              
subtitle:   Hello World, Hello Golang 
date:       2021-02-22              
author:     Resulte                      
header-img: img/post-bg-desk.jpg  
catalog: true                       
tags:                               
    - Golang
---

## 用golang实现tcp端口扫描器 

tcp端口扫描器 就是针对某一IP地址，扫描某个范围内的端口是否打开。

### **1. 非并发版本实现**

```go
package main

import (
	"fmt"
	"net"
)

func main() {
	ip := "11.111.10.11"
	startPort := 21
	endPort := 120
	for port := startPort; port < endPort; port++ {
		address := fmt.Sprintf("%s:%d", ip, port)
		conn, err := net.Dial("tcp", address)
		if err != nil {
			fmt.Printf("%s 关闭了\n", address)
			continue
		}
		conn.Close()
		fmt.Printf("%s 打开了\n", address)
	}
}

```

> 注：url，初始扫描端口，结束扫描端口可以自己更改 

非并发下，代码执行时间特别慢。

### **2. 并发版本实现**

并发版本使用goroutine + WaitGroup的形式。

WaitGroup本质上是一个计数器，起阻塞作用，让所有的goroutine执行完再结束。

```go
package main

import (
	"fmt"
	"net"
	"sync"
	"time"
)

func main() {
	start := time.Now()

	var wg sync.WaitGroup

	ip := "11.111.10.11"
	startPort := 21
	endPort := 120
	for port := startPort; port < endPort; port++ {
		wg.Add(1)
		go func(j int) {
			defer wg.Done()
			address := fmt.Sprintf("%s:%d", ip, j)
			conn, err := net.Dial("tcp", address)
			if err != nil {
				fmt.Printf("%s 关闭了\n", address)
				return
			}
			conn.Close()
			fmt.Printf("%s 打开了\n", address)
		}(port)
	}

	wg.Wait()

	elapsed := time.Since(start) / 1e9
	fmt.Printf("%d seconds", elapsed)
}
```

### **3. goroutine池并发版本实现**

可以使用另外一个管道results来阻塞。

```go
package main

import (
	"fmt"
	"net"
	"time"
)

var ip = "11.111.10.11"

func worker(ports chan int, results chan int) {
	for port := range ports {
		address := fmt.Sprintf("%s:%d", ip, port)
		conn, err := net.Dial("tcp", address)
		if err != nil {
			// fmt.Printf("%s 关闭了\n", address)
			results <- 0
			continue
		}
		conn.Close()
		// fmt.Printf("%s 打开了\n", address)
		results <- port
	}
}

func main() {
	start := time.Now()

	ports := make(chan int, 100)
	results := make(chan int)
	var openPort []int

	for i := 0; i < cap(ports); i++ {
		go worker(ports, results)
	}

	startPort := 21
	endPort := 120

	go func() {
		for port := startPort; port < endPort; port++ {
			ports <- port
		}
	}()

	for p := startPort; p < endPort; p++ {
		port := <-results
		if port != 0 {
			openPort = append(openPort, port)
		}
	}

	elapsed := time.Since(start) / 1e9
	fmt.Printf("%d seconds", elapsed)
}

```

