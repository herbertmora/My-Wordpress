Acerca de este instructivo
Este instructivo muestra una aplicación de Python que se ejecuta en Google App Engine con Python Flask y que interactúa con la API de Google Books. La aplicación muestra cómo Travis puede implementar y ejecutar pruebas de extremo a extremo en un entorno de etapa de pruebas, como parte del proceso de prueba activado por un comando git push. En el siguiente diagrama, se muestra el proceso desde un nivel alto:

Travis CI ejecuta pruebas locales entre implementaciones GitHub y Cloud Platform

Luego de describir el ejemplo de App Engine, este artículo analiza las consideraciones de la integración con el entorno flexible de Google App Engine y Compute Engine. El repositorio de GitHub que contiene la muestra también tiene una rama de VM administrada que demuestra la implementación de la misma aplicación mediante la utilización del entorno flexible.

Travis CI tiene compatibilidad integrada para la implementación en entornos estándares o flexibles de App Engine mediante la utilización de proveedores de implementación. Cuando usas estos proveedores, algunos de los pasos de ajuste básicos son los mismos, pero el archivo de configuración es más simple. Este instructivo muestra el funcionamiento interno de estos proveedores. Se descarga el SDK de Google Cloud, que también se puede usar con otros fines a parte de la implementación, como la ejecución de emuladores locales para las pruebas de unidades. Para obtener más información, consulta Cómo usar los proveedores de implementación de Travis CI más adelante en este artículo.

Requisitos previos
Ejecutar Linux o macOS.
Instalar el SDK de Google Cloud.
Tener una cuenta de GitHub y también instalar git en tu computadora local. Este instructivo supone que tienes conocimientos básicos de GitHub y git.
Registrarse en Travis. Travis se integra a GitHub mediante tu cuenta de GitHub.
Instalar la herramienta de línea de comandos de Travis. Necesitarás esta herramienta para encriptar credenciales. Quizás primero debas instalar Ruby.
Importante: Este instructivo utiliza Google Compute Engine, que es un componente facturable de Cloud Platform.
Usa la calculadora de precios para generar una estimación de los costos según el uso previsto. Es posible que los usuarios nuevos de Cloud Platform sean aptos para una prueba gratuita.

Configuración de los códigos de muestra
Bifurca el repositorio de muestra. En GitHub, haz clic en Bifurcar. Es importante realizar la bifurcación ya que reemplazarás las credenciales del proyecto de muestra con tus propias credenciales del proyecto.

Clona la bifurcación que creaste. En la ventana de terminal, ingresa el siguiente comando. Reemplaza <your-GitHub-username> por tu nombre de usuario:

$ git clone https://github.com/<your-GitHub-username>/continuous-deployment-demo.git
Cambia el directorio del repositorio:

$ cd continuous-deployment-demo
Cómo funciona la muestra
La aplicación de muestra tiene un extremo Python Flask simple que busca el autor de un libro por su título:

main.pyVER EN GITHUB
@app.route('/get_author/<title>')
def get_author(title):
    host = 'https://www.googleapis.com/books/v1/volumes?q={}&key={}&country=US'.format(title, key)
    request = urllib2.Request(host)
    try:
        response = urllib2.urlopen(request)
    except urllib2.HTTPError, error:
        contents = error.read()
        print ('Received error from Books API {}'.format(contents))
        return str(contents)
    html = response.read()
    author = json.loads(html)['items'][0]['volumeInfo']['authors'][0]
    return author
Una prueba de integración accede a este extremo y, luego, verifica que para el título del libro “Ulises”, la aplicación muestre el nombre de autor “James Joyce”:

e2e_test.pyVER EN GITHUB
response = urllib2.urlopen("{}/get_author/ulysses".format(HOST))
html = response.read()
assert(html == "James Joyce")
Cómo crear un proyecto
Crea un proyecto de Google Cloud Platform Console con el fin de usarlo como entorno de etapa de pruebas para la implementación.

En GCP Console, en la página de selección de proyecto, selecciona o crea un proyecto de GCP.

Nota : Si no planeas conservar los recursos que creas en este procedimiento, crea un proyecto en lugar de seleccionar un proyecto existente. Cuando termines, puedes borrar el proyecto y quitar todos los recursos asociados con él.
IR A LA PÁGINA DE SELECCIÓN DE PROYECTO

Habilita las Books API necesarias.
HABILITA LAS API

