---
layout:     post
title:      kubernetes client-go 实现 kubectl copy
subtitle:   kubernetes client-go 实现 kubectl copy
date:       2019-08-31
author:     ica10888
catalog: true
tags:
    - kubernetes
    - client-go
---


#  kubernetes client-go 实现 kubectl copy

### kubectl cp

kubectl cp 指令是将容器当中的数据拷贝到当前机器上，同理，也可以将当前机器上的文件或文件目录拷贝到容器当中。

如果去查看 kubernetes/pkg/kubectl/cmd/cp/cp.go 目录下的代码，可以看到具体实现。

其实现是将文件或文件夹抽象成一个数据流 Stream，然后通过 kube-apiserver 访问容器，对进行文件传输。


### tar

使用` tar -cf -` 将具有文件夹结构的数据转换成数据流，再通过 linux 管道接收这个数据流；通过` tar -xf -` 将数据流转换成 linux 文件系统。

以一下文件结构为例

```
/tmp
├─hello
   └─ world    "你好"
```
当使用 tar 的时候，会将文件目录非压缩地转换成数据流

``` shell

[root@iZt4nipafhgmbv6v4e4yjfZ hello]# tar -cf - /tmp/hello
tar: Removing leading `/' from member names
tmp/hello/0000755000000000000000000000000013542405175011465 5ustar  rootroottmp/hello/world0000644000000000000000000000000713542405175012534 0ustar  rootroot你好

```
可以看到数据流中包含有 mod 和 own 的内容。

包括有文件属性与权限

比如目录`/tmp/hello`是0000755,文件`/tmp/hello/world`是0000644，因为没有特殊权限SUID、SGID、SBIT，这里前面都是0

接下来表示文件的储存位置000000000000000000000000013542405175011465 和 0000644000000000000000000000000713542405175012534

然后是文件类型，由于`/tmp/hello`是目录，`/tmp/hello/world`是文件，分别用`5ustar`和`0ustar`表示目录类型和文件类型

接下来就是文件内容

`rootroot` 分别表示拥有者及所属群组，linux的权限即以此为划分的

比如文件`/tmp/hello/world` 的0644权限，即`-rw-r--r--`，文件拥有者root用户拥有读写权限,所属群组root的用户和其他用户只拥有读权限。

然后就是文件数据了。


### 代码

代码结构如下

``` go
├─kubectl
   │  client.go
   │  cp.go
   └─ stub.s

```

client实现

``` go
package kubectl 

import (
	corev1client "k8s.io/client-go/kubernetes/typed/core/v1"
	"k8s.io/client-go/rest"
	"k8s.io/client-go/tools/clientcmd"
)


func InitRestClient() (*rest.Config, error, *corev1client.CoreV1Client) {
	// Instantiate loader for kubeconfig file.
	kubeconfig := clientcmd.NewNonInteractiveDeferredLoadingClientConfig(
		clientcmd.NewDefaultClientConfigLoadingRules(),
		&clientcmd.ConfigOverrides{},
	)
	// Get a rest.Config from the kubeconfig file.  This will be passed into all
	// the client objects we create.
	restconfig, err := kubeconfig.ClientConfig()
	if err != nil {
		panic(err)
	}
	// Create a Kubernetes core/v1 client.
	coreclient, err := corev1client.NewForConfig(restconfig)
	if err != nil {
		panic(err)
	}
	return restconfig, err, coreclient
}
```
copyToPod 和 copyFromPod 实现

``` go
package kubectl

import (
	"archive/tar"
	"fmt"
	"io"
	corev1 "k8s.io/api/core/v1"
	"k8s.io/client-go/kubernetes/scheme"
	"k8s.io/client-go/tools/remotecommand"
	_ "k8s.io/kubernetes/pkg/kubectl/cmd/cp"
	cmdutil "k8s.io/kubernetes/pkg/kubectl/cmd/util"
	"log"
	"os"
	"path"
	"path/filepath"
	"strings"
	_ "unsafe"
)

