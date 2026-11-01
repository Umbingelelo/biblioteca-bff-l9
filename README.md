# biblioteca-bff

El BFF del proyecto guía de DSY1107, **tal como queda al terminar L8**, en su propio repositorio. NestJS 12,
puerto **3000**. Arma el panel de préstamos de un lector con una sola llamada: pide libros y préstamos en
paralelo, cruza los datos, filtra por dueño y valida las escrituras. No publica eventos: autoriza y reenvía
`Authorization` a `biblioteca-prestamos`, que es quien escribe y publica.

## Qué hay en cada archivo

| Archivo | Qué es |
|---|---|
| `src/main.ts` | Arranque de Nest, puerto **3000** |
| `src/app.module.ts` | Módulo raíz: `ConfigModule` global más `PanelModule` |
| `src/auth/jwt.guard.ts` | `JwtGuard`: verifica firma, `iss`, `token_use` y `client_id` contra el JWKS de Cognito, y deja `req.usuario` con `sub`, `scope` y `grupos` |
| `src/auth/roles.decorator.ts` | `@Roles(...grupos)`: anota qué grupos exige una ruta |
| `src/auth/rol.guard.ts` | `RolGuard`: lee la anotación de `@Roles` y la compara con `req.usuario.grupos` |
| `src/panel/panel.service.ts` | La lógica: llama a `LIBROS_URL` y `PRESTAMOS_URL` en paralelo, cruza los datos, filtra por dueño y valida las escrituras |
| `src/panel/panel.controller.ts` | Las rutas HTTP del panel |

## Las rutas que sirve

| Método y ruta | Guards | Qué hace | Llama a |
|---|---|---|---|
| `GET /panel` | `JwtGuard` | Los préstamos del usuario del token, con su libro cruzado | `LIBROS_URL` y `PRESTAMOS_URL`, en paralelo |
| `GET /panel/serie` | `JwtGuard` | Lo mismo, en serie, solo para medir la diferencia con el paralelo | Los mismos, uno tras otro |
| `GET /panel/todos` | `JwtGuard`, `RolGuard` + `@Roles('bibliotecarios')` | Los préstamos de todos | Los mismos |
| `POST /panel/prestamos` | `JwtGuard` | Crea un préstamo del `libroId` del cuerpo a nombre del `sub` del token | `PRESTAMOS_URL` (`POST`) |
| `DELETE /panel/prestamos/:id` | `JwtGuard` | Marca un préstamo como devuelto, solo si es del `sub` del token | `PRESTAMOS_URL` (`DELETE`) |

Sin token, las cinco responden **401**. Con un token válido sin el grupo `bibliotecarios`,
`/panel/todos` responde **403**. El gateway las publica como `/v1/panel...`.

## Cómo usarlo

Necesitas **Node 24.15.0 o superior** y **npm 11** (`node -v`, `npm -v`). Forkea `biblioteca-bff-l9` y clónalo
**sin el sufijo**, junto a los otros siete. Los mismos comandos sirven en Windows (PowerShell), macOS y Linux:

```bash
cd $HOME/DSY1107/l9
git clone https://github.com/TU_USUARIO/biblioteca-bff-l9.git biblioteca-bff
cd biblioteca-bff
npm install
```

Copia `.env.example` a `.env` (Windows: `Copy-Item .env.example .env`; macOS y Linux: `cp .env.example .env`) y
completa `COGNITO_ISSUER` y `COGNITO_CLIENT_ID` con los de tu ficha: son los mismos que usa tu gateway.
`LIBROS_URL` y `PRESTAMOS_URL` ya vienen con los puertos del curso. Arráncalo en su propia terminal:

```bash
npm run start:dev
```

Si falta una variable, no arranca y te dice cuál: `Configuration key "COGNITO_ISSUER" does not exist`.
Con todo el sistema, desde `biblioteca-plataforma`: `docker compose up -d --build --wait`.

## Cómo se comprueba

Con `biblioteca-libros` y `biblioteca-prestamos` corriendo y un access token vigente de `lector@biblioteca.test`:

**Windows (PowerShell):**

```powershell
$t = "<TOKEN_LECTOR>"
curl.exe -i http://localhost:3000/panel
curl.exe -i -H "Authorization: Bearer $t" http://localhost:3000/panel
```

**macOS y Linux:**

```bash
t="<TOKEN_LECTOR>"
curl -i http://localhost:3000/panel
curl -i -H "Authorization: Bearer $t" http://localhost:3000/panel
```

**Qué tienes que ver:** **401** sin cabecera y **200** con el token, con tus préstamos y su libro cruzado.

`test/app.e2e-spec.ts` es el que dejó `nest new` y prueba un `GET /` que ya no existe. No lo corras.
