# SsmsObjectScripter

Extensión para SQL Server Management Studio (SSMS) que permite generar el script de un procedimiento almacenado, una vista o una función directamente desde el texto seleccionado en el editor SQL.

En lugar de buscar manualmente el objeto en el Explorador de objetos, basta con seleccionar una referencia como:

```sql
EXEC dbo.spReporte2026
```

y utilizar una de las acciones agregadas al menú contextual del editor.

## Características

- Integración con el menú contextual del editor SQL de SSMS.
- Compatible con SSMS 21 y SSMS 22 de 64 bits.
- Reconoce nombres simples, nombres con esquema y nombres de tres partes.
- Genera scripts para procedimientos almacenados, vistas y funciones.
- Utiliza SQL Server Management Objects (SMO) para obtener la definición del objeto.
- Respeta el servidor y la base de datos de la consulta activa.
- Abre el resultado en una nueva pestaña conectada al mismo servidor.
- No ejecuta automáticamente el script generado.

## Acciones disponibles

### Generar DROP y CREATE

Genera dos lotes independientes:

1. Eliminación del objeto, con comprobación de existencia producida por SMO.
2. Creación completa del objeto.

Los lotes se separan con `GO`, por ejemplo:

```sql
DROP PROCEDURE IF EXISTS [dbo].[spReporte2026]
GO

CREATE PROCEDURE [dbo].[spReporte2026]
AS
BEGIN
    -- Definición del procedimiento
END
GO
```

> Esta alternativa elimina y vuelve a crear el objeto. Según el objeto y la configuración del servidor, podría ser necesario revisar permisos u otras propiedades asociadas antes de ejecutar el resultado.

### Generar CREATE OR ALTER (o ALTER compatible)

Cuando el servidor admite `CREATE OR ALTER`, la extensión solicita a SMO esa modalidad:

```sql
CREATE OR ALTER PROCEDURE [dbo].[spReporte2026]
AS
BEGIN
    -- Definición del procedimiento
END
GO
```

`CREATE OR ALTER` requiere SQL Server 2016 SP1 o posterior. Si el servidor no lo admite, la extensión conserva la definición generada por SMO y cambia únicamente el encabezado a una instrucción `ALTER` compatible:

```sql
ALTER PROCEDURE [dbo].[spReporte2026]
AS
BEGIN
    -- Definición del procedimiento
END
GO
```

Para objetos existentes, `ALTER` evita eliminarlos y normalmente conserva sus permisos.

## Tipos de selección admitidos

La selección puede contener solamente el nombre o una invocación `EXEC`/`EXECUTE`:

```sql
spReporte2026
dbo.spReporte2026
[dbo].[spReporte2026]
EXEC dbo.spReporte2026
EXEC dbo.spReporte2026 @Empleado = 10
EXEC @resultado = [rrhh].[sp Rep Vacaciones]
[Nomina].[dbo].[vwVacaciones]
```

Se admiten nombres de hasta tres partes:

```text
[base_de_datos].[esquema].[objeto]
```

Los nombres de cuatro partes que incluyen un servidor remoto no están admitidos.

## Objetos compatibles

- Procedimientos almacenados.
- Vistas.
- Funciones escalares.
- Funciones con valores de tabla en línea.
- Funciones con valores de tabla de múltiples instrucciones.

Los objetos cifrados no pueden ser leídos ni generados. El usuario conectado debe tener permiso para consultar la definición del objeto, normalmente mediante `VIEW DEFINITION`.

## Instalación

1. Descarga `SsmsObjectScripter.vsix` desde la sección **Releases** del repositorio.
2. Cierra todas las instancias de SQL Server Management Studio.
3. Ejecuta el archivo `.vsix`.
4. Selecciona SQL Server Management Studio 21 o 22 en el instalador.
5. Abre SSMS y verifica la instalación en **Extensiones > Administrar extensiones**.

Si Windows no asocia el archivo con el instalador correcto, utiliza directamente el instalador de SSMS. Para SSMS 22, la ubicación habitual es:

```text
C:\Program Files\Microsoft SQL Server Management Studio 22\Release\Common7\IDE\VSIXInstaller.exe
```

Después de actualizar la extensión, comprueba que la versión instalada sea `1.1.5` o posterior.

## Uso

