# TouringGo Gateway

API Gateway de TouringGO basado en [YARP](https://microsoft.github.io/reverse-proxy/)
(Yet Another Reverse Proxy). Centraliza el acceso a los microservicios del sistema y
enruta cada prefijo de ruta al servicio correspondiente.

## Tecnologías

- .NET 8
- ASP.NET Core
- YARP (Yarp.ReverseProxy)
- Docker

## Estructura

```text
src/
└── TouringGo.Gateway/
    ├── Program.cs          # Configuración del Reverse Proxy (AddReverseProxy / LoadFromConfig)
    ├── appsettings.json    # Definición de rutas y clusters (sección "ReverseProxy")
    └── Dockerfile
```

## Rutas

Cada ruta se define en `appsettings.json`, sección `ReverseProxy:Routes`. Las rutas
hacen coincidir un prefijo de URL y lo despachan a un cluster.

| Ruta (`Routes`) | Cluster | Match (`Path`) | Transforms |
| --- | --- | --- | --- |
| `usuarios-route` | `usuarios-cluster` | `/usuarios/{**remainder}` | `PathRemovePrefix: /usuarios`, `RequestHeaderOriginalHost: true` |
| `lugares-route` | `lugares-cluster` | `/lugares/{**remainder}` | `PathRemovePrefix: /lugares` |
| `itinerarios-route` | `itinerarios-cluster` | `/itinerarios/{**remainder}` | `PathRemovePrefix: /itinerarios` |
| `usuarios-google-callback-route` | `usuarios-cluster` | `/signin-google` | `RequestHeaderOriginalHost: true`, headers `X-Forwarded-Host`/`X-Forwarded-Proto` |
| `usuarios-app-callback-route` | `usuarios-cluster` | `/auth/google/callback` | `RequestHeaderOriginalHost: true` |

> `usuarios-google-callback-route` y `usuarios-app-callback-route` existen para el flujo
> de autenticación con Google OAuth: preservan el host original para que el redirect de
> Google vuelva al gateway correctamente.

## Clusters

Cada cluster se define en `appsettings.json`, sección `ReverseProxy:Clusters`, y apunta
al destino interno de cada microservicio dentro de la red de Docker.

| Cluster | Destino (Address) |
| --- | --- |
| `usuarios-cluster` | `http://usuarios-api:8080/` |
| `lugares-cluster` | `http://lugares-api:8080/` |
| `itinerarios-cluster` | `http://itinerarios-api:8080/` |

Los nombres de host (`usuarios-api`, `lugares-api`, `itinerarios-api`) corresponden a los
nombres de servicio definidos en el `compose.yaml` del repositorio
`touringgo-infraestructura`.

## Ejecución

El gateway se levanta junto con el resto de servicios desde `touringgo-infraestructura`:

```powershell
docker compose --env-file local.env up -d --build
```

El gateway expone el puerto `8000` (mapeado al `8080` interno):

```text
http://localhost:8000
```

Para construir y ejecutar únicamente el gateway:

```powershell
docker build -t touringgo-gateway .
docker run --rm -p 8000:8080 touringgo-gateway
```
