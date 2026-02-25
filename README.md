# Modelado

Basándome en el esquema SQL de mi anterior proyecto sobre gestión de incidencias, he desarrollado el siguiente schema design:

![WhatsApp Image 2026-02-23 at 18 35 31](https://github.com/user-attachments/assets/8148e9c1-6a73-4409-aebe-37981ab6bed9)

# Implementación y funcionamiento

Aquí describo como he diseñado esta automatización, qué decisiones he tomado y porqué.

## Servicios

Para llevar a cabo el propósito final de mi implementación, he tenido que usar los siguientes servicios:
 - **n8n**: Estoy utilizando el trial web de 14 días para tener más versatilidad y poder trabajar donde quiera en vez de self-hostearlo.
 - **MongoDB**: Mi base de datos principal. He decidido hostearla en Microsoft Azure.
 - **Redis**: Guarda el cache del chatbot. También está hosteado en Microsoft Azure. 

## Workflow principal - Telegram + Deepseek

<img width="1021" height="429" alt="image" src="https://github.com/user-attachments/assets/592eb398-0653-4efd-85b7-74ca69cd2938" />

El workflow principal es el core del funcionamiento de mi automatización. La idea es muy sencilla: 
 - El **usuario** le envía un mensaje a mi **bot de telegram**.
 - El mensaje es recogido por el trigger de n8n.
 - El contenido de este se envía a un **nodo AI Agent**, configurado con el modelo de deepseek y memoria almacenada en redis.
 - El agente **extrae los datos del mensaje** en lenguaje humano, y los envía a un sub-workflow que creará la incidencia en BBDD en base a los datos proporcionados por la IA.
 - Una vez acabada la lógica de BBDD, se envia una respuesta al usuario via Telegram.

## Sub-Workflow - MongoDB + JavaScript

<img width="1599" height="660" alt="{FAB3E350-6FE5-41F7-93F8-E4CE90A7DE5A}" src="https://github.com/user-attachments/assets/8b7fd600-fc39-43d8-8848-5d32c0a27e4a" />

Este subworkflow tiene como input los parametros basicos de la incidencia y del usuario. Su objetivo es hacer todos los parses y lógica necesarias para insertar tanto el usuario como la incidencia.

 1. **Parsea la entrada** a JSON
 2. Prepara la query y busca en **BBDD si existe un usuario** con ese nombre y apellidos. En caso de que **exista** simplemente inserta la incidencia referenciando su ID.
 3. Si **NO existe** el usuario lo crea generando el username en base a su nombre y apellido, comprobando que no exista otro username igual.
 4. Inserta la incidencia.
