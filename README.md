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

## Contrato de URLs del catálogo de lugares

El microservicio de lugares expone sus endpoints bajo `api/lugares`. Como el gateway le
asigna el prefijo `/lugares`, la URL pública termina con un segmento duplicado
(`/lugares/api/lugares/...`). Es la misma convención ya usada por `usuarios`
(`/usuarios/auth/...`), y se mantiene por decisión de no modificar la configuración.

Flujo de una petición (frontend → gateway → lugares-api):

| Frontend (vite / nginx) | Gateway recibe | Forward a `lugares-api` |
| --- | --- | --- |
| `GET /api/lugares/api/lugares` | `/lugares/api/lugares` | `GET /api/lugares` |
| `GET /api/lugares/api/lugares/{id}` | `/lugares/api/lugares/{id}` | `GET /api/lugares/{id}` |
| `GET /api/lugares/api/lugares/categorias` | `/lugares/api/lugares/categorias` | `GET /api/lugares/categorias` |

### Filtros del catálogo

El endpoint `GET /api/lugares` acepta los siguientes query params (mapean al
`LugarFiltroDto`):

| Query param | Tipo | Descripción |
| --- | --- | --- |
| `barrio` | enum | Filtra por barrio |
| `tipoLugar` | enum | Filtra por tipo de lugar |
| `categoriaCodigo` | string | Código lógico de categoría |
| `soloBajoTecho` | bool | Solo lugares bajo techo (modo lluvia) |
| `soloAlAireLibre` | bool | Solo al aire libre |
| `aptoNinos` | bool | Apto para niños |
| `accesibleMovilidadReducida` | bool | Accesible para movilidad reducida |
| `ofreceMenuInfantil` | bool | Ofrece menú infantil |
| `restriccionDietaria` | enum | Filtra por opción dietaria disponible |
| `nivelPrecio` | short | Nivel de precio máximo (1 a 3) |
| `busqueda` | string | Búsqueda libre en nombre/descripción/subzona |
| `latitudReferencia` | decimal | Referencia para ordenar por cercanía |
| `longitudReferencia` | decimal | Referencia para ordenar por cercanía |
| `radioMetrosMaximo` | double | Radio máximo en metros (con lat/lon) |
| `pagina` | int | Número de página (default `1`) |
| `tamano` | int | Tamaño de página, máx `100` (default `20`) |

La respuesta de búsqueda es `ResultadoPaginadoDto`:

```json
{
  "items": [],
  "total": 0,
  "pagina": 1,
  "tamano": 20
}
```

### Valores de enums

La API serializa los enums como string en PascalCase (`JsonStringEnumConverter`). El
binding de query params es case-insensitive, por lo que el frontend puede enviar los
mismos valores en minúscula.

| Enum | Valores |
| --- | --- |
| `BarrioCaba` | `Palermo`, `Recoleta`, `SanTelmo`, `SanNicolas`, `PuertoMadero` |
| `TipoLugar` | `PuntoInteres`, `Alojamiento`, `NodoUrbano` |
| `RestriccionDietaria` | `Celiaco`, `Vegano`, `Vegetariano`, `IntoleranteLactosa` |
| `OrigenLugar` (solo respuesta) | `CuradoLocal`, `GooglePlaces` |

### Categorías (seed)

| Código (`categoriaCodigo`) | Nombre |
| --- | --- |
| `cafe_especialidad` | Café de especialidad |
| `museos_galerias` | Museos y galerías |
| `parques_aire_libre` | Parques y aire libre |
| `librerias_diseno` | Librerías y diseño |
| `bares_vermu` | Bares y vermú |
| `gastronomia_clasica` | Gastronomía clásica/Bodegón |
| `cocteleria_autor` | Coctelería de autor |

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