1. Abre una ventana de consulta conectada al servidor y a la base de datos correctos.
2. Selecciona el nombre o la ejecución del objeto.
3. Haz clic derecho sobre la selección.
4. Elige una de estas acciones:

   - **Generar DROP y CREATE del objeto seleccionado**.
   - **Generar CREATE OR ALTER (o ALTER compatible)**.

5. Revisa el script abierto en la nueva pestaña antes de ejecutarlo.

La extensión nunca ejecuta por sí misma `DROP`, `CREATE` ni `ALTER`.

## Funcionamiento

La extensión realiza el siguiente proceso:

1. Interpreta la selección y extrae base de datos, esquema y nombre.
2. Reutiliza la información de conexión de la consulta activa.
3. Realiza una consulta puntual a `sys.objects` para determinar el tipo y esquema reales.
4. Obtiene el objeto correspondiente mediante SMO.
5. Genera los lotes SQL con `ScriptingOptions`.
6. Compone los lotes con separadores `GO` y abre el resultado en una nueva consulta.

La consulta a `sys.objects` se utiliza únicamente para localizar eficientemente el objeto. La definición SQL se obtiene mediante SMO.

## Requisitos para compilar

- Windows de 64 bits.
- .NET Framework 4.8 Developer Pack.
- Visual Studio 2022 con la carga **Desarrollo de extensiones de Visual Studio**.
- SSMS 21 o SSMS 22 para ejecutar y probar el VSIX.
- SDK de .NET compatible con los proyectos de la solución.

## Compilación

Desde la raíz del repositorio:

```powershell
dotnet build .\SsmsObjectScripter.slnx -c Release
```

El paquete resultante queda en:

```text
src\SsmsObjectScripter\bin\Release\SsmsObjectScripter.vsix
```

## Pruebas

Las pruebas del analizador y del compositor de lotes pueden ejecutarse con:

```powershell
dotnet run --project .\tests\SsmsObjectScripter.Core.Tests\SsmsObjectScripter.Core.Tests.csproj -c Release
```

Actualmente se verifican, entre otros casos:

- `EXEC` con y sin esquema.
- Asignación del valor de retorno de un procedimiento.
- Nombres de tres partes.
- Separación correcta de lotes mediante `GO`.
- Conversión compatible de `CREATE` a `ALTER` sin modificar comentarios ni el cuerpo del objeto.

## Estructura del proyecto

```text
SsmsObjectScripter/
├── src/
│   ├── SsmsObjectScripter.Core/       Parser y composición de lotes
│   └── SsmsObjectScripter/            Paquete VSIX, integración con SSMS y SMO
├── tests/
│   └── SsmsObjectScripter.Core.Tests/ Pruebas ejecutables
├── SsmsObjectScripter.slnx
└── README.md
```

## Solución de problemas

### Las opciones no aparecen al hacer clic derecho

- Confirma que la extensión esté habilitada en **Extensiones > Administrar extensiones**.
- Comprueba que la versión instalada sea la más reciente.
- Cierra todas las instancias de SSMS antes de instalar o actualizar el VSIX.
- Abre una pestaña `.sql`, selecciona texto y haz clic derecho dentro del editor.

### El objeto no se encuentra

- Verifica que la consulta esté conectada a la base correcta.
- Incluye el esquema en la selección, por ejemplo `dbo.spReporte2026`.
- Para otra base de datos, utiliza `[Base].[dbo].[Objeto]`.
- Comprueba que la cuenta tenga permiso `VIEW DEFINITION`.

### Se genera ALTER en lugar de CREATE OR ALTER

El servidor no admite `CREATE OR ALTER` o SMO está generando para una versión de destino anterior. La extensión utiliza `ALTER` como alternativa compatible para el objeto existente.

### Consideraciones sobre extensiones de SSMS

SSMS utiliza la infraestructura de extensibilidad de Visual Studio, pero las extensiones de terceros pueden requerir ajustes después de una actualización importante de SSMS. Si una nueva versión deja de cargar el paquete, será necesario revisar el rango de versiones del manifiesto y las APIs internas de integración.

## Seguridad

- La extensión no transmite credenciales ni definiciones a servicios externos.
- Reutiliza localmente la conexión proporcionada por SSMS.
- No ejecuta el script generado.
- El usuario debe revisar siempre el resultado antes de ejecutarlo, especialmente en la modalidad `DROP + CREATE`.

## Tecnologías utilizadas

- C# y .NET Framework 4.8.
- Visual Studio SDK / VSIX.
- SQL Server Management Objects (SMO).
- APIs de integración del editor de SQL Server Management Studio.

