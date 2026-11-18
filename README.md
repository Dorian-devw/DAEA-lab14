# 14Serpentarios — Serpentario Bio-Farmacéutico y Banco de Venenos (Semana 14)

Web API REST desarrollada con **ASP.NET Core (.NET 10)**, **Entity Framework Core 10 (Database-First)**, autenticación con **JWT Bearer**, hashing seguro con **BCrypt**, control de acceso por **Roles del Negocio** y documentación interactiva con **Swagger / OpenAPI** (con soporte Bearer Token), siguiendo una arquitectura limpia en cuatro capas.

---

## 1. Descripción del tema y alcance

En el ámbito bio-farmacéutico, el veneno ofídico es un insumo crítico para la producción de **sueros antiofídicos** y la investigación de fármacos (inhibidores enzimáticos, analgésicos y terapias oncológicas).

Esta solución gestiona:
- El inventario taxonómico y sanitario de especies ofídicas (**`EspeciesSerpiente`**).
- El control de trazabilidad, stock en miligramos, cadena de frío, pureza y vigencia de cada lote extraído (**`LotesVeneno`**).
- La gestión de usuarios, credenciales protegidas con hash y autenticación basada en tokens (**`Usuarios`**).

---

## 2. Novedades de la Semana 14: Autenticación JWT y Roles de Negocio

La API se encuentra protegida mediante **JSON Web Tokens (JWT)** y control de acceso basado en roles propios del negocio, cumpliendo con la regla de no emplear roles genéricos (*Admin / Usuario*).

### Roles propios del negocio
1. **`Herpetologo`**: Científico especialista en ofidios y venenos. Autorizado para dar de alta, modificar y eliminar especies en el serpentario, así como gestionar la baja de lotes de veneno.
2. **`TecnicoLaboratorio`**: Profesional de laboratorio responsable del control de calidad, análisis de pureza y parámetros de almacenamiento en cadena de frío de los lotes.

> El servidor decide y asigna el rol de forma exclusiva; el cliente nunca envía el rol en la solicitud de registro.

---

## 3. Matriz de Permisos (Endpoint × Rol)

| Endpoint | Método | Sin token (Público) | Rol `TecnicoLaboratorio` | Rol `Herpetologo` | Descripción / Justificación |
| :--- | :---: | :---: | :---: | :---: | :--- |
| `POST /api/auth/registro` | POST | **Sí (201)** | Sí | Sí | Registro público; el servidor asigna el rol por defecto. |
| `POST /api/auth/login` | POST | **Sí (200)** | Sí | Sí | Autenticación y emisión de token JWT. |
| `GET /api/auth/perfil` | GET | **No (401)** | **Sí (200)** | **Sí (200)** | Perfil obtenido de claims del token (`User.Claims`). |
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

## 4. Respuestas y Códigos de Estado HTTP Implementados

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

Cadena de dependencias:
`13Serpentarios.Api` → `13Serpentarios.Negocio` → `13Serpentarios.Datos` → `13Serpentarios.Entidades`

---

## 6. Guía de Ejecución y Pruebas

1. **Configurar base de datos:**
   - Para SQL Server / LocalDB local: ejecuta [`database/Serpentario13DB.sql`](database/Serpentario13DB.sql) y luego [`database/Serpentario14_Usuarios_JWT.sql`](database/Serpentario14_Usuarios_JWT.sql).
   - Para MonsterASP.NET: ejecuta [`database/Serpentario14DB_MonsterASP.sql`](database/Serpentario14DB_MonsterASP.sql) desde la consola Web SQL.

2. **Compilar y ejecutar la API:**
   ```powershell
   dotnet build
   dotnet run --project 13Serpentarios.Api --launch-profile http
   ```

3. **Probar con Swagger UI:**
   - Abre `http://localhost:5158/swagger` en tu navegador.
   - Da clic en el botón verde **Authorize** (arriba a la derecha).
   - Inicia sesión con `POST /api/auth/login`, copia el token y pégalo en el cuadro de autorización de Swagger.

4. **Probar con archivo HTTP:**
   - Ejecuta las peticiones de prueba en [`13Serpentarios.Api/13Serpentarios.Api.http`](13Serpentarios.Api/13Serpentarios.Api.http).
