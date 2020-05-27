---
date: 2018-12-18T18:00:00
description: "Learn how to compile and port Go for SPARC Solaris "
#featured_image: "/images/golang_gopher.png"
tags: ["GO","Solaris"]
# categories: "POC"
title: "Porting Go to SPARC Solaris"
comment : false
---

This article aims to present the steps required to create a port of Go for SPARC Solaris. The 'spark' that kicked this idea off was part of an aspiration to first get the Prometheus node exporter for the core machine metrics running on SPARC tin for Solaris 11.3, then potentially other node exporters such as the JMX exporter and a Oracle DB exporter.

Note: Oracle announced Go version 1.7.6 will be available in Solaris 11.4. However, it seems like that 11.4 is a no go for a lot of SPARC hardware

A port of Go for SPARC is available here:

> https://github.com/4ad/go

This post is based on discussions from the porting go to sparc Solaris discussion from the golang-dev@googlegroups.com Google group available here:

> https://Go-dev.narkive.com/g6r4eJoA/porting-go-to-sparc-solaris


What I ended up doing was compiling Go on an x86 based Linux system and then copying that over and using it on my sparc based Solaris system.

## Let's give it a GO!

First things first, let's stage the required media and get Go installed on our Linux box.

```text
mkdir -p /opt/stage
cd /opt/stage
wget https://dl.google.com/go/go1.12.5.linux-amd64.tar.gz
tar -C /usr/local -xzf go1.12.5.linux-amd64.tar.gz
export PATH=$PATH:/usr/local/go/bin
git clone https://github.com/4ad/go.git
```

Now every thing is in place, time to try a build for SPARC Solaris:
```text
cd /opt/stage/go
echo "GO_build_01" >> VERSION

export GOROOT_BOOTSTRAP=/usr/local/go
export GOROOT=/usr/local/go

export GOARCH=sparc64
export GOOS=solaris

cd /opt/stage/go/src
./make.bash
```

Great, that looked to have worked. Now let's archive that up and get it over to the SPARC server.

```text
cd /opt/stage
tar -cvf go_go_sun.tar go
```


## Testing

Here's a quick example of building and running a 'Hello World' program. All the below steps are performed on a Solaris 11.3 zone.

Sample code for 'hello.go':
```go
package main

import "fmt"

func main() {
        fmt.Printf("hello, world\n")
}
```

After copying over and extracting our 'go_go_sun.tar' archive to '/usr/local/gosparc' set the below relevant enviorment variables.
```text
mkdir /usr/local/gosparc
export GOOS=solaris
export GOARCH=sparc64
export PATH=$PATH:/usr/local/gosparc/go/bin/solaris_sparc64
go build
```

Post performing the build for our 'Hello World' application we now have a 'hello' binary.
```text
ls -l
  total 1675
  -rw-r--r--   1 user     user          82 Jun  5 17:06 hello.go
  -rwxr-xr-x   1 user     user     1814246 Jun  6 09:52 hello
```
```text
./hello
  hello, world
```
```text
go version
  go version GO_build_01 solaris/sparc64
```

Great it works!
