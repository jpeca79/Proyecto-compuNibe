Este proyecto muestra cómo implementar el patrón Bulkhead en una aplicación Spring Boot usando Resilience4j y Chaos Monkey para simular fallos y probar la resiliencia de la aplicación. Además, se despliega la aplicación en Docker y se sube a Docker Hub utilizando Docker Compose.

Requisitos Previos
Java 17 o superior
Maven (si deseas compilar y construir manualmente el proyecto)
Docker y Docker Compose para el despliegue en contenedores
Cuenta en Docker Hub (para subir las imágenes)
Paso a Paso
1. Crear el Proyecto Spring Boot en Spring Initializr
Ve a Spring Initializr.

Configura el proyecto:

Project: Maven Project
Language: Java
Spring Boot: Version compatible (e.g., 3.x.x)
Group: com.example
Artifact: resilience-bulkhead
Packaging: Jar
Java: 17
Agrega las siguientes dependencias desde la interfaz:

Resilience4j (para manejar la resiliencia de la aplicación)
Chaos Monkey (para simular fallos en el sistema)
Spring Web (para crear endpoints de prueba)
Genera el proyecto y descárgalo. Luego, descomprime y abre el proyecto en tu editor de código preferido.

2. Modificar pom.xml
Spring Initializr habrá agregado automáticamente las dependencias de Resilience4j y Chaos Monkey en el archivo pom.xml. Verifica que estas dependencias están presentes:
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.3.5</version>
        <relativePath/>
    </parent>
    <groupId>com.example</groupId>
    <artifactId>microusers</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>microusers</name>
    <description>Demo project for Spring Boot</description>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-devtools</artifactId>
            <scope>runtime</scope>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-configuration-processor</artifactId>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-aop</artifactId>
        </dependency>
        <dependency>
            <groupId>io.github.resilience4j</groupId>
            <artifactId>resilience4j-spring-boot3</artifactId>
            <version>2.0.2</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <dependency>
            <groupId>de.codecentric</groupId>
            <artifactId>chaos-monkey-spring-boot</artifactId>
            <version>3.1.0</version>
        </dependency>

        <!-- Dependencia para pruebas con JUnit 5 -->
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter-engine</artifactId>
            <version>5.9.3</version>
            <scope>test</scope>
        </dependency>
    </dependencies>
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>

3. Configurar application.properties
Configura el límite de concurrencia de Bulkhead y habilita Chaos Monkey en src/main/resources/application.properties:

# src/main/resources/application.properties
server.address=0.0.0.0
server.port=8081
java.net.preferIPv4Stack=true
# Configuración de Bulkhead
resilience4j.bulkhead.instances.userService.maxConcurrentCalls=3
resilience4j.bulkhead.instances.userService.maxWaitDuration=0ms

# Exposición de Actuator
management.endpoint.health.show-details=always
management.health.bulkhead.enabled=true
spring.cloud.discovery.enabled=false
# Configuración de Chaos Monkey
management.endpoints.web.exposure.include=*
management.endpoint.chaosmonkey.enabled=true
chaos.monkey.enabled=true
chaos.monkey.watcher.controller=true
chaos.monkey.watcher.service=true
chaos.monkey.assaults.level=1
chaos.monkey.assaults.latencyActive=true
chaos.monkey.assaults.latencyRangeStart=500
chaos.monkey.assaults.latencyRangeEnd=1500
chaos.monkey.assaults.exceptionsActive=true
chaos.monkey.assaults.exceptionProbability=100

4. Editar UserService.java
Crea o edita el archivo UserService.java en src/main/java/com/example/microusers/service para implementar el patrón Bulkhead.

// src/main/java/com/example/microusers/service/UserService.java
package com.example.microusers.service;

import com.example.microusers.model.User;
import io.github.resilience4j.bulkhead.annotation.Bulkhead;
import org.springframework.stereotype.Service;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.atomic.AtomicLong;

@Service
public class UserService {

    private List<User> users = new ArrayList<>();
    private AtomicLong counter = new AtomicLong(1);

    public UserService() {
        // Agregar usuarios predeterminados
        users.add(new User(counter.getAndIncrement(), "Juan Perez", "juan@example.com", "juan", "password123"));
        users.add(new User(counter.getAndIncrement(), "Maria Gomez", "maria@example.com", "maria", "password456"));
    }

    @Bulkhead(name = "userService", type = Bulkhead.Type.SEMAPHORE)
    public User authenticate(String username, String password) {
        return users.stream()
                .filter(u -> u.getUsername().equals(username) && u.getPassword().equals(password))
                .findFirst()
                .orElse(null);
    }

    @Bulkhead(name = "userService", type = Bulkhead.Type.SEMAPHORE)
    public List<User> getAllUsers() {
        return users;
    }

    @Bulkhead(name = "userService", type = Bulkhead.Type.SEMAPHORE)
    public User createUser(User user) {
        user.setId(counter.getAndIncrement());
        users.add(user);
        return user;
    }

