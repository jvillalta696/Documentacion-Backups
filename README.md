# Sincronizador de Backups para servidores de bases de datos Hana

## Descripción

Este proyecto se enfoca en poder solventar de generar de manera automática la creación de backups de base de datos Hana, poder comprimirlos y copiarlos en una ruta donde puedan ser luego cargados en alguna solución de nube como Google Drive o One Drive.

## Funcionamiento
Instalar Node version 16 tls previa verificacion si no lo tiene ... 
se verifica en consola de linux con node --version

La aplicación Node inicia consultando el servidor de base de datos para obtener el query de exportación de cada base de datos del servidor. Luego de esto se ejecuta cada exportación que se obtuvo de la consulta de manera asíncrona. 
Cada ejecución de export devolverá un objeto con las propiedades: status, error y query las cuales brindan información del estado de la ejecución de la exportación de cada base de datos en el server.

Al obtener todas las respuestas ejecutadas de manera asíncrona se procede a ejecutar una función de envío de correo que enviará un archivo JSON con la información de estatus de cada una de las exportaciones. 

Después del envío de este correo se ejecutará una función que llama a un script de Bash el cual busca la carpeta que contiene el backup, comprime esta carpeta en formato tar.gz y copia este zip en la ruta de destino asignada al script la cual debería de estar asignada como carpeta de respaldo en alguna app de alojamiento como Google Drive. 

Finalmente se enviará un correo electrónico informando que se ejecutó correctamente la compresión de la carpeta y su copia a la ruta destino.

## Instalación

Inicialmente se debe de crear una vista en el servidor para obtener todos los QUERYs de exportación de base de datos para su consulta, en el archivo `src/hana/hana.query.js` se encuentra el query que se asigna para verificar que se llame de manera correcta ya sea por vista o procedimiento almacenado.
Ejemplo
````javascript
export const getExports = () => {
    const query = `SELECT "QUERY" FROM "_SYS_BIC"."BACKUPS/GENERATE_BACKUPS"`;
    return query;
};
````

