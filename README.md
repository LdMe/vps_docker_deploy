# Despliegue en VPS con Docker

Guía completa para desplegar proyectos web en un servidor propio usando Docker, nginx como reverse proxy y certificados HTTPS automáticos con Let's Encrypt.

Está orientada a desarrolladores que conocen la terminal y han trabajado con Node o herramientas similares, pero que nunca han configurado un servidor remoto. No se asume experiencia previa con Linux, Docker ni redes.

---

## Contenido

| # | Capítulo | Qué cubre |
|---|----------|-----------|
| 01 | Presentación | Para quién es la guía y cómo seguirla |
| 02 | Introducción y arquitectura | Qué vamos a montar y por qué |
| 03 | Conceptos previos | VPS, DNS, SSH, Docker, reverse proxy, HTTPS |
| 04 | Conexión al servidor y claves SSH | Primera conexión, autenticación por clave, alias en `~/.ssh/config` |
| 05 | Instalación de Docker | Docker Engine y Docker Compose en Ubuntu |
| 06 | Configurar nginx-proxy y HTTPS | Reverse proxy, Let's Encrypt, configuración avanzada de nginx |
| 07 | Desplegar un frontend (React/Vite) | Compilación, `scp`, nginx estático, React Router |
| 08 | Desplegar un backend con base de datos | Node + MySQL/PostgreSQL/MongoDB, redes internas, backups, cron |
| 09 | Troubleshooting | Diagnóstico de los problemas más comunes |
| 10 | Migrar a un VPS nuevo | Transferencia sin pérdida de datos y con mínimo downtime |

---

## Qué se consigue al seguir la guía

- Un servidor con dominio propio (`https://midominio.com`) y HTTPS automático y gratuito.
- Capacidad para alojar varios proyectos en el mismo servidor, cada uno en su subdominio, sin reconfigurar el proxy al añadir uno nuevo.
- Flujo de actualización sencillo: `git pull` + `docker compose up -d --build`.
- Backups de base de datos y procedimiento de migración a otro servidor.

---

## Arquitectura resultante

```
                        INTERNET
                            │
                     (puerto 80 / 443)
                            │
                            ▼
 ┌────────────────────────────────────────────────────┐
 │                    VPS                             │
 │                                                    │
 │   ┌─────────────────────────────────────────┐      │
 │   │  nginx-proxy + acme-companion            │      │
 │   └─────────────────────────────────────────┘      │
 │        │               │               │           │
 │        ▼               ▼               ▼           │
 │   ┌─────────┐    ┌──────────┐    ┌─────────┐       │
 │   │ Frontend│    │ Backend  │    │  Otros  │       │
 │   │  React  │    │  Node    │    │proyectos│       │
 │   └─────────┘    │    │     │    └─────────┘       │
 │                  │    ▼     │                      │
 │                  │  ┌────┐  │                      │
 │                  │  │ DB │  │                      │
 │                  │  └────┘  │                      │
 │                  └──────────┘                      │
 └────────────────────────────────────────────────────┘
```

El proxy recibe todo el tráfico y lo distribuye según el dominio. Cada proyecto vive en su propio contenedor Docker, aislado del resto. La base de datos está en una red interna: no es accesible desde internet.

---

## Requisitos previos

- Una tarjeta para contratar un VPS (desde 3-5 €/mes).
- Un dominio registrado en cualquier proveedor (OVH, Namecheap, Porkbun, etc.).
- Un ordenador con terminal y acceso a internet.
- Familiaridad básica con la terminal: `cd`, `ls`, `npm install`.

El VPS puede ser de cualquier proveedor. Se recomienda solicitar Ubuntu 22.04 o 24.04 con al menos 1 vCPU, 2 GB de RAM y 20 GB de disco.

---

## Cómo usar esta guía

Sigue los capítulos en orden la primera vez. Cada uno asume que el anterior está completado.

Una vez desplegado el primer proyecto, los capítulos 07 y 08 se pueden usar de forma independiente para añadir proyectos nuevos. El capítulo 09 es de consulta puntual cuando algo no funciona.

---

## Tecnologías utilizadas

- **Docker** y **Docker Compose** para contenerización y orquestación.
- **[nginxproxy/nginx-proxy](https://github.com/nginx-proxy/nginx-proxy)** como reverse proxy automático.
- **[nginxproxy/acme-companion](https://github.com/nginx-proxy/acme-companion)** para gestión automática de certificados Let's Encrypt.
- **nginx** para servir frontends estáticos.
- Ubuntu 22.04 / 24.04 como sistema operativo del servidor.

---

## Notas sobre la configuración del proxy

La guía usa `nginxproxy/nginx-proxy` (sucesor oficial de la antigua imagen `jwilder/nginx-proxy`). El volumen `vhost.d` se monta como carpeta local, lo que permite personalizar el comportamiento de nginx por dominio editando archivos de texto, sin tocar el contenedor. El capítulo 06 cubre los casos más habituales: timeouts para APIs lentas, límite de tamaño de subida y streaming de respuestas.
