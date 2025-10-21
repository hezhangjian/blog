---
title: Go 调用交互式shell
link:
date: 2021-12-12 17:34:56
tags:
---

本文的代码已上传到[github](https://github.com/hezhangjian/go_demo/blob/main/demo_shell/interactive_shell_test.go)

交互式shell常用在输入密码的场景，为了防止密码泄露在`cmdline`中被`ps -ef`读取

举个🌰

```bash
#!/bin/bash

read -s -p "Enter Password: "  pwd
echo -e "\nYour password is: " $pwd
```

go调用交互式shell代码样例如下

```go
func TestCallInteractiveShell(t *testing.T) {
	dir, err := os.Getwd()
	if err != nil {
		panic(err)
	}
	cmd := exec.Command("/bin/bash", dir+"/interactive_shell.sh")
	var stdout, stderr bytes.Buffer
	cmd.Stdout = &stdout
	cmd.Stderr = &stderr
	buffer := bytes.Buffer{}
	buffer.Write([]byte("ZhangJian"))
	cmd.Stdin = &buffer
	err = cmd.Run()
	if err != nil {
		panic(err)
	}
	outStr, errStr := string(stdout.Bytes()), string(stderr.Bytes())
	fmt.Println("output is ", outStr)
	fmt.Println("err output is ", errStr)
	fmt.Println("Execute Over")
}
```

输出结果

```
=== RUN   TestCallInteractiveShell
output is  Your password is:  ZhangJian

err output is  
Execute Over
--- PASS: TestCallInteractiveShell (0.00s)
PASS
```