El servidor Linux debe contar con una versión de Node.js igual o superior a la 16.0.0. Se modificarán los archivos `src/bkZips.sh` y `src/run_script.sh` para asignarles la ruta origen del backup de base de datos, la ruta destino del archivo zip, la ruta del archivo de ejecución de Node y la ruta del script `bkZips.sh` respectivamente.
### Instalación de Node.js
Para la instalacion de node.js se utilizara la herramienta `nvm` la cual es un gestor de versiones de node.js el cual puede instalar varias versiones y cambiar entre versiones de node. Acá esta el link de la documentacion de este paquete: [nvm en Github](https://github.com/nvm-sh/nvm).

#### Instalar NVM en SUSE Linux Enterprise 15

Primer paso: abrimos la terminal y ejecutamos el comando que descarga y ejecuta el shell que instala `nvm`:
```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.38.0/install.sh | bash
```

Segundo paso: En SUSE Linux Enterprise 15, es posible que no exista el archivo .bashrc por defecto. Vamos a crearlo y luego aplicarle el comando `source`.
```bash
touch ~/.bashrc
echo 'export NVM_DIR="$HOME/.nvm"' >> ~/.bashrc
echo '[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"' >> ~/.bashrc
source ~/.bashrc
```
Tercero: Cerramos la terminal y la volvemos a abrir para verificar que se ejecute correctamente `nvm`:
```bash
nvm -v
```
Este comando te devolvera la version de `nvm` instalada.

### Instalando Node

Al instalar `nvm` ya se puede proceder a la instalacion de `node.js`, esto por medio de nvm.

Primer paso: En la terminal ejecutamos el comando que lista las versiones lts (long time suppoort) para verificar que versiones tenemos disponibles y seleccionar la que necesitemos (En el caso de estos servidores lo recomendable es la ultima versión LTS del v16):
```bash
nvm ls-remote --lts
```

Segundo paso: Al verificar el numero de version `vxx.xx.x` ejemplo v16.20.2 procedemos a instalar esa versión: 
```bash
nvm install v16.20.2
```

Tercero: Por último verificamos la que se instaló correctamente la version de `node.js`:
```bash
node --version
```
### Configurar ruta de destino para los backup

Luego de instalar `node.js` se debe obtener acceso al servidor de windows que contiene la carpeta con el app para los backups y las carpetas que almacenan los backups de cada servidor.
Para esto se verifica que tanto el servidor windows como el de linux puedan comunicarse por medio de la misma red(Verificar ips y hacer ping a los servidores). Luego acceder Other Locations en el explorador de archivos de linux y dar acceso a la carpeta en cuestión por medio del protocolo smb ejemplo: `smb://ip_o_nombre_de_servidor/c$` e ingresar los credenciales.

### Guardar carpeta app en servidor Linux

Con el acceso al servidor windows donde se encuentra alojada la aplicación y las carpetas de backup de los server se debe copiar la carpeta a una carpeta en la raiz del sistema linux. Crear una carpeta en la raiz del disco y copiar la carpeta de app allí.

### Instalando librerias de la aplicacion 

Despues de tener copiada la carpeta de la aplicacion en el servidor de linux y accesamos a ella. Con click derecho abrimos la terminal y ejecutamos el comando para instalar los modulos de node de la aplicación.

Nota: Para validar si es la carpeta correcta se debe verificar que en esta carpeta exista el archivo `package.json`.
```bash
npm install
```

### Permisos de Archivos .sh

Con la carpeta que contiene el app en linux ingresamos mediante terminal a la carpeta que contiene los archibos `bkZips.sh` y `run_script.sh` y allí darles permisos de ejecución desde la terminal.
```bash
sudo chmod +x bkZips.sh
sudo chmod +x run_script.sh
```

### Modificar archivos

Ya con los permisos listos se modifican los archivos `src/bkZips.sh` , `src/run_script.sh` y `src/config/config.js`. 

Archivo `src/bkZips.sh`: se modifca las rutas origen y destino de los backups segun sea el caso. La ruta destino para obtenerla de manera correcta se debe accesar pormedio del explorador de archivos de linux a esta y abrir la terminal, la terminal te dara la ruta de manera como la necesita el script se copiara la informacion desde la palabra `server=...` en adelante.
```bash
#!/bin/bash

# Ruta de la carpeta con la fecha actual en formato yyyy-mm-dd
ruta_origen="/hana/log/$(date +\%Y-\%m-\%d)"
##cambiar desde la parte de server=... a la carpeta donde caera el backup
ruta_destino="/run/user/$(id -u)/gvfs/smb-share:server=despr-ccc,share=e$/BackupsHana"
```

Archivo `src/config/config.js`: En este archivo se indican los credenciales de conexion a base de datos que corresponden al servidor que se le realizara los backups:
```javascript
export const config = {
    connParams: { 
        serverNode: 'ip o host del serviodr:30015',
        uid: 'usuiario de hana',
        pwd: 'password de hana',    
    }
}
```

Archivo `src\run_script.sh`: En este archivo se modifica la ruta donde se encuentra el ejecutable de la tarea (`app.js`) y se indica la ruta de node para ejecutarlo estas rutas se modifican segun corresponda.
Nota: para obtener la ruta correcta de node se puede copiar la siguiente ruta `/root/.nvm/versions/node/` en el explorador de archivos de linux y validar el nombre de la carpeta que se encuentra ahí y modificar el ejemplo:
```bash
#!/bin/bash
cd  /BackupDev/Desarrollo/src # ruta del proyecto
/root/.nvm/versions/node/v16.15.1/bin/node app.js    # Cambia esto a la ubicación de tu archivo app.js
```
### Configurar tarea cron

Finalmente se  debe configurar el archivo cron del servidor para asignar la hora de ejecución del archivo. Para abrir el archivo cron se abre la terminal desde cualquier ruta y se ejecuta el siguiente comando:
```bash
# Abre el archivo cron con el editor vi
crontab -e
```

Esto abrirá el editor de VIM en la terminal y acá tienes un ejemplo de como debe editarse el archivo: Los 2 primeros asteriscos indica los minutos y la hora a ejecutar y luego se coloca la ruta absoluto del archivo script `src\run_script.sh`.
```bash
# Añade la siguiente línea para ejecutar tu script a las 2 AM todos los días
0 2 * * * /ruta/a/tu/script.sh
```

Con esto ya tendriamos configurada la tarea que ejecuta los backups en hana y comprime en formato `tar.gz` el back para luego enviarlo a la carpeta del servidor Windows dondes se cargara en Google Drive.

## Comandos básicos de vi

Para los usuarios inexpertos en Linux, aquí hay algunos comandos básicos de vi:

- `i`: entra en el modo de inserción, lo que te permite escribir texto.
- `esc`: sale del modo de inserción.
- `:w`: guarda los cambios que has hecho.
- `:q`: sale de vi.
- `:wq`: guarda los cambios y sale de vi.
- `:q!`: sale de vi sin guardar los cambios.

## Contribución

Si deseas contribuir a este proyecto, por favor abre un issue o realiza un pull request.

## Licencia

Este proyecto está licenciado bajo la licencia MIT.

## Contacto

Si tienes alguna pregunta o sugerencia, por favor abre un issue en GitHub.