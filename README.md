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

# Consultas / Aggregation Framework

## Métodos CRUD

#### Insertar nueva incidencia abierta

```javascript
db.incidencias.insertOne({
  estado: "ABIERTAS",
  descripcion: "El monitor parpadea constantemente.",
  IP: "192.168.1.50",
  tipo: "hardware",
  momento: new Date(),
  usuario_que_la_abrio: ObjectId("65d8a1b2c3e4f5a6b7c8d9e0"),
  tecnico_asignado: null,
  historial_tecnicos: [],
  solucion: null,
  motivo_cierre: null
})
```

#### Actualizar una incidencia (ponerla en proceso y agregarle un tecnico)

```javascript
// Opcional: parámetro definido por el usuario
var idIncidenciaAModificar = ObjectId("65d8a2f1c3e4f5a6b7c8d9f7"); 

db.incidencias.updateOne(
  { _id: idIncidenciaAModificar },
  { 
    $set: { 
      estado: "EN PROCESO", 
      tecnico_asignado: ObjectId("65d8a1b2c3e4f5a6b7c8d9e1") 
    },
    $push: { 
      historial_tecnicos: {
        tecnico_id: ObjectId("65d8a1b2c3e4f5a6b7c8d9e1"),
        fecha_asignacion: new Date(),
        notas: "Asignación manual desde administración."
      }
    }
  }
)
```

#### Borrado de una incidencia

```javascript
var idIncidenciaABorrar = ObjectId("65d8a2f1c3e4f5a6b7c8d9f7");

db.incidencias.deleteOne({ _id: idIncidenciaABorrar })
```

## Consultas

#### Buscar todas las incidencias por estado (ej. "ABIERTAS")

```javascript
var parametroEstado = "ABIERTAS";
db.incidencias.find({ estado: parametroEstado })
```

#### Buscar incidencias de un usuario concreto ordenadas por la más reciente

```javascript
var parametroUsuarioId = ObjectId("65d8a1b2c3e4f5a6b7c8d9e0");
db.incidencias.find({ usuario_que_la_abrio: parametroUsuarioId }).sort({ momento: -1 })
```

#### Buscar incidencias que contengan una palabra específica en la descripción (búsqueda con regex)

```javascript
var palabraClave = "VPN";
db.incidencias.find({ descripcion: { $regex: palabraClave, $options: "i" } })
```

## Aggregation Frameworks

#### Productividad de los técnicos: Número de incidencias resueltas por cada técnico, mostrando su nombre real

```javascript
db.incidencias.aggregate([
  { $match: { estado: { $in: ["RESUELTAS", "CERRADAS"] } } },
  { $group: { _id: "$tecnico_asignado", total_solucionadas: { $sum: 1 } } },
  { $lookup: { 
      from: "usuarios", 
      localField: "_id", 
      foreignField: "_id", 
      as: "datos_tecnico" 
  }},
  { $unwind: "$datos_tecnico" },
  { $project: { 
      _id: 0, 
      tecnico: { $concat: ["$datos_tecnico.nombre", " ", "$datos_tecnico.apellidos"] }, 
      total_solucionadas: 1 
  }},
  { $sort: { total_solucionadas: -1 } }
])
```

#### Análisis de reasignaciones: Mostrar las incidencias que han sido pasadas de un técnico a otro más de X veces

```javascript
var minimoReasignaciones = 1;

db.incidencias.aggregate([
  { $match: { estado: { $ne: "ABIERTAS" } } }, 
  { $unwind: "$historial_tecnicos" },
  { $group: { 
      _id: "$_id", 
      descripcion: { $first: "$descripcion" }, 
      num_reasignaciones: { $sum: 1 } 
  }},
  { $match: { num_reasignaciones: { $gt: minimoReasignaciones } } },
  { $sort: { num_reasignaciones: -1 } }
])
```

#### Estadísticas de tipos de incidencias creadas por un TIPO de usuario específico (ej: "ADMINISTRADOR")

```javascript
var parametroTipoUsuario = "ADMINISTRADOR";

db.incidencias.aggregate([
  { $lookup: { 
      from: "usuarios", 
      localField: "usuario_que_la_abrio", 
      foreignField: "_id", 
      as: "info_usuario" 
  }},
  { $unwind: "$info_usuario" },
  { $match: { "info_usuario.tipo": parametroTipoUsuario } },
  { $group: { 
      _id: { tipo_incidencia: "$tipo", estado: "$estado" }, 
      cantidad: { $sum: 1 } 
  }},
  { $sort: { cantidad: -1 } }
])
```

# Vídeo explicativo

[Click aquí](https://drive.google.com/file/d/1xzoomsgH3dnct47OorwLuWTCLbBhgNmG/view?usp=sharing)

# Conclusión

Las automatizaciones con inteligencia artificial sin duda ahorran mucho trabajo y son fáciles de desplegar, este supuesto por ejemplo aunque no esté preparado para una aplicación real me ha llevado unas pocas horas para ponerlo a funcionar de forma básica. Las automatizaciones como tal son una cosa que ya había hecho y que ya existía desde hace muchos años cuando se hacían con python o javascript en una VPS, pero con n8n, es muy fácil hacerse al sistema de los nodos y sobre todo es muy fácil trasladar el conocimiento de automaticaciones a la plataforma. La única pega que le veo, es que tiene muchas inconsistencias sobre todo el tema de la inteligencia artificial, aunque estemos tomando de base el uso de una gratuita. Los datos se mezclan, no viajan bien, y el formato tiene que estar especificado con flechas y circulos para que no falle nunca, y aún así, tiene inconsistencias si no eres lo suficientemente específico.

En conclusión, me ha gustado hacer este trabajo, me parece algo aplicable incluso a mis proyectos personales, donde ya tenía alguna automatización pensada para hacer que seguramente lleve a cabo después de este primer contacto.
