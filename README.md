# 14Serpentarios — Serpentario Bio-Farmacéutico y Banco de Venenos (Semana 14)

Web API REST desarrollada con **ASP.NET Core (.NET 10)**, **Entity Framework Core 10 (Database-First)**, autenticación con **JWT Bearer**, hashing seguro con **BCrypt**, control de acceso por **Roles del Negocio** y documentación interactiva con **Swagger / OpenAPI** (con botón Authorize), siguiendo una arquitectura limpia en cuatro capas.

---

## 1. Novedades de la Semana 14: Autenticación JWT y Roles de Negocio

En esta entrega se asegura la API mediante **JSON Web Tokens (JWT)** y control de acceso basado en roles propios del negocio, cumpliendo con la regla de no utilizar roles genéricos (*Admin / Usuario*).

### Roles propios del negocio
1. **`Herpetologo`**: Científico especialista en ofidios y venenos. Tiene privilegios para el alta, edición y baja de especies en el serpentario, así como la baja y extracción de lotes de veneno.
2. **`TecnicoLaboratorio`**: Profesional de laboratorio responsable del análisis químico, pureza, control de calidad y parámetros de almacenamiento en cadena de frío de los lotes de veneno.

> El servidor decide y asigna el rol de forma exclusiva; el cliente nunca envía el rol en la solicitud de registro.

---

## 2. Matriz de Permisos (Endpoint × Rol)

| Endpoint | Método | Sin token (Público) | Rol `TecnicoLaboratorio` | Rol `Herpetologo` | Descripción / Justificación |
| :--- | :---: | :---: | :---: | :---: | :--- |
| `POST /api/auth/registro` | POST | **Sí (201)** | Sí | Sí | Registro público; el servidor asigna el rol por defecto. |
| `POST /api/auth/login` | POST | **Sí (200)** | Sí | Sí | Autenticación y emisión de token JWT. |
| `GET /api/auth/perfil` | GET | **No (401)** | **Sí (200)** | **Sí (200)** | Perfil a partir de claims del token (`User.Claims`). |
| `GET /api/especies` | GET | **Sí (200)** | Sí (200) | Sí (200) | Catálogo taxonómico público de especies ofídicas. |
| `GET /api/especies/{id}` | GET | **Sí (200)** | Sí (200) | Sí (200) | Ficha pública de especie por identificador. |
| `GET /api/especies/{id}/lotes` | GET | **No (401)** | **Sí (200)** | **Sí (200)** | Lotes asociados; información interna del serpentario. |
| `POST /api/especies` | POST | **No (401)** | **No (403)** | **Sí (201)** | Exclusivo del `Herpetologo` para dar de alta taxones. |
| `PUT /api/especies/{id}` | PUT | **No (401)** | **No (403)** | **Sí (204)** | Exclusivo del `Herpetologo` para modificar datos taxonómicos. |
| `DELETE /api/especies/{id}` | DELETE | **No (401)** | **No (403)** | **Sí (204)** | Exclusivo del `Herpetologo` para eliminar especies. |
| `GET /api/lotesveneno` | GET | **No (401)** | **Sí (200)** | **Sí (200)** | Trazabilidad del banco de venenos (personal interno). |
| `GET /api/lotesveneno/{id}` | GET | **No (401)** | **Sí (200)** | **Sí (200)** | Detalle técnico del lote por personal interno. |
| `POST /api/lotesveneno` | POST | **No (401)** | **Sí (201)** | **Sí (201)** | Ambos roles pueden ingresar lotes extraídos. |
| `PUT /api/lotesveneno/{id}` | PUT | **No (401)** | **Sí (204)** | **No (403)** | Control de calidad y pureza exclusivo del laboratorio. |
| `DELETE /api/lotesveneno/{id}` | DELETE | **No (401)** | **No (403)** | **Sí (204)** | Baja y descarte de lote exclusivo del herpetólogo. |

---

## 3. Respuestas y Códigos de Estado HTTP Implementados

