# CST8915 Lab 3 - Deploying the Algonquin Pet Store on Azure

## Overview

This lab demonstrates the deployment of the Algonquin Pet Store application using Microsoft Azure services.

The application follows a microservices architecture and consists of the following components:

- **Store Front**: Vue.js frontend
- **Product Service**: Python Flask REST API
- **Order Service**: Node.js REST API
- **RabbitMQ**: Message broker used by the Order Service

The Product Service and Order Service are deployed using **Azure App Service**.

RabbitMQ is deployed on an **Azure Virtual Machine**.

The Store Front was originally intended to be deployed using **Azure Static Web Apps**. However, the Azure for Students subscription prevented the creation of the Static Web App because of an Azure regional deployment policy. Following the instructor-approved alternative, the Store Front was deployed on an **Azure Virtual Machine**.

---

## Application Architecture

```text
                         User Browser
                              |
                              v
                   Store Front Azure VM
                      Vue.js :8080
                       /          \
                      /            \
                     v              v
          Azure App Service    Azure App Service
           Product Service      Order Service
           Python / Flask       Node.js / Express
                                     |
                                     v
                               RabbitMQ Azure VM
                                  TCP 5672
                                     |
                                     v
                                order_queue
```

---

## Service Repositories

| Service | Repository |
| --- | --- |
| Order Service | [divine-wander/order-service](https://github.com/divine-wander/order-service) |
| Product Service | [divine-wander/product-service](https://github.com/divine-wander/product-service) |
| Store Front | [divine-wander/store-front](https://github.com/divine-wander/store-front) |

---

# Deployment

## 1. RabbitMQ

RabbitMQ was deployed on a dedicated Ubuntu 24.04 Azure Virtual Machine.

The RabbitMQ Management plugin was enabled to provide a browser-based interface for monitoring queues and messages.

### RabbitMQ Network Ports

| Port | Protocol | Purpose |
| --- | --- | --- |
| 22 | TCP | SSH administration |
| 5672 | TCP | RabbitMQ AMQP communication |
| 15672 | TCP | RabbitMQ Management UI |

The RabbitMQ AMQP inbound rule on port `5672` was restricted to the outbound IP addresses used by the Azure Order Service App Service.

The RabbitMQ Management interface was used to verify that orders were successfully published to the queue.

The application uses the following durable queue:

```text
order_queue
```

---

## 2. Product Service

The Product Service from Lab 2 was originally implemented using Rust.

For Lab 3, the service was rewritten using **Python and Flask** so that it could be deployed using a runtime natively supported by Azure App Service.

The service maintains the same functionality as the original Rust implementation.

### Endpoint

```http
GET /products
```

The service returns the following products:

| ID | Product | Price |
| ---: | --- | ---: |
| 1 | Dog Food | $19.99 |
| 2 | Cat Food | $34.99 |
| 3 | Bird Seeds | $10.99 |

The service was first tested locally using the VS Code REST Client and then deployed to Azure App Service through GitHub Actions.

### Azure Endpoint

```text
https://product-service-mwg-e8fmbtgehbdbhfga.swedencentral-01.azurewebsites.net/products
```

Repository:

[Product Service](https://github.com/divine-wander/product-service)

---

## 3. Order Service

The Order Service is implemented using **Node.js and Express**.

It was deployed to Azure App Service using GitHub Actions.

### Endpoint

```http
POST /orders
```

Example request:

```json
{
  "product": "Cat Food"
}
```

The Order Service communicates with RabbitMQ using the Azure App Service application setting:

```text
RABBITMQ_CONNECTION_STRING
```

The connection string is stored securely in the Azure App Service environment configuration and is not committed to GitHub.

When an order is received, the Order Service publishes the message to:

```text
order_queue
```

### Azure Endpoint

```text
https://order-service-mwg-dfc8ftarcpg3h7cb.swedencentral-01.azurewebsites.net/orders
```

Repository:

[Order Service](https://github.com/divine-wander/order-service)

---

## 4. Store Front

The Store Front is implemented using **Vue.js**.

It communicates with both backend services through environment variables:

```text
VUE_APP_ORDER_SERVICE_URL
VUE_APP_PRODUCT_SERVICE_URL
```

The configured values point to the Azure App Service deployments of the Order Service and Product Service.

### Azure Static Web Apps Limitation

The Store Front was originally configured for deployment using Azure Static Web Apps.

During the deployment attempt, Azure returned the following policy error:

```text
RequestDisallowedByAzure
```

The Azure for Students subscription had a regional deployment policy that prevented the creation of the Static Web App.

The instructor provided an approved alternative allowing the Store Front to be hosted on an Azure Virtual Machine when this restriction occurs.

Therefore, the Store Front was deployed on an Ubuntu Azure VM.

### Store Front VM

The Vue application runs on:

```text
Port 8080
```

The Store Front was started using:

```bash
npm run serve -- --host 0.0.0.0
```

The VM Network Security Group permits inbound TCP traffic on port `8080` so the Store Front can be accessed from a web browser.

Repository:

[Store Front](https://github.com/divine-wander/store-front)

---

# Testing and Validation

## Product Service Testing

The Product Service was tested locally using:

```http
GET http://localhost:3030/products
```

It was also tested after deployment using:

```http
GET https://product-service-mwg-e8fmbtgehbdbhfga.swedencentral-01.azurewebsites.net/products
```

The Azure endpoint successfully returned:

```json
[
  {
    "id": 1,
    "name": "Dog Food",
    "price": 19.99
  },
  {
    "id": 2,
    "name": "Cat Food",
    "price": 34.99
  },
  {
    "id": 3,
    "name": "Bird Seeds",
    "price": 10.99
  }
]
```

---

## Order Service Testing

The Order Service was tested locally and through its Azure App Service endpoint.

An Azure request similar to the following was used:

```http
POST https://order-service-mwg-dfc8ftarcpg3h7cb.swedencentral-01.azurewebsites.net/orders
Content-Type: application/json

{
  "product": "Cat Food"
}
```

The service successfully returned:

```text
Order received
```

---

## Full Application Test

The complete application was tested from the Store Front.

A test order was placed for:

```text
Product: Cat Food
Quantity: 2
Price per item: $34.99
Total: $69.98
```

The Store Front displayed the successful confirmation:

```text
Order for 2 x Cat Food placed successfully! Total: $69.98
```

The RabbitMQ Management UI was then used to confirm that the message was successfully added to:

```text
order_queue
```

RabbitMQ showed:

```text
Queues: 1
Ready: 2
Unacked: 0
Total: 2
```

One message had previously been added during REST API testing, while the second message was generated through the Store Front.

This confirmed the complete communication path:

```text
Store Front
     |
     v
Order Service
     |
     v
RabbitMQ
     |
     v
order_queue
```

The Product Service integration was also confirmed:

```text
Store Front
     |
     v
Product Service
     |
     v
Products displayed
```

---

# GitHub Actions

GitHub Actions was used to automatically deploy the Product Service and Order Service to Azure App Service.

The workflow performs the following process:

```text
Push to main
     |
     v
GitHub Actions
     |
     v
Build Application
     |
     v
Authenticate to Azure
     |
     v
Deploy to Azure App Service
```

This allows changes pushed to the service repositories to trigger automated Azure deployments.

---

# Environment Variables

Configuration values were separated from the application source code using environment variables.

## Product Service

```text
PORT
```

## Order Service

```text
PORT
RABBITMQ_CONNECTION_STRING
```

## Store Front

```text
VUE_APP_ORDER_SERVICE_URL
VUE_APP_PRODUCT_SERVICE_URL
```

Sensitive configuration values, including RabbitMQ credentials, were not committed to GitHub.

---

# Deployment Summary

| Component | Technology | Azure Service |
| --- | --- | --- |
| Store Front | Vue.js | Azure Virtual Machine |
| Product Service | Python / Flask | Azure App Service |
| Order Service | Node.js / Express | Azure App Service |
| RabbitMQ | RabbitMQ | Azure Virtual Machine |

---

# Demo Video

The demonstration video shows:

1. The deployed Store Front.
2. Products being retrieved from the Product Service.
3. An order being submitted through the Order Service.
4. The successful order confirmation.
5. RabbitMQ receiving the order in `order_queue`.
6. The Azure resources used to deploy the application.

### YouTube Demo

[Watch Demo Video]()

---

# Reflection Questions

## 1. What challenges did you encounter when configuring environment variables in the GitHub Actions workflow?

One of the main challenges was that I was unable to complete the originally planned Azure Static Web Apps deployment because my Azure for Students subscription blocked the resource through the `RequestDisallowedByAzure` regional deployment policy.

Because the Static Web App could not be created, its GitHub Actions workflow could not be used to configure the frontend environment variables as originally intended.

Following the instructor-approved alternative, I deployed the Store Front on an Azure Virtual Machine. The `VUE_APP_ORDER_SERVICE_URL` and `VUE_APP_PRODUCT_SERVICE_URL` variables were configured using a `.env` file on the VM.

For the backend, the `RABBITMQ_CONNECTION_STRING` was configured securely using the Azure App Service environment variables instead of storing RabbitMQ credentials directly in the source code.

---

## 2. How does deploying microservices on Azure App Service differ from running them locally?

When running the services locally, I am responsible for installing the required runtime and dependencies, starting each service manually, selecting the ports, and ensuring that the processes remain available.

Azure App Service manages much of the underlying infrastructure automatically. Each service receives a public HTTPS endpoint, and Azure manages the operating environment used to run the application.

Azure App Service can also integrate with GitHub Actions, allowing the application to be automatically built and deployed whenever changes are pushed to the GitHub repository.

This reduces the amount of infrastructure administration required compared with running every microservice directly on a virtual machine.

---

## 3. Why is it important to use environment variables for configurations in a cloud environment?

Environment variables separate application configuration from application source code.

This allows values such as API URLs, ports, usernames, and connection strings to change between development and cloud environments without requiring changes to the application code.

Environment variables are also important for security because sensitive values such as passwords and RabbitMQ connection strings should not be hard-coded or committed to public GitHub repositories.

Using environment variables makes applications easier to configure across multiple environments and follows the configuration principles of the 12-Factor App methodology.

---

# Final Result

The Algonquin Pet Store was successfully deployed and tested using Azure.

The final architecture consists of:

```text
Vue.js Store Front
        |
        +--------------------------+
        |                          |
        v                          v
Python Product Service       Node.js Order Service
Azure App Service            Azure App Service
                                   |
                                   v
                              RabbitMQ VM
                                   |
                                   v
                              order_queue
```

The application successfully demonstrates communication between a frontend application, cloud-hosted microservices, and a RabbitMQ message broker.

The Store Front was deployed using the instructor-approved Azure VM alternative because Azure Static Web Apps were blocked by the Azure for Students subscription policy.