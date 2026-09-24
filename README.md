# Migración de Google Forms → AppSheet (cada usuario ve y edita solo sus registros)

Guía paso a paso para convertir un Google Form que guarda respuestas en una hoja de cálculo en una app de AppSheet donde:

- Cada usuario entra con su cuenta de Google.
- Crea nuevos registros (como antes con el Form).
- Ve y **modifica solo sus propios registros**, incluidos los que ya envió por el Form.

> Supuesto: el Form **recopilaba correos**, así que la hoja ya tiene una columna tipo `Dirección de correo electrónico` que identifica al dueño de cada fila.

---

## Índice

1. [Respaldo](#1-respaldo)
2. [Desvincular el Form](#2-desvincular-el-form)
3. [Limpiar y preparar la hoja](#3-limpiar-y-preparar-la-hoja)
4. [Agregar una columna ID única](#4-agregar-una-columna-id-única)
5. [Crear la app en AppSheet](#5-crear-la-app-en-appsheet)
6. [Configurar las columnas](#6-configurar-las-columnas)
7. [Permitir agregar y editar](#7-permitir-agregar-y-editar)
8. [Seguridad: cada quien lo suyo](#8-seguridad-cada-quien-lo-suyo)
9. [Vistas (UX)](#9-vistas-ux)
10. [Probar como otro usuario](#10-probar-como-otro-usuario)
11. [Compartir y desplegar](#11-compartir-y-desplegar)
12. [Problemas comunes](#12-problemas-comunes)
13. [Checklist final](#13-checklist-final)

---

## 1. Respaldo

Antes de tocar nada:

1. Abre la hoja de respuestas.
2. Ve a **Archivo → Hacer una copia**.
3. Nómbrala `Respuestas - RESPALDO AAAA-MM-DD`.

Si algo sale mal, vuelves a esta copia.

---

## 2. Desvincular el Form

AppSheet y el Form no deben escribir en la misma hoja a la vez. Si no los separas, se desordenan filas y columnas.

1. Abre el Google Form → pestaña **Respuestas**.
2. Desactiva **Aceptando respuestas**. Opcional: pon un mensaje como "Ahora usa la app: <link>".
3. Haz clic en **⋮ → Desvincular formulario**.

Las respuestas existentes se quedan en la hoja.

> Si necesitas un periodo de transición, deja el Form activo unos días. Desvincúlalo **antes** de dar acceso a la app.

---

## 3. Limpiar y preparar la hoja

AppSheet lee la **fila 1 como encabezados** y cada fila siguiente como un registro.

### 3.1 Encabezados

- Renombra los encabezados largos (las preguntas del Form) a nombres cortos y claros.

  | Antes (pregunta del Form) | Después |
  |---|---|
  | Marca temporal | `Fecha` |
  | Dirección de correo electrónico | `Correo` |
  | ¿Cuál es el nombre de tu proyecto? | `Proyecto` |
  | Describe brevemente el avance | `Avance` |

- Sin encabezados repetidos, sin celdas combinadas, sin filas vacías arriba.
- Evita caracteres raros (`#`, `/`, `[ ]`) en los nombres.

### 3.2 Pestaña de datos

- Renombra la pestaña, por ejemplo de `Respuestas de formulario 1` a `Registros`.
- Borra fórmulas, filtros o formatos condicionales que ocupen filas debajo de los datos. AppSheet podría leerlas como registros vacíos.

### 3.3 Correos

- Revisa que la columna `Correo` esté en **minúsculas y sin espacios**. `USEREMAIL()` devuelve el correo en minúsculas, así que `Juan@...` no coincidiría.
- Para limpiarla rápido: en una columna auxiliar usa `=LOWER(TRIM(B2))`, copia, **pega solo valores** sobre `Correo` y borra la auxiliar.

### 3.4 ¿Un registro o varios por usuario?

Decide cuál es tu caso:

- **Varios registros por usuario** (cada envío es un registro): no hagas nada.
- **Un solo registro por usuario** (cada quien tiene "su ficha"): elimina duplicados y deja solo la respuesta más reciente de cada correo.
  1. Ordena por `Fecha` de más reciente a más antigua.
  2. Ve a **Datos → Limpieza de datos → Quitar duplicados**, marcando solo la columna `Correo`.
  3. Se conserva la primera aparición, que es la más reciente.

---

## 4. Agregar una columna ID única

AppSheet necesita una **clave** (Key) única por fila para poder editar registros. El correo no sirve como clave si un usuario tiene varias filas, y la fecha puede repetirse.

1. Inserta una columna nueva en la **columna A** llamada `ID`.
2. Llena los IDs de las filas existentes con una de estas opciones.

### Opción A — Rápida (fórmula)

1. En `A2` escribe `="R-"&ROW()` y arrástrala hasta la última fila.
2. Selecciona la columna → copia → **Pegar especial → Solo valores**. Importante: el ID no puede ser una fórmula viva.

### Opción B — IDs aleatorios (Apps Script)

**Extensiones → Apps Script**, pega esto y ejecútalo una vez:

```javascript
function llenarIDs() {
  const hoja = SpreadsheetApp.getActive().getSheetByName('Registros');
  const ultima = hoja.getLastRow();
  if (ultima < 2) return;
  const rango = hoja.getRange(2, 1, ultima - 1, 1); // columna A, desde fila 2
  const valores = rango.getValues().map(([v]) =>
    [v ? v : Utilities.getUuid().slice(0, 8)]
  );
  rango.setValues(valores);
}
```

Los registros nuevos que se creen en la app generarán su ID solos (paso 6).

---

## 5. Crear la app en AppSheet

1. Con la hoja abierta: **Extensiones → AppSheet → Crear una app**.
2. Inicia sesión con tu cuenta. La primera vez te pedirá permisos para leer la hoja.
3. AppSheet detecta la tabla `Registros` y genera una app básica.

> Alternativa: entra a appsheet.com → **Create → App → Start with existing data** y elige la hoja.

---

## 6. Configurar las columnas

Ve a **Data → Registros → View columns** (o el ícono de columnas).

### 6.1 Columnas de sistema

| Columna | Type | Key | Show | Editable | Initial value |
|---|---|---|---|---|---|
| `ID` | Text | ✅ | ❌ | ❌ | `UNIQUEID()` |
| `Fecha` | DateTime | | ✅ | ❌ | `NOW()` |
| `Correo` | Email | | opcional | ❌ | `USEREMAIL()` |

- En `ID`, activa **Key**. Si AppSheet eligió otra columna como Key, desactívala allí.
- Para que no sean editables, apaga el switch **Editable** (o pon `Editable_If = FALSE`).
- Opcional: marca `Fecha` y `Correo` como **Label** solo si te sirven como título del registro. Normalmente es mejor usar `Proyecto` o un campo descriptivo.

### 6.2 Columnas de contenido (las preguntas del Form)

Ajusta el **Type** según el tipo de pregunta que tenías:

| Tipo de pregunta en Forms | Type en AppSheet | Notas |
|---|---|---|
| Respuesta corta | Text | |
| Párrafo | LongText | |
| Opción múltiple / Desplegable | Enum | Captura las opciones en *Values* |
| Casillas de verificación | EnumList | **Item separator: `, `** (coma + espacio), porque Forms guarda así los valores |
| Fecha | Date | |
| Hora | Time | |
| Escala lineal | Number o Enum | Enum si quieres botones 1–5 |
| Cuadrícula | Una columna Enum por fila | Forms crea una columna por fila de la cuadrícula |
| Subida de archivo | File / Image | Los viejos son links de Drive; revisa el punto 12 |

- Marca **Require** en los campos que eran obligatorios en el Form.
- En **Display name** puedes poner el texto largo de la pregunta, para que el usuario vea lo mismo que antes aunque el encabezado sea corto.
- Para campos que el usuario **no** debe cambiar después de crearlos, usa `Editable_If`: `ISBLANK([_THIS])`. Así solo se pueden llenar la primera vez.

Guarda con **Save** (arriba a la derecha).

---

## 7. Permitir agregar y editar

**Data → Registros → Table settings** (o *Are updates allowed?*):

- ✅ **Adds**: crear registros nuevos.
- ✅ **Updates**: modificar registros.
- ❌ **Deletes**: actívalo solo si quieres que los usuarios borren.

---

## 8. Seguridad: cada quien lo suyo

### 8.1 Exigir inicio de sesión

**Security → Require Sign-In**: debe estar **activado**. Sin esto, `USEREMAIL()` queda vacío y el filtro no funciona.

### 8.2 Security filter

**Security → Security filters → Registros**:

```
[Correo] = USEREMAIL()
```

Con un administrador que ve todo:

```
OR(
  IN(USEREMAIL(), LIST("admin@tudominio.com", "otro.admin@tudominio.com")),
  [Correo] = USEREMAIL()
)
```

> ¿Por qué *security filter* y no un *slice*? Un slice solo esconde filas en la interfaz, pero todos los datos se descargan al dispositivo. El security filter hace que el servidor **nunca envíe** las filas de otros usuarios.

### 8.3 Evitar que alguien se "robe" registros

Como `Correo` no es editable (paso 6.1), nadie puede cambiar el dueño de un registro. Para más seguridad, en `Correo` pon **Valid If**: `[_THIS] = USEREMAIL()`. Un admin tendría que ir en un `OR` como arriba.

---

## 9. Vistas (UX)

En **App → Views** (o *UX → Views*):

1. **Vista principal "Mis registros"**
   - View type: **Deck** o **Table**.
   - For this data: `Registros`.
   - Sort by: `Fecha` descendente.
   - El security filter ya limita los registros, no necesitas filtrar aquí.
2. **Formulario** (`Registros_Form`, se crea solo)
   - Ordena los campos en **Column order** igual que en el Form original.
   - Oculta `ID`, `Fecha` y `Correo`.
3. **Botón de agregar**: la vista Deck/Table ya muestra un botón **+** si Adds está activo.
4. **Editar**: al abrir un registro aparece el ícono de lápiz si Updates está activo.

Si es "un registro por usuario", puedes ocultar el botón **+** cuando el usuario ya tenga su ficha. En **Actions → Add** (acción del sistema), usa *Only if this condition is true*:

```
COUNT(Registros[ID]) = 0
```

Esto funciona porque el security filter hace que `Registros` solo contenga las filas del usuario.

---

## 10. Probar como otro usuario

En el editor, en el panel de vista previa, abajo está **Preview app as**:

1. Escribe el correo de un usuario que tenga respuestas en la hoja.
2. Verifica que:
   - Solo ve **sus** registros.
   - Puede editarlos y el cambio aparece en la hoja.
   - Al crear uno nuevo, la hoja recibe `ID`, `Fecha` y `Correo` automáticamente.
3. Prueba con un correo **sin registros**: debe ver la lista vacía.
4. Prueba con el correo del admin: debe ver todo.

---

## 11. Compartir y desplegar

1. **Share** (arriba a la derecha) → agrega los correos de los usuarios o, en Workspace, todo el dominio.
2. Los usuarios reciben un link. Pueden usar la app en el navegador o instalar **AppSheet** en Android/iOS.
3. Cuando esté lista: **Manage → Deploy → Move app to Deployed state**.
   - En estado *Prototype* solo puedes compartir con pocos usuarios de prueba.
   - Para desplegar a todos, cada usuario necesita licencia de AppSheet. Muchas ediciones de Google Workspace incluyen **AppSheet Core**. Confírmalo con tu administrador.
4. Actualiza el mensaje del Form cerrado con el link a la app.

---

## 12. Problemas comunes

| Síntoma | Causa probable | Solución |
|---|---|---|
| Un usuario no ve sus registros viejos | Correo con mayúsculas, espacios o una cuenta distinta | Normaliza con `LOWER(TRIM())` (paso 3.3) y confirma con qué cuenta entra |
| Nadie ve nada | Require Sign-In apagado, o la columna de correo tiene otro nombre | Revisa el paso 8.1 y que el filtro use el nombre exacto de la columna |
| "Key column values are not unique" | IDs repetidos o vacíos | Rellena los IDs (paso 4) y verifica que no se repitan |
| Casillas múltiples salen como un solo texto | Separador de EnumList incorrecto | Item separator = `, ` |
| Los archivos subidos por el Form no se abren | Son links de Drive con permisos del dueño original | Deja la columna como **Url**, o comparte la carpeta de Drive con los usuarios |
| Aparecen filas vacías en la app | Fórmulas o formatos debajo de los datos | Borra el contenido debajo de la última fila real |
| Los cambios no llegan a la hoja | La app no sincronizó | Toca el ícono de sincronizar; revisa **Manage → Monitor → Audit history** |
| Cambié la hoja y la app falla | Columnas agregadas o renombradas | **Data → Registros → Regenerate structure** (revisa después los tipos) |

---

## 13. Checklist final

- [ ] Respaldo creado
- [ ] Form cerrado y desvinculado
- [ ] Encabezados limpios, pestaña renombrada
- [ ] Correos en minúsculas y sin espacios
- [ ] Columna `ID` llena y marcada como Key
- [ ] `ID` = `UNIQUEID()`, `Fecha` = `NOW()`, `Correo` = `USEREMAIL()` (no editables)
- [ ] Tipos de columna ajustados (Enum, EnumList, etc.)
- [ ] Adds + Updates activos
- [ ] Require Sign-In activo
- [ ] Security filter `[Correo] = USEREMAIL()`
- [ ] Probado con "Preview app as" (usuario, usuario sin registros, admin)
- [ ] Compartido con los usuarios y desplegado
