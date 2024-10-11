In this article, we will walk through the process of setting up a Kubernetes environment using Kind, building a simple Java Hello World application, and deploying it using Docker and Helm. This guide is perfect for developers looking to streamline their Kubernetes development workflow.

We will go through two phases here: the first phase focuses on setting up a local Kubernetes environment and deploying a Java Hello World application, while the second phase covers modifying the application and redeploying it to reflect those changes.

Prerequisites

Before you begin, ensure you have the following installed on your machine:

- Docker
- Kind
- kubectl
- Helm
- Maven


## Step 1: Create a Kind Cluster

Kind (Kubernetes IN Docker) allows you to run Kubernetes clusters in Docker containers. To create a Kind cluster, use the following command:

```
kind create cluster
```

Verify that your cluster is running:

```
kubectl cluster-info --context kind-kind
```

```
kubectl cluster-info --context kind-kind
Kubernetes control plane is running at https://127.0.0.1:42723
CoreDNS is running at https://127.0.0.1:42723/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```


## Step 2: Create a Java Hello World Application
Create a folder structure like below and create two files as mentioned

```
└── src
    └── main
        ├── java
        │   └── com
        │       └── example
        │           └── HelloWorldApplication.java
        └── resources
            └── application.properties 
```

**HelloWorldApplication.java **

```
package com.example;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@SpringBootApplication
@RestController
public class HelloWorldApplication {

    public static void main(String[] args) {
        SpringApplication.run(HelloWorldApplication.class, args);
    }

    @GetMapping("/")
    public String hello() {
        return "Hello, World!";
    }
}
```

**application.properties**

```
server.port=8081
```

**Create a pom.xml file in the root folder** [same folder of src]

```
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example</groupId>
    <artifactId>hello-world-app</artifactId>
    <version>1.0-SNAPSHOT</version>
    <packaging>jar</packaging>

    <properties>
        <java.version>11</java.version>
        <spring.version>2.6.3</spring.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
            <version>${spring.version}</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <version>${spring.version}</version>
                <executions>
                    <execution>
                        <goals>
                            <goal>repackage</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.8.1</version>
                <configuration>
                    <source>${java.version}</source>
                    <target>${java.version}</target>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>

```

## Step 3: Build your application using maven

```
mvn clean package
```
This will create a jar file inside the target folder

```
$ls
pom.xml  src  target
$ls target/
classes  generated-sources  hello-world-app-1.0-SNAPSHOT.jar  hello-world-app-1.0-SNAPSHOT.jar.original  maven-archiver  maven-status
```

## Step 4: Build the Docker Image

Create a **Dockerfile** in the root directory [same dir of src]

```
# Use the official OpenJDK image as the base image
FROM openjdk:11-jre-slim

# Set the working directory
WORKDIR /app

# Copy the JAR file into the container
COPY target/hello-world-app-1.0-SNAPSHOT.jar app.jar

# Expose the port the app runs on
EXPOSE 8081

# Run the application
ENTRYPOINT ["java", "-jar", "app.jar"]
```

Build your application and create the Docker image:

```
docker build -t <your-dockerhub-username>/hello-world-app:latest .
```

## Step 5: Push the Docker Image to Docker Hub

Log in to your Docker Hub account:

```
docker login
```
Push your image to Docker Hub:

```
docker push <your-dockerhub-username>/hello-world-app:latest
```

## Step 6: Create Helm Charts
Generate a new Helm chart:

```
helm create hello-world-java
```
This will create below structure

```
├── hello-world-java
│   ├── charts
│   ├── Chart.yaml
│   ├── templates
│   │   ├── deployment.yaml
│   │   ├── _helpers.tpl
│   │   ├── hpa.yaml
│   │   ├── ingress.yaml
│   │   ├── NOTES.txt
│   │   ├── serviceaccount.yaml
│   │   ├── service.yaml
│   │   └── tests
│   │       └── test-connection.yaml
│   └── values.yaml
```

We need only two files from these. Those are 

```
Chart.yaml
values.yaml
templates/deployment.yaml
```
Edit the `values.yaml` file to specify your Docker image and Service

```
image:
  repository: <your-dockerhub-username>/hello-world-app
  tag: latest
```

```
service:
  type: NodePort
  port: 8081
```

Make sure the `deployment.yaml` file under `templates` is configured correctly:

```
containers:
        - name: {{ .Chart.Name }}
          securityContext:
            {{- toYaml .Values.securityContext | nindent 12 }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.service.port }}
              protocol: TCP
```

## Step 7: Deploy the Application Using Helm

```
helm install hello-world-java ./hello-world-java
```
Verify that your application is running:

```
kubectl get pods
```

```
NAME                               READY   STATUS    RESTARTS   AGE
hello-world-java-54fb55948-597hk   1/1     Running   0          144m
```

## Step 8: Access Your Application
To access your application, you can use port forwarding since you're running in a Kind cluster:

```
kubectl port-forward svc/hello-world-java 8081:8081
```
You should see output indicating that the connection is being forwarded. Open your browser and go to:

```
http://localhost:8081
```
Now you can successfully access the Hello World page from your browser. Let’s explore how to make a change and redeploy it.

## Step 9: Modify Application

First, navigate to the `HelloWorldApplication.java `file and change "Hello World" to "Hi World."

## Step 10: Rebuild your application using Maven

```
mvn clean package
```

## Step 11: Re-build your Docker image

```
docker build -t <your-dockerhub-username>/hello-world-app:latest .
```

## Step 12: Push the updated image to Docker Hub

```
docker push <your-dockerhub-username>/hello-world-app:latest
```
Before deploying it with Helm, ensure the image pull policy is set to Always in `values.yaml`; otherwise, it wont download the latest image. 

```
image:
  repository: your-docker-hub-username/hello-world-java
  pullPolicy: Always
  tag: "latest"
```

## Step 13: Helm Upgrade
Now you can upgrade your Helm release:

```
helm upgrade hello-world-java ./hello-world-java
```
Finally, set up port forwarding to access your application:

```
kubectl port-forward svc/hello-world-java 8081:8081
```

## Step 14:Access Your Application
 
Now you can navigate to `http://localhost:8081` in your browser and see your changes reflected.You will now see the "Hi World" page instead of "Hello World."


