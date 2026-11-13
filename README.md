# 🔐 TaskBoard — Semana 14

![Laravel](https://img.shields.io/badge/Laravel-12-red?logo=laravel)
![PHP](https://img.shields.io/badge/PHP-8.2-blue?logo=php)
![MySQL](https://img.shields.io/badge/MySQL-MariaDB-orange?logo=mysql)
![Sanctum](https://img.shields.io/badge/Laravel-Sanctum-red)
![API](https://img.shields.io/badge/API-REST-0B2E59)
![UPED](https://img.shields.io/badge/UPED-Integración_de_Sistemas-0B2E59)

Proyecto desarrollado para la asignatura **Integración de Sistemas** de la **Universidad Pedagógica de El Salvador (UPED)**.

Durante la Semana 14 se protegió la API REST de TaskBoard utilizando **Laravel Sanctum**, agregando autenticación mediante tokens personales, aislamiento de datos por usuario, logout y administración de sesiones abiertas.

**Estudiante:** Josthyn Stanley Cruz Vásquez  
**Carrera:** Ingeniería en Sistemas y Computación  
**Universidad:** Universidad Pedagógica de El Salvador  
**Ciclo:** 02-2026  

---

## 🎯 Objetivo de la semana

Convertir la API abierta de la Semana 13 en una API protegida capaz de identificar al usuario mediante tokens personales, proteger rutas con `auth:sanctum`, filtrar datos por propietario, revocar tokens y administrar sesiones abiertas por dispositivo.

---

## 🚪 Estado inicial: API abierta

Antes de agregar Sanctum se comprobó que la API permitía acciones sin autenticación:

```text
GET /api/comercios
GET /api/transacciones
POST /api/transacciones
DELETE /api/transacciones/{id}
```

Esto demostró la necesidad de proteger la API.

---

## 🛡️ Laravel Sanctum

Se verificó Sanctum con:

```bash
composer show laravel/sanctum
php artisan migrate:status
```

y la existencia de la tabla:

```text
personal_access_tokens
```

---

## 👤 HasApiTokens en User

El modelo `User` utiliza:

```php
use Laravel\Sanctum\HasApiTokens;
```

y:

```php
use HasApiTokens, HasFactory, Notifiable;
```

Esto habilita `createToken()`, `tokens()` y `currentAccessToken()`.

---

## 🔑 Login de API

Se creó:

```text
app/Http/Controllers/Api/AuthApiController.php
```

El login valida correo, contraseña y dispositivo.

Una autenticación correcta responde:

```text
201 Created
```

y devuelve un token Bearer.

Una contraseña incorrecta responde:

```text
422 Unprocessable Content
```

---

## 🔒 Rutas protegidas con auth:sanctum

El login permanece público:

```php
Route::post('/login', [AuthApiController::class, 'login'])
    ->middleware('throttle:5,1');
```

Las demás rutas están dentro de:

```php
Route::middleware('auth:sanctum')->group(function () {
    // rutas protegidas
});
```

Entre ellas:

```text
GET    /api/yo
POST   /api/logout
GET    /api/comercios
GET    /api/comercios/{comercio}
GET    /api/transacciones
POST   /api/transacciones
GET    /api/transacciones/{transaccion}
DELETE /api/transacciones/{transaccion}
GET    /api/tokens
DELETE /api/tokens/{id}
```

---

## 🙋 Endpoint /api/yo

`GET /api/yo` permite comprobar qué usuario pertenece al token.

Con token válido:

```text
200 OK
```

Sin token:

```text
401 Unauthorized
```

---

## 📡 Accept: application/json

Se comprobó que:

```text
Sin token + Accept: application/json → 401
Sin token + sin Accept              → 302
```

Esto demuestra la importancia de indicar que el cliente espera JSON.

---

## 🏪 Aislamiento de comercios

Los comercios se filtran mediante:

```php
$request->user()
    ->comercios()
    ->orderBy('nombre_comercio')
    ->get();
```

Además, el detalle de un comercio ajeno se protege con:

```php
abort_unless(
    $comercio->user_id === $request->user()->id,
    404
);
```

Se comprobó que Ana solo ve sus comercios y Carlos solo los suyos.

---

## 💳 Protección de transacciones

Las transacciones se filtran por el usuario dueño del comercio:

```php
Transaccion::whereHas(
    'comercio',
    fn ($q) => $q->where(
        'user_id',
        $request->user()->id
    )
)
```

También se agregó una validación de propiedad para bloquear acceso a transacciones ajenas.

---

## ✅ Validación al crear transacciones

`GuardarTransaccionApiRequest` valida que el comercio pertenezca al usuario autenticado mediante `Rule::exists()` y `user_id`.

Si Ana intenta crear una transacción en un comercio de Carlos, la API responde:

```text
422 Unprocessable Content
```

---

## 🚪 Logout y revocación del token

El logout elimina únicamente el token actual:

```php
$request->user()
    ->currentAccessToken()
    ->delete();
```

Ciclo comprobado:

```text
GET /api/yo      → 200
POST /api/logout → 204
GET /api/yo      → 401
```

---

## ⏳ Expiración de tokens

En `config/sanctum.php` se configuró:

```php
'expiration' => 60 * 24,
```

equivalente a 24 horas.

---

## 📱 Sesiones abiertas

Se agregó:

```text
GET /api/tokens
```

para listar sesiones abiertas sin exponer el hash del token.

También:

```text
DELETE /api/tokens/{id}
```

para cerrar una sesión específica.

Una sesión eliminada deja de funcionar y responde:

```text
401 Unauthorized
```

si se intenta reutilizar su token.

---

## ✅ Códigos HTTP comprobados

| Código | Significado | Ejemplo |
|---|---|---|
| `200` | Petición correcta | `/api/yo`, comercios y transacciones |
| `201` | Token creado | Login correcto |
| `204` | Sin contenido | Logout / eliminar token |
| `302` | Redirección | Sin `Accept` |
| `401` | No autenticado | Sin token o token revocado |
| `404` | Recurso ajeno | Comercio/transacción de otro usuario |
| `422` | Validación fallida | Credenciales o comercio inválido |

---

## 🧪 Pruebas realizadas

- API abierta antes de Sanctum.
- Login correcto de Ana.
- Login correcto de Carlos.
- Login incorrecto.
- `GET /api/yo`.
- Peticiones con y sin token.
- Peticiones con y sin `Accept`.
- Comercios filtrados por usuario.
- Bloqueo de comercios ajenos.
- Transacciones filtradas por usuario.
- Bloqueo de transacciones ajenas.
- Validación de comercio al crear transacciones.
- Logout.
- Reutilización de token revocado.
- Consulta de sesiones abiertas.
- Eliminación de una sesión concreta.

---

## 🔐 Buenas prácticas

- No publicar tokens reales.
- No colocar tokens en URLs.
- Usar `Bearer Token`.
- Configurar expiración.
- Revocar tokens al cerrar sesión.
- Usar HTTPS en producción.
- No exponer hashes de `personal_access_tokens`.
- Filtrar datos por usuario.

---

## 🛠️ Tecnologías utilizadas

- Laravel 12
- Laravel Sanctum
- PHP 8.2
- MySQL / MariaDB
- Eloquent ORM
- REST
- JSON
- Insomnia
- Git
- GitHub
- XAMPP

---

## ▶️ Ejecución

```bash
git clone https://github.com/StanleyJv/taskboard-semana14.git
cd taskboard-semana14
composer install
php artisan key:generate
php artisan migrate:fresh --seed
php artisan serve
```

La API estará disponible en:

```text
http://127.0.0.1:8000/api
```

---

## 🔗 Repositorio

https://github.com/StanleyJv/taskboard-semana14

---

**Integración de Sistemas · Ciclo 02-2026 · UPED**