Cómo especificar el host
Especifica la aplicación de App Engine en e2e_test.py. Edita el archivo y modifica la siguiente línea para usar la URL de tu aplicación en lugar de la URL de muestra:

HOST='http://continuous-deployment-python.appspot.com'
Debes especificar el ID de tu proyecto en el archivo de configuración, que a veces se denomina archivo Travis. Edita el archivo denominado .travis.yml. En la siguiente línea, reemplaza continuous-deployment-python por el nombre de tu proyecto de GCP Console:

- gcloud config set project `continuous-deployment-python`
Confirma los archivos:

$ git add e2e_test.py .travis.yml
$ git commit -m "Changed host and project ID"
Cómo crear las credenciales
Para implementar la aplicación en este proyecto, debes crear dos tipos de credenciales:

Credenciales de cuenta de servicio que habilitan al SDK de Cloud a autenticar correctamente el proyecto de Cloud Platform.
Una clave de API pública que la aplicación de App Engine utiliza para comunicarse con la API de Books.
Cómo crear las credenciales de cuenta de servicio
En tu proyecto de GCP Console, abre la página Credenciales.
Haz clic en Crear credenciales > Clave de cuenta de servicio.
En Cuenta de servicio, selecciona Nueva cuenta de servicio.
Ingresa un Nombre de cuenta de servicio como continuous-integration-test.
En Función, selecciona Proyecto > Editor.
En Tipo de clave, selecciona JSON.
Haz clic en Crear. La GCP Console descarga un archivo JSON nuevo en la computadora. El nombre de este archivo comienza con el ID de tu proyecto.
Copia este archivo en la raíz del repositorio de GitHub y cambia el nombre del archivo por client-secret.json.
Cómo crear la clave de API pública
Desde la misma página de Credenciales, haz clic en Crear credenciales > Clave de API.

Se creará una clave de manera automática.

Copia la clave de API en el portapapeles.

Edita api_key.py.sample, y luego reemplaza YOUR-API-KEY por la clave de API que copiaste de la consola.

Guarda el archivo. Cuando lo guardes, cambia el nombre borrando .sample de la extensión del nombre del archivo:

api_key.py
Sal del editor.

Encriptación y desencriptación
Debido a que estas credenciales se agregan a un repositorio de GitHub público, deben ser encriptadas y desencriptadas por parte de Travis. Puedes leer más sobre cómo encriptar archivos de Travis CI.

Crea un archivo de almacenamiento tar que contenga ambas credenciales. Esto es importante porque Travis CI solo puede desencriptar un archivo. En la ventana de terminal, ejecuta el siguiente comando:

$ tar -czf credentials.tar.gz client-secret.json api_key.py
Importante: No agregues este archivo a tu repositorio de Git. Este archivo contiene tus credenciales privadas. El archivo .gitignore del proyecto de muestra está configurado para ignorar este archivo, pero si cambias su nombre, deberás agregarlo a .gitignore de nuevo.
En el archivo .travis.yml, borra la información de encriptación. Esta información se agregará de manera automática cuando encriptes la clave en el próximo paso. Edita el archivo y, luego de before_install, borra la siguiente línea:

- openssl aes-256-cbc -K $encrypted_4cb330591e04_key -iv $encrypted_4cb330591e04_iv -in credentials.tar.gz.enc -out credentials.tar.gz
Accede a Travis. Te pedirán tus credenciales de registro de GitHub:

$ sudo travis login
Encripta el archivo de manera local. Si te piden que reemplaces el archivo existente, responde yes:

$ sudo travis encrypt-file credentials.tar.gz --add
Es la opción --add la que provoca que el cliente de Travis agregue de manera automática el paso de desencriptación al paso before_install en el archivo .travis.yml.

Agrega el archivo encriptado al repositorio:

$ git add credentials.tar.gz.enc .travis.yml
$ git commit -m "Adds encrypted credentials for Travis"
Cómo comprender el archivo .travis.yml
El archivo .travis.yml de la muestra contiene comentarios detallados por línea que explican qué contiene el archivo. Estas son algunas consideraciones que hay que tener en cuenta:

El comando cache mantiene la distribución del SDK de Cloud activa entre ejecuciones, lo que acelera la ejecución repetida de pruebas.

En la sección before_install, se desencripta el archivo de credenciales tarball con las claves que subiste con la herramienta Travis. Luego, se autentica gcloud auth activate-service-account con esas credenciales, lo que permite la implementación del proyecto.

