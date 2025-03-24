---
title: Building Container Images - A Deep Dive into Dockerfiles
date: 2025-03-15 15:39:12
tags: ["devops", "docker", "container"]
---

# What is container?

A container image is a lightweight, standalone, and executable package that includes everything needed to run a piece of software, including the code, runtime, libraries, and dependencies. The Dockerfile is the blueprint used to build these images.


## Writing Dockerfile

A Dockerfile is a `text-based document` that's used to create a container image. It provides instructions to the image builder on the commands to run, files to copy, startup command, and more.

```Dockerfile
# Defining base image and workdir
FROM python:3.13.2-slim-bookworm
WORKDIR /app

#Copy everything in current folder to workdir
COPY . ./app

#Create user jack and run that container with that user
RUN useradd jack
USER jack

#Run this command when container started
CMD ["python", "main.py"]
```

### Explanation of Each Instruction

1. Base Image (FROM)
  - The FROM instruction specifies the base image.
  - Here, python:3.9-slim-bookworm is used, which is a lighter version of Python with minimal dependencies.
  - Every Docker image must start with a FROM instruction (except for "scratch" base images).

2. Set Working Directory (WORKDIR)
  - Sets /app as the working directory inside the container.
  - All subsequent commands (COPY, RUN, etc.) will be executed relative to this directory.

3. Copy Application Files (COPY)
  - Copies all files from the local build directory (the folder containing the Dockerfile) to the /app directory inside the container.
  - This allows the container to have access to all necessary application files.

4. Execute Some Commands (RUN)
  - Added newuser called jack using Unix CLI tool useradd.
  - After RUN keyword, you can write bash commands if base image supports it. 

5. Change Default User (USER)
  - This instruction sets the default user for all subsequent instructions.
  - Container shouldn't run as the root user

6. Define Default Command (CMD)
  - The CMD instruction specifies the default command to run the container.
  - Here, it starts the Python application by running `python main.py`
  - Difference between CMD and RUN:
    - `RUN` executes at **build time** (used for installing    dependencies, configurations).
    - `CMD` executes **at runtime** (used for starting the main process of the container).
  - If a user overrides `CMD` while running the container (e.g., docker run my-app bash), Docker will `execute the specified command instead of CMD`.

If your application has dependencies like requirements.txt for python you must add this line to your Dockerfile;

```Dockerfile
# Install any needed dependencies specified in requirements.txt
RUN pip install --no-cache-dir -r requirements.txt
```

If your application will listen port like 80, 8080, 4443 you must expose that port like this;

```Dockerfile
# Make port 8080 available to the world outside this container
EXPOSE 8080
```

## Common Dockerfile Commands Cheat Sheet

| Instruction  | Purpose |
|-------------|---------|
| `FROM`      | Defines the base image |
| `WORKDIR`   | Sets the working directory inside the container |
| `COPY`      | Copies files from host to container |
| `ADD`       | Copies and extracts files (supports URLs) |
| `RUN`       | Executes commands at build time |
| `CMD`       | Defines the default command to run at runtime |
| `ENTRYPOINT`| Sets a command that always executes |
| `EXPOSE`    | Declares ports for the application (documentation only) |
| `ENV`       | Sets environment variables |
| `VOLUME`    | Defines persistent storage directories |
| `LABEL`     | Adds metadata to the image |
| `ARG`       | Defines build-time variables |
| `HEALTHCHECK` | Defines a health check command for the container |

## What is HEALTHCHECK in Docker?

The HEALTHCHECK instruction in a Dockerfile allows you to define a command that runs inside the container to determine if the application is healthy or unhealthy. This helps Docker detect when a container is not functioning correctly and take appropriate actions like restarting it.

## How HEALTHCHECK Works?

Docker periodically runs the command defined in the HEALTHCHECK instruction. The command should return one of these exit codes:

0 → The container is healthy.
1 → The container is unhealthy.
2 → Reserved (not used by Docker).

Here is the example of Healtchek usage for your container;

```Dockerfile
FROM nginx:latest

RUN apt-get update && apt-get install -y curl

HEALTHCHECK --interval=30 --timeout=10s --start-period=5s --retries=3
  CMD curl -f http://localhost || exit

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

### Explanation of HEALTHCHECK Parameters

- --interval=30s → Runs the health check every 30 seconds.
- --timeout=10s → The check must complete within 10 seconds.
- --start-period=5s → Wait 5 seconds before the first check.
- --retries=3 → Marks the container unhealthy if it fails 3 times.
- CMD curl -f http://localhost || exit 1 → Runs curl to check if NGINX responds; exits with 1 if it fails.

After that we need to build our contaier like this `sudo docker build -t nginx-test .` so as finished run this container like this `sudo docker run -d -p 8080:80 nginx-test`

## How to use Environment Variables?

The `ENV command` in a Dockerfile is used to set environment variables inside the container. These environment variables can then be used by the application running inside the container.

```golang
package main

import (
	"fmt"
	"log"
	"net/http"
	"os"
)

func main() {
	// Read environment variables
	dbHost := os.Getenv("DB_HOST")
	dbPort := os.Getenv("DB_PORT")
	dbUser := os.Getenv("DB_USER")
	dbPassword := os.Getenv("DB_PASSWORD")

	// Print database configuration (for debugging)
	fmt.Printf("Connecting to DB at %s:%s as user %s\n", dbHost, dbPort, dbUser)

	// Simple HTTP handler
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintf(w, "Database Host: %s\nDatabase Port: %s\nUser: %s\n", dbHost, dbPort, dbUser)
	})

	// Start the server
	port := "8080"
	fmt.Printf("Server is running on port %s...\n", port)
	log.Fatal(http.ListenAndServe(":"+port, nil))
}
```

As you see 4 types of Environment variable should be read by golang applicatin. In order to achive this, we must define these environment variables in our dockerfile. Here is the example for this;

```Dockerfile
# Use the official Golang image
FROM golang:1.20

# Set environment variables inside the container
ENV DB_HOST=db.example.com
ENV DB_PORT=5432
ENV DB_USER=admin
ENV DB_PASSWORD=securepassword

# Set the working directory
WORKDIR /app

# Copy the Go application files
COPY main.go .

# Build the application
RUN go build -o app

# Expose port 8080
EXPOSE 8080

# Run the application
CMD ["./app"]
```

## Final Thoughts

Dockerfiles automate container image creation and ensure portability, consistency, and efficiency in deployments. By understanding these instructions, you can optimize performance, security, and manageability.