| Código | Método / Resultado | Endpoint de ejemplo | Causa |
| :---: | :--- | :--- | :--- |
| **200** | `Ok(...)` | `GET /api/especies`, `POST /api/auth/login`, `GET /api/auth/perfil` | Consulta exitosa o inicio de sesión con JWT generado |
| **201** | `StatusCode(201, ...)` / `CreatedAtAction(...)` | `POST /api/auth/registro`, `POST /api/especies` | Registro de usuario o creación exitosa de entidad |
| **204** | `NoContent()` | `PUT /api/especies/{id}`, `DELETE /api/especies/{id}` | Actualización o eliminación exitosa sin cuerpo de retorno |
| **400** | `BadRequest(ModelState)` | `POST /api/auth/registro`, `POST /api/especies` | Parámetros inválidos por Data Annotations |
| **401** | `Unauthorized(...)` | `GET /api/auth/perfil` sin token, `POST /api/auth/login` con clave errónea | Token ausente/expirado o credenciales inválidas |
| **403** | `Forbid()` automático | `POST /api/especies` con token de `TecnicoLaboratorio` | Token válido pero sin el rol requerido por el endpoint |
| **404** | `NotFound(...)` | `GET /api/especies/9999` | Recurso no encontrado (`RecursoNoEncontradoException`) |
| **409** | `Conflict(...)` | `POST /api/auth/registro` (correo repetido), `DELETE` especie con lotes | Regla de negocio incumplida (`ReglaNegocioException`) |

---

## 4. Sustentación de Seguridad

### 1. En tus pruebas, ¿por qué un caso devolvió 401 y otro 403? Cita un ejemplo de cada uno.
- **`401 Unauthorized` (Falta de autenticación o credenciales no válidas):**
  - *Ejemplo 1:* Al consultar `GET /api/auth/perfil` o `GET /api/lotesveneno` sin incluir la cabecera `Authorization: Bearer <token>`. El servidor no puede verificar la identidad del solicitante.
  - *Ejemplo 2:* Al enviar credenciales incorrectas en `POST /api/auth/login`. El servidor responde 401 indistintamente si el correo no existe o la contraseña no coincide, evitando ataques de enumeración de usuarios.
- **`403 Forbidden` (Autenticado pero sin autorización para el recurso):**
  - *Ejemplo:* El usuario inicia sesión correctamente como `TecnicoLaboratorio` y obtiene un token válido. Sin embargo, al intentar ejecutar `POST /api/especies` o `DELETE /api/especies/1`, el filtro `[Authorize(Roles = "Herpetologo")]` detecta que el claim `role` no coincide y deniega el acceso con 403.

### 2. ¿Qué claims incluiste en el token y por qué? ¿Qué no pondrías nunca dentro de un token?
- **Claims incluidos en el token:**
  - `sub` (`JwtRegisteredClaimNames.Sub`): ID único del usuario (`UsuarioId`) para identificarlo en el sistema.
  - `email` (`JwtRegisteredClaimNames.Email`): Correo electrónico del usuario.
  - `role` / `ClaimTypes.Role`: Rol del usuario (`Herpetologo` o `TecnicoLaboratorio`) para que ASP.NET Core aplique el control de accesos basado en roles.
  - `jti` (`JwtRegisteredClaimNames.Jti`): Identificador único por cada token emitido para trazabilidad y mitigación de ataques de repetición.
- **Qué NO pondrías nunca:**
  - **Contraseñas o hashes:** La carga útil (*payload*) del JWT solo está codificada en Base64Url y cualquier persona puede decodificarla y leerla con herramientas como jwt.io.
  - **Datos sensibles o confidenciales:** Números de tarjetas de crédito, datos biométricos, DNI, direcciones personales o llaves de cifrado privadas.
  - **Tokens de refresco o credenciales de terceros:** Deben custodiarse en almacenamiento seguro del servidor (o cookies `HttpOnly`), nunca en el payload del JWT.