.travis.ymlVER EN GITHUB
# Decrypt the credentials we added to the repo using the key we added with the Travis command line tool
- openssl aes-256-cbc -K $encrypted_45d1b36fa803_key -iv $encrypted_45d1b36fa803_iv -in credentials.tar.gz.enc -out credentials.tar.gz -d
# If the SDK is not already cached, download it and unpack it
- if [ ! -d ${HOME}/google-cloud-sdk ]; then
     curl https://sdk.cloud.google.com | bash;
  fi
- tar -xzf credentials.tar.gz
- mkdir -p lib
# Here we use the decrypted service account credentials to authenticate the command line tool
- gcloud auth activate-service-account --key-file client-secret.json
Finalmente, durante la etapa de script, se implementa el proyecto y se ejecuta la prueba de extremo a extremo.

.travis.ymlVER EN GITHUB
# Deploy the app
- gcloud -q app deploy app.yaml --promote
# Run and end to end test
- python e2e_test.py
Cómo habilitar la bifurcación en Travis CI
Ahora que está todo listo, habilita la bifurcación:

En el sitio web de Travis, haz clic en tu nombre de usuario en la esquina superior derecha para abrir la configuración de cuenta de Travis.
Si no ves el repositorio de muestra en la lista, haz clic en Sincronizar.
Cuando aparezca el repositorio de muestra en la lista, enciende el interruptor del repositorio, como se muestra en la siguiente imagen:
La página de configuración de cuenta de Travis CI muestra la lista de proyectos

Ten en cuenta que tu lista mostrará tu nombre de usuario en lugar de GoogleCloudPlatform.

Cada vez que ingresas una confirmación en la base de código, se implementa el código y se ejecuta la prueba de integración en un entorno real. Haz un ingreso ahora para comenzar una compilación:

    $ git push
Cómo trabaja Travis
Cuando se completan las pruebas, puedes examinar los resultados para ver si las pruebas aprobaron o fallaron.

Cuando las pruebas de Travis son aprobadas, la página es verde

Mira lo que sucede si agregas un pequeño error de tipeo al código borrando la “s” de la palabra volumes en la solicitud a la API de Books.

Edita main.py.
Cambia la siguiente línea. Borra la letra “s” de la palabra volumes:

host = 'https://www.googleapis.com/books/v1/volumes?q={}&key={}&country=US'.format(title, api_key)
Guarda el archivo y sal del editor.

Confirma los cambios:

$ git add main.py
$ git commit -m "Intentional error."
$ git push
Una vez que confirmas este código en el repositorio, falla la prueba de integración:

Cuando las pruebas de Travis no son aprobadas, la página es roja

Implementación continua en instancias de entorno flexible de App Engine
Si usas el entorno flexible, solo debes hacer un pequeño cambio para que la implementación siga funcionando en Travis. Debes agregar la siguiente línea al archivo .travis.yml en la sección before_install:

- ssh-keygen -q -N "" -f ~/.ssh/google_compute_engine
Este paso crea la Llave SSH necesaria, con una frase de contraseña vacía, que el SDK de Cloud utiliza con el objetivo de copiar los archivos del proyecto a una VM y compilar la imagen de Docker para la aplicación:

- gcloud -q app deploy
Implementación continua en Compute Engine
Debido a que los pasos de la implementación en una instancia de Compute Engine pueden variar mucho, los pasos de Travis para implementar variarán en consecuencia. El único paso que probablemente no cambie es la descarga del SDK de Cloud y su autenticación con las credenciales de cuenta de servicio.

Una de las posibles maneras de lograr la implementación es tener una VM configurada con anterioridad, luego usa gcloud compute scp para copiar el artefacto compilado en la instancia, y sigue con el comando gcloud compute ssh con el fin de iniciar la secuencia de comandos de implementación.

También necesitas el paso de ssh-keygen, el que se describió anteriormente para el entorno flexible de App Engine, con el fin de ejecutar el comando gcloud compute scp desde una compilación de Travis.

Cómo utilizar los proveedores de implementación de Travis CI
Travis CI agregó asistencia experimental para incorporar muchos de los pasos compilados en su herramienta de implementación con el objetivo de simplificar el archivo de Travis. Tendrás que descargar credenciales de cuenta de servicio, pero Travis descarga automáticamente el SDK de Cloud y ejecuta el comando de implementación. Puedes configurar la rama desde la cual implementar, la versión a implementar, el nombre de las credenciales de cuenta de servicio y otras opciones de configuración.