    @Bulkhead(name = "userService", type = Bulkhead.Type.SEMAPHORE)
    public User updateUser(Long id, User updatedUser) {
        User user = getUserById(id);
        if (user != null) {
            user.setName(updatedUser.getName());
            user.setEmail(updatedUser.getEmail());
            user.setUsername(updatedUser.getUsername());
            user.setPassword(updatedUser.getPassword());
            return user;
        }
        return null;
    }

    @Bulkhead(name = "userService", type = Bulkhead.Type.SEMAPHORE)
    public boolean deleteUser(Long id) {
        return users.removeIf(u -> u.getId().equals(id));
    }

    private User getUserById(Long id) {
        return users.stream().filter(u -> u.getId().equals(id)).findFirst().orElse(null);
    }
}

6. Crear el docker-compose.yml
En la carpeta raíz del proyecto (~/sincronizado/final-Bulkhead/), crea un archivo llamado docker-compose.yml para definir el servicio:
vagrant@ubuntu-bionic:~/sincronizado/final-Bulkhead$ sudo cat docker-compose.yml
version: '3.8'

services:
  microusers:
    image: jpeca79/microusers:latest
    container_name: microusers
    ports:
      - "8081:8081"
    networks:
      - app-network

  frontend:
    image: jpeca79/frontend:latest
    container_name: frontend
    ports:
      - "8080:8080"
    networks:
      - app-network

networks:
  app-network:
    driver: bridge
    ![image](https://github.com/user-attachments/assets/17c6d78b-e430-49d5-a6d0-27011f7016ef)

    Instrucciones
Guarda este archivo en la carpeta ~/sincronizado/final-Bulkhead/.

Ejecuta el siguiente comando desde la carpeta raíz del proyecto para levantar ambos contenedores:

bash
Copiar código
docker-compose up -d

Requisitos Previos
Docker: Asegúrate de tener Docker instalado, ya que Minikube usará Docker como su controlador.
Kubectl: Herramienta de línea de comandos de Kubernetes, necesaria para gestionar el clúster. Puedes instalarla siguiendo la guía oficial.
Paso 1: Instalar Minikube
Descarga Minikube desde la terminal:

bash
Copiar código
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
Verifica la instalación ejecutando:

bash
Copiar código
minikube version
Paso 2: Iniciar Minikube
Inicia el clúster de Minikube utilizando el controlador de Docker:

bash
Copiar código
minikube start --driver=docker
Configura tu terminal para usar el entorno Docker de Minikube:

bash
Copiar código
eval $(minikube -p minikube docker-env)
Esto permitirá que cualquier imagen de Docker que construyas se utilice dentro de Minikube.

Para iniciar tus servicios en Minikube después de crear los archivos de manifiesto YAML, sigue estos pasos:

Asegúrate de estar en la carpeta donde tienes los archivos frontend-deployment.yaml y microusers-deployment.yaml.

Verifica que Minikube esté en funcionamiento:

Si no has iniciado Minikube, hazlo con:

bash
Copiar código
minikube start --driver=docker
Apunta tu terminal a Docker en Minikube para asegurarte de que las imágenes se usen dentro del clúster:

bash
Copiar código
eval $(minikube -p minikube docker-env)
Desplegar los servicios en Kubernetes usando los archivos YAML.

Ejecuta los siguientes comandos en la carpeta donde tienes los archivos YAML para crear los despliegues y servicios en Kubernetes:

bash
Copiar código
kubectl apply -f frontend-deployment.yaml
kubectl apply -f microusers-deployment.yaml
Verifica que los pods y servicios estén corriendo:

Para ver los pods, ejecuta:

bash
Copiar código
kubectl get pods
Deberías ver algo como esto cuando los pods estén en estado "Running":

sql
Copiar código
NAME                          READY   STATUS    RESTARTS   AGE
frontend-xxxxx-yyyyy          1/1     Running   0          1m
microusers-xxxxx-yyyyy        1/1     Running   0          1m
Verificar los servicios y obtener la URL:

Para ver los servicios y los puertos asignados:

bash
Copiar código
kubectl get services
Deberías ver algo como:

scss
Copiar código
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
frontend     NodePort    10.108.231.65    <none>        8080:30002/TCP   2m
microusers   NodePort    10.103.22.139    <none>        8081:30001/TCP   2m
Accede a los servicios:

Para acceder a los servicios, puedes utilizar el siguiente comando para obtener la URL directa desde Minikube:

bash
Copiar código
minikube service frontend --url
minikube service microusers --url
Esto te dará las URLs donde puedes acceder a frontend y microusers desde tu navegador o mediante curl.

Prueba de los Servicios:

Una vez que tengas las URLs, abre el navegador y navega a esas direcciones, o utiliza curl para probarlas:

bash
Copiar código
curl <URL-del-frontend>
curl <URL-del-microusers>
Esto debería iniciar y exponer tus aplicaciones en el clúster de Minikube, accesibles mediante las URLs proporcionadas.