func (i *pod) copyToPod(srcPath string, destPath string) error {
	restconfig, err, coreclient := InitRestClient()

	reader, writer := io.Pipe()
	if destPath != "/" && strings.HasSuffix(string(destPath[len(destPath)-1]), "/") {
		destPath = destPath[:len(destPath)-1]
	}
	if err := checkDestinationIsDir(i, destPath); err == nil {
		destPath = destPath + "/" + path.Base(srcPath)
	}
	go func() {
		defer writer.Close()
		err := cpMakeTar(srcPath, destPath, writer)
		cmdutil.CheckErr(err)
	}()
	var cmdArr []string

	cmdArr = []string{"tar", "-xf", "-"}
	destDir := path.Dir(destPath)
	if len(destDir) > 0 {
		cmdArr = append(cmdArr, "-C", destDir)
	}
	//remote shell.
	req := coreclient.RESTClient().
		Post().
		Namespace(i.Namespace).
		Resource("pods").
		Name(i.Name).
		SubResource("exec").
		VersionedParams(&corev1.PodExecOptions{
			Container: i.ContainerName,
			Command:   cmdArr,
			Stdin:     true,
			Stdout:    true,
			Stderr:    true,
			TTY:       false,
		}, scheme.ParameterCodec)

	exec, err := remotecommand.NewSPDYExecutor(restconfig, "POST", req.URL())
	if err != nil {
		log.Fatalf("error %s\n", err)
		return err
	}
	err = exec.Stream(remotecommand.StreamOptions{
		Stdin:  reader,
		Stdout: os.Stdout,
		Stderr: os.Stderr,
		Tty:    false,
	})
	if err != nil {
		log.Fatalf("error %s\n", err)
		return err
	}
	return nil
}

func checkDestinationIsDir(i *pod, destPath string) error {
	return i.Exec([]string{"test", "-d", destPath})
}

//go:linkname cpMakeTar k8s.io/kubernetes/pkg/kubectl/cmd/cp.makeTar
func cpMakeTar(srcPath, destPath string, writer io.Writer) error

func (i *pod) copyFromPod(srcPath string, destPath string) error {
	restconfig, err, coreclient := InitRestClient()
	reader, outStream := io.Pipe()
	//todo some containers failed : tar: Refusing to write archive contents to terminal (missing -f option?) when execute `tar cf -` in container
	cmdArr := []string{"tar", "cf", "-", srcPath}
	req := coreclient.RESTClient().
		Get().
		Namespace(i.Namespace).
		Resource("pods").
		Name(i.Name).
		SubResource("exec").
		VersionedParams(&corev1.PodExecOptions{
			Container: i.ContainerName,
			Command:   cmdArr,
			Stdin:     true,
			Stdout:    true,
			Stderr:    true,
			TTY:       false,
		}, scheme.ParameterCodec)

	exec, err := remotecommand.NewSPDYExecutor(restconfig, "POST", req.URL())
	if err != nil {
		log.Fatalf("error %s\n", err)
		return err
	}
	go func() {
		defer outStream.Close()
		err = exec.Stream(remotecommand.StreamOptions{
			Stdin:  os.Stdin,
			Stdout: outStream,
			Stderr: os.Stderr,
			Tty:    false,
		})
		cmdutil.CheckErr(err)
	}()
	prefix := getPrefix(srcPath)
	prefix = path.Clean(prefix)
	prefix = cpStripPathShortcuts(prefix)
	destPath = path.Join(destPath, path.Base(prefix))
	err = untarAll(reader, destPath, prefix)
	return err
}

func untarAll(reader io.Reader, destDir, prefix string) error {
	tarReader := tar.NewReader(reader)
	for {
		header, err := tarReader.Next()
		if err != nil {
			if err != io.EOF {
				return err
			}
			break
		}

		if !strings.HasPrefix(header.Name, prefix) {
			return fmt.Errorf("tar contents corrupted")
		}

		mode := header.FileInfo().Mode()
		destFileName := filepath.Join(destDir, header.Name[len(prefix):])

		baseName := filepath.Dir(destFileName)
		if err := os.MkdirAll(baseName, 0755); err != nil {
			return err
		}
		if header.FileInfo().IsDir() {
			if err := os.MkdirAll(destFileName, 0755); err != nil {
				return err
			}
			continue
		}

		evaledPath, err := filepath.EvalSymlinks(baseName)
		if err != nil {
			return err
		}

		if mode&os.ModeSymlink != 0 {
			linkname := header.Linkname

			if !filepath.IsAbs(linkname) {
				_ = filepath.Join(evaledPath, linkname)
			}

			if err := os.Symlink(linkname, destFileName); err != nil {
				return err
			}
		} else {
			outFile, err := os.Create(destFileName)
			if err != nil {
				return err
			}
			defer outFile.Close()
			if _, err := io.Copy(outFile, tarReader); err != nil {
				return err
			}
			if err := outFile.Close(); err != nil {
				return err
			}
		}
	}

	return nil
}

func getPrefix(file string) string {
	return strings.TrimLeft(file, "/")
}

//go:linkname cpStripPathShortcuts k8s.io/kubernetes/pkg/kubectl/cmd/cp.stripPathShortcuts
func cpStripPathShortcuts(p string) string


```

同目录下的 stub.s ，go:linkname依赖的，没有任何内容，同级目录下需 touch stub.s

``` go

```