### 3. ¿Qué pasaría si guardaras la contraseña en texto en vez de su hash?
Si ocurriera una filtración de base de datos (por inyección SQL, respaldo expuesto o acceso indebido de administradores), todas las contraseñas de los usuarios quedarían inmediatamente al descubierto. Dado que la gran mayoría de usuarios reutiliza contraseñas en múltiples servicios, los atacantes podrían comprometer correos personales, cuentas bancarias y sistemas corporativos. Utilizar **BCrypt** garantiza que la contraseña se guarde con una función unidireccional de retardo (*cost factor*) y un *salt* criptográfico aleatorio, haciendo inviables los ataques de fuerza bruta y tablas arcoíris (*rainbow tables*).

### 4. ¿Por qué el rol sale del token firmado y no lo envía el cliente en el registro?
Si el cliente pudiera enviar su rol en el registro o en cada petición HTTP, cualquier usuario malicioso podría interceptar la petición y asignarse el rol `Herpetologo` para acceder a operaciones restringidas. Al emitir el servidor el token firmado con una clave secreta y algoritmo HMAC-SHA256, el servidor es quien define el rol en la base de datos y lo incrusta en el token. Si un cliente altera el payload del token, la firma criptográfica se invalida y ASP.NET Core rechaza la petición de inmediato.

---

## 5. Arquitectura en Capas

```text
14Serpentarios (DAEA-sem14/13Serpentarios)
├── 13Serpentarios.Entidades
│   ├── Models/               (EspeciesSerpiente, LotesVeneno, Usuario)
│   └── DTOs/                 (Especie DTOs, Lote DTOs, UsuarioLoginDto, UsuarioRegistroDto, TokenDto, UsuarioPerfilDto)
├── 13Serpentarios.Datos
│   ├── Models/               (SerpentarioContext con DbSet<Usuario>)
│   └── Repositorios/         (IEspecie..., ILote..., IUsuarioRepositorio, UsuarioRepositorio)
├── 13Serpentarios.Negocio
│   ├── Excepciones/          (RecursoNoEncontradoException, ReglaNegocioException)
│   ├── Mappings/             (MappingProfile)
│   └── Servicios/            (Especie..., Lote..., IAuthServicio, AuthServicio con BCrypt y JWT)
└── 13Serpentarios.Api
    ├── Controllers/          (EspeciesController, LotesVenenoController, AuthController)
    ├── Program.cs            (AddAuthentication con JwtBearer, SwaggerUI con botón Authorize)
    ├── appsettings.json      (Sección Jwt con Key de 32+ caracteres, Issuer, Audience)
    └── 13Serpentarios.Api.http (Suite completa de pruebas de todos los códigos HTTP)
```

---

## 6. Guía de Ejecución y Pruebas

1. **Crear tabla y usuarios semilla:**
   Ejecuta el script SQL en SQL Server o LocalDB:
   ```powershell
   sqlcmd -S "(localdb)\MSSQLLocalDB" -i database\Serpentario14_Usuarios_JWT.sql
   ```
   *Usuarios semilla creados:*
   - Herpetólogo: `herpetologo@serpentario.pe` / `Clave123*`
   - Técnico de Laboratorio: `laboratorio@serpentario.pe` / `Clave123*`

2. **Compilar y ejecutar la API:**
   ```powershell
   dotnet build
   dotnet run --project 13Serpentarios.Api --launch-profile http
   ```

3. **Probar con Swagger UI:**
   - Abre `http://localhost:5158/swagger` en tu navegador.
   - Da clic en el botón verde **Authorize** (arriba a la derecha).
   - Inicia sesión con `POST /api/auth/login`, copia el valor del token y pégalo en el cuadro de autorización de Swagger.

4. **Probar con archivo HTTP:**
   - Ejecuta las pruebas en [`13Serpentarios.Api/13Serpentarios.Api.http`](13Serpentarios.Api/13Serpentarios.Api.http) con la extensión REST Client o Thunder Client.