Puedes ver los repositorios de appengine_travis_deploy y managed_vms_travis_deploy para ver cómo es la configuración de Travis en esta opción. A continuación, se muestra la configuración de la implementación para el ejemplo de App Engine:

.travis.ymlVER EN GITHUB
deploy:
    provider: gae
    # Skip cleanup so api_key.py and vendored dependencies are still there
    skip_cleanup: true
    keyfile: client-secret.json
    project: continuous-deployment-python
    default: true
    on:
        all_branches: true
Presta atención a la configuración de skip_cleanup. Esta opción evita que se limpie el directorio de trabajo antes del paso deploy. La demostración requiere esta opción para evitar la eliminación de api_key.py y las bibliotecas que instaló pip.

Otras consideraciones
En el comando gcloud app deploy, puedes especificar una versión determinada con la opción --version. Puedes usar las variables del entorno de Travis con el fin de implementar una versión específica para esa compilación y, luego, hacer una prueba de la URL de la versión.

Si ejecutas pruebas de extremo a extremo en el mismo trabajo que usas para bloquear las fusiones de la rama de desarrollo con la rama principal, es posible que se ralentice el proceso de desarrollo. En cambio, puedes usar la prueba rápida solo para aprobar la fusión y ejecutar la prueba lenta de manera periódica en las confirmaciones más actuales. De manera alternativa, puedes iniciar un mensaje de Cloud Pub/Sub para activar las pruebas lentas en segundo plano.

Limpieza
Luego de finalizar el instructivo de Travis CI, puedes limpiar los recursos que creaste en Google Cloud Platform para que no los facturen en el futuro. La siguiente sección describe cómo borrar o desactivar estos recursos.

Cómo borrar el proyecto
La manera más fácil de eliminar la facturación es borrar el proyecto que creaste para el instructivo.

Para borrar el proyecto, haz lo siguiente:

Precaución: Borrar un proyecto tiene las siguientes consecuencias:
Todo en el proyecto se borra. Si utilizaste un proyecto existente para este instructivo, cuando lo borres, también borrarás cualquier otro trabajo que hayas realizado en el proyecto.
Los ID personalizados de proyectos se pierden. Cuando creaste este proyecto, es posible que hayas creado un ID del proyecto personalizado que desees utilizar en el futuro. Para conservar las URL que utilizan el ID del proyecto, como una URL de appspot.com, borra los recursos seleccionados dentro del proyecto en lugar de borrar todo el proyecto.
Si planeas explorar varios instructivos y guías de inicio rápido, la reutilización de proyectos puede ayudarte a evitar exceder los límites de las cuotas del proyecto.

En la GCP Console, dirígete a la página Proyectos.
IR A LA PÁGINA PROYECTOS

En la lista de proyectos, selecciona el proyecto que deseas borrar y haz clic en Borrar.
En el cuadro de diálogo, escribe el ID del proyecto y, luego, haz clic en Cerrar para borrar el proyecto.
Cómo detener la aplicación de App Engine
Borrar tu proyecto es la única forma de eliminar la versión predeterminada de tu app de App Engine. Sin embargo, puedes detener la versión predeterminada en GCP Console. Esta acción detiene todas las instancias asociadas a la versión. Puedes reiniciar estas instancias más tarde si es necesario.

En el entorno estándar de App Engine, puedes detener la versión predeterminada si tu app tiene escalamiento manual o básico.

Cómo borrar las instancias de entorno flexible de App Engine
Para borrar la versión de una app:

En GCP Console, dirígete a la página Versiones de App Engine.
IR A LA PÁGINA DE VERSIONES

Haz clic en la casilla de verificación junto a la versión de app no predeterminada que deseas borrar.
Nota:Borrar tu proyecto es la única forma de eliminar la versión predeterminada de tu app de App Engine. Sin embargo, puedes detener la versión predeterminada en GCP Console. Esta acción detiene todas las instancias asociadas a la versión. Puedes reiniciar estas instancias más tarde si es necesario.
En el entorno estándar de App Engine, puedes detener la versión predeterminada si tu app tiene escalamiento manual o básico.

Haz clic en el botón Borrar en la parte superior de la página para borrar la versión de la app.
