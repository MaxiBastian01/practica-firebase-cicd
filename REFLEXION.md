# Reflexión final — CI/CD con Firebase Hosting

## 1. ¿Por qué el preview tiene vencimiento? ¿Qué riesgos tendría un preview que no vence nunca?

El preview tiene vencimiento porque es un entorno temporal que se utiliza para probar los cambios antes de incorporarlos a producción.

Si no venciera, se acumularían versiones antiguas de la aplicación, lo que podría generar confusión, mantener disponibles versiones con vulnerabilidades y consumir recursos innecesariamente.

## 2. ¿Qué diferencia hay entre desplegar y liberar? ¿Dónde aparece cada concepto en este flujo?

**Desplegar** significa subir una versión de la aplicación a un entorno donde pueda ejecutarse o utilizarse.

**Liberar** significa poner esa versión a disposición de los usuarios finales.

En nuestro caso, cuando abrimos un Pull Request, GitHub Actions despliega la aplicación en una URL de preview, pero todavía no la libera al público como versión oficial.

Cuando hacemos el merge a `main`, se ejecuta otro workflow que despliega la nueva versión al canal `live` de Firebase Hosting, dejándola disponible para los usuarios.

## 3. ¿Por qué la clave de la service account va en un secreto y no en el repositorio? ¿Qué podría hacer alguien que la obtenga? ¿Cómo limitarías el daño?

La clave de la service account se guarda como secreto de GitHub porque permite que GitHub Actions se autentique con Firebase para realizar los despliegues.

Si la guardáramos directamente en el repositorio, cualquier persona con acceso al código podría obtenerla y utilizar los permisos de esa cuenta, por ejemplo, para publicar contenido no autorizado.

Para limitar el daño, se deben otorgar únicamente los permisos necesarios, evitar compartir la clave, rotarla si se filtra y restringir quién puede ejecutar workflows con acceso a los secretos.

## 4. La CSP rompió la app: ¿por qué? ¿Qué opciones evaluaste y qué costo de seguridad tiene cada una?

La Content-Security-Policy (CSP) controla desde dónde puede cargar recursos una página web y qué scripts puede ejecutar.

Una política demasiado restrictiva puede bloquear scripts necesarios para que funcione Next.js, además de imágenes o estilos.

Para solucionarlo, utilizamos `script-src 'self' 'unsafe-inline'` y `style-src 'self' 'unsafe-inline'`, permitiendo los scripts y estilos que necesita la aplicación.

El costo de seguridad es que `unsafe-inline` reduce la protección frente a ciertos ataques de inyección de código, como XSS.

Una alternativa más segura sería utilizar hashes SHA-256 para autorizar únicamente los scripts inline conocidos. Sin embargo, requeriría calcular y actualizar esos hashes durante el proceso de compilación.

## 5. Para cada header de B2, ¿qué ataque mitiga?

**Strict-Transport-Security (HSTS):** obliga al navegador a utilizar HTTPS en futuras conexiones, reduciendo el riesgo de ataques que intentan degradar la conexión a HTTP.

**X-Content-Type-Options:** con `nosniff`, evita que el navegador interprete archivos como un tipo de contenido diferente al declarado, reduciendo riesgos de ejecución de contenido malicioso.

**X-Frame-Options:** con `DENY`, impide que otros sitios incorporen nuestra página dentro de un iframe, protegiendo contra ataques de clickjacking.

**Referrer-Policy:** con `strict-origin-when-cross-origin`, limita la información de la URL que se comparte al navegar hacia otros sitios, reduciendo la exposición de datos.

**Permissions-Policy:** restringe el acceso a funciones del navegador. En nuestro caso, deshabilitamos cámara, micrófono y geolocalización para reducir el uso no autorizado de esas capacidades.

## 6. ¿Por qué los assets de /_next/static/ pueden cachearse un año y el HTML no?

Los archivos de `/_next/static/` tienen nombres que incluyen hashes de su contenido. Cuando cambia un archivo, se genera otro nombre, por lo que podemos guardarlos en caché durante un año sin riesgo de utilizar una versión antigua con el mismo nombre.

En cambio, el HTML puede cambiar entre despliegues manteniendo la misma URL.

Por eso configuramos los assets con `max-age=31536000, immutable` y el HTML con `max-age=0, must-revalidate`.

De esta manera, mejoramos la velocidad de carga sin impedir que los usuarios vean los cambios o un rollback rápidamente.

## 7. Si la app necesitara SSR, ¿qué cambiaría en la arquitectura, en la CSP y en el costo?

Actualmente utilizamos Next.js con exportación estática, por lo que Firebase Hosting puede servir los archivos sin necesidad de ejecutar un servidor.

Si necesitáramos SSR (Server-Side Rendering), necesitaríamos un entorno capaz de ejecutar código del lado del servidor, como Firebase App Hosting con Cloud Run.

La CSP también podría cambiar, ya que tendríamos que considerar los recursos generados dinámicamente y podríamos utilizar nonces por solicitud para autorizar scripts específicos.

En cuanto al costo, dejaríamos de depender únicamente del Hosting estático en el plan Spark y necesitaríamos el plan Blaze con facturación habilitada, debido al uso de infraestructura de ejecución del servidor.