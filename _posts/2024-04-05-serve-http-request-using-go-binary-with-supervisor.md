---
layout: post
title: 'Serving HTTP requests using a Go binary file and Supervisor in Ubuntu'
date: 2024-04-05
categories: dev
---

Go builds everything into a single binary file, making deployment easier. However, it's not feasible to keep the terminal open to run the Go file forever.
Supervisor comes to the rescue by managing the process for us!

## Prepare in the local machine

Firstly, we create a basic Go hello file that serves HTTP requests.
We will run it on port 8000 or any other port given by the environment variable.

```go
package main

import (
	"fmt"
	"net/http"
	"os"
)

func main() {
	port := os.Getenv("PORT")
	if port == "" {
		port = "8000"
	}

	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		w.Write([]byte(fmt.Sprintf("Hello from port %s", port)))
	})

	fmt.Printf("Listening on :%s\n", port)
	http.ListenAndServe(fmt.Sprintf(":%s", port), nil)
}
```

Build the binary file, please make sure to add the two environment variables to build for Linux. Otherwise, it will build the wrong binary. By default Go will build the binary file for the current operating system (in my case is MacOS).

```bash
GOOS=linux GOARCH=amd64 go build -o hello_bin hello.go
```

Copy the built binary file `hello_bin` to the server's `/home/ubuntu/hello` folder. I recommend using `rsync` instead of `scp` because it offers better performance and can handle file and folder copying, updating old files, and creating new folders if they don't exist, all in a single command 🚀.

```bash
rsync -avz ./hello_bin ubuntu@159.223.47.97:/home/ubuntu/hello
```

## Go to remote Ubuntu server

Go to our Ubuntu remote server:

```bash
ssh ubuntu@159.223.47.97 #change to your user and server IP
```

Please install Supervisor if it is not already installed.

```bash
sudo apt update && sudo apt install supervisor
```

Let's create a Supervisor configuration file called `hello1.conf`.
You can use either `vim` or `nano` to create or edit the file.

```bash
sudo vim /etc/supervisor/conf.d/hello1.conf
```

Please be noted to use the full path for the `command` key, rather than a relative path from the current directory.

```
[program:hello1]
command=/home/ubuntu/hello/hello_bin
directory=/home/ubuntu/hello
user=ubuntu
autostart=true
autorestart=true
stderr_logfile=/var/log/hello1.err.log
stdout_logfile=/var/log/hello1.out.log
```

Similarly, let's create a program configuration file called `hello2.conf`. This file should include the environment variable `PORT=8001` so that later we can run our two programs simultaneously on different ports.

```bash
sudo vim /etc/supervisor/conf.d/hello2.conf
```

```
[program:hello2]
command=/home/ubuntu/hello/hello_bin
directory=/home/ubuntu/hello
user=ubuntu
autostart=true
autorestart=true
stderr_logfile=/var/log/hello2.err.log
stdout_logfile=/var/log/hello2.out.log
environment=PORT=8001
```

Now, let's update the Supervisor with the new configuration. Let's use the `supervisorctl reread` command for two purposes:

- to help Supervisor review new information (without changing the configuration)
- and to check if our newly added configurations are valid before performing the actual update.

```bash
sudo supervisorctl reread
# hello1: available
# hello2: available
```

Next, we perform the actual update in configuration state of Supervisor. After update the programs are ready to run.

```bash
sudo supervisorctl update
```

Now, we can run our programs using either the `start` or `restart` command. It is okay to simply use `restart` as if the program already run we will get a small error but it still work properly.

```bash
sudo supervisorctl restart hello1
# hello1: ERROR (not running) # you see it when the program has never run and it is FINE
# hello1: started
```

Do the same for hello2:

```bash
sudo supervisorctl restart hello2
# hello2: stopped # you see it when the program is currently running
# hello2: started
```

## Moment of Truth

Go back to our **Local machine** to test our two programs hello1 and hello2:

```bash
curl http://159.223.47.97:8000/
# Hello from port 8000

curl http://159.223.47.97:8001/
# Hello from port 8001
```

Great! We can see that `hello2` is receiving the `PORT` environment variable and running properly.

We've done now!

## Bonus

To group `hello1` and `hello2` together for easier management, you can modify the `/etc/supervisor/conf.d/hello2.conf` file by adding two additional lines at the end.

```
[program:hello2]
command=/home/ubuntu/hello/hello_bin
directory=/home/ubuntu/hello
user=ubuntu
autostart=true
autorestart=true
stderr_logfile=/var/log/hello2.err.log
stdout_logfile=/var/log/hello2.out.log
environment=PORT=8001

[group:hello]
programs=hello1,hello2
```

Now, we can run the group `hello:` (note the colon `:`) to manage both `hello1` and `hello2` together.

```bash
sudo supervisorctl reread
sudo supervisorctl update
sudo supervisorctl restart hello:
```

Please note that you can no longer directly interact with the individual processes `hello1` and `hello2`. Instead, you will need to use the `hello` group to manage them. Running individual program is error now:

```bash
sudo supervisorctl restart hello1
# hello1: ERROR (no such process)
```

We must do this instead:

```bash
sudo supervisorctl restart hello:
# hello:hello1: stopped
# hello:hello2: stopped
# hello:hello1: started
# hello:hello2: started
```

Source code: https://gist.github.com/anvodev/19c1402b93258bb4272ecf4dd64ba6bc

Thanks for reading! Good luck with the server-related tasks!
