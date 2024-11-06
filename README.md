Proyecto de Resiliencia con Spring Boot, Resilience4j, Chaos Monkey y Docker

Este proyecto muestra cómo implementar el patrón Bulkhead en una aplicación Spring Boot usando Resilience4j y Chaos Monkey para simular fallos y probar la resiliencia de la aplicación. Además, se despliega la aplicación en Docker y se sube a Docker Hub utilizando Docker Compose y Minikube.

Requisitos Previos
Java 17 o superior
Maven (para compilar y construir el proyecto)
Docker y Docker Compose (para el despliegue en contenedores)
Cuenta en Docker Hub (para subir las imágenes)
Paso a Paso
1. Crear el Proyecto Spring Boot en Spring Initializr
Ve a Spring Initializr.
Configura el proyecto con los siguientes parámetros:
Project: Maven Project
Language: Java
Spring Boot: 3.x.x
Group: com.example
Artifact: resilience-bulkhead
Packaging: Jar
Java: 17
Agrega las siguientes dependencias:
Resilience4j (para manejar la resiliencia de la aplicación)
Chaos Monkey (para simular fallos en el sistema)
Spring Web (para crear endpoints de prueba)
Genera el proyecto y descárgalo. Luego, descomprime y abre el proyecto en tu editor de código preferido.
2. Modificar pom.xml
Spring Initializr añadirá automáticamente las dependencias de Resilience4j y Chaos Monkey en el archivo pom.xml. Asegúrate de que estas dependencias estén presentes:

xml
Copiar código
<dependencies>
    <!-- Spring Boot y configuración -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-aop</artifactId>
    </dependency>
    
    <!-- Resilience4j -->
    <dependency>
        <groupId>io.github.resilience4j</groupId>
        <artifactId>resilience4j-spring-boot3</artifactId>
        <version>2.0.2</version>
    </dependency>

    <!-- Chaos Monkey -->
    <dependency>
        <groupId>de.codecentric</groupId>
        <artifactId>chaos-monkey-spring-boot</artifactId>
        <version>3.1.0</version>
    </dependency>

    <!-- Actuator -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
</dependencies>
3. Configurar application.properties
Configura el límite de concurrencia de Bulkhead y habilita Chaos Monkey en src/main/resources/application.properties:

properties
Copiar código
# Configuración del servidor
server.address=0.0.0.0
server.port=8081
java.net.preferIPv4Stack=true

# Configuración de Bulkhead
resilience4j.bulkhead.instances.userService.maxConcurrentCalls=3
resilience4j.bulkhead.instances.userService.maxWaitDuration=0ms

# Exposición de Actuator
management.endpoint.health.show-details=always
management.health.bulkhead.enabled=true

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
Implementa el patrón Bulkhead en src/main/java/com/example/microusers/service/UserService.java:

java
Copiar código
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
    
    // Otros métodos con Bulkhead
}
5. Crear el docker-compose.yml
En la carpeta raíz del proyecto (~/sincronizado/final-Bulkhead/), crea un archivo llamado docker-compose.yml:

yaml
Copiar código
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
6. Ejecutar Docker Compose
Guarda este archivo en la carpeta ~/sincronizado/final-Bulkhead/ y ejecuta:

bash
Copiar código
docker-compose up -d
Esto iniciará ambos contenedores en segundo plano.

Despliegue en Minikube
Requisitos Previos
Docker: Para que Minikube use Docker como controlador.
Kubectl: Herramienta de línea de comandos de Kubernetes.
Paso 1: Instalar Minikube
Descarga e instala Minikube:

bash
Copiar código
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
Verifica la instalación:

bash
Copiar código
minikube version
Paso 2: Iniciar Minikube
Inicia el clúster de Minikube con el controlador Docker:

bash
Copiar código
minikube start --driver=docker
Configura el entorno de Docker de Minikube:

bash
Copiar código
eval $(minikube -p minikube docker-env)
Paso 3: Desplegar en Kubernetes
Apunta tu terminal a Docker en Minikube:

bash
Copiar código
eval $(minikube -p minikube docker-env)
Aplica los archivos YAML de despliegue:

bash
Copiar código
kubectl apply -f frontend-deployment.yaml
kubectl apply -f microusers-deployment.yaml
Verifica que los pods y servicios estén corriendo:

bash
Copiar código
kubectl get pods
kubectl get services
Obtén la URL de los servicios para probarlos:

bash
Copiar código
minikube service frontend --url
minikube service microusers --url
Accede a los servicios desde tu navegador o mediante curl:

bash
Copiar código
curl <URL-del-frontend>
curl <URL-del-microusers>
