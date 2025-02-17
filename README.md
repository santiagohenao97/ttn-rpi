# Guía de instalación y configuración de The Things Stack

Esta guía cubre los pasos para configurar The Things Stack en una Raspberry Pi con Docker y Docker Compose.

## Paso 1: Actualizar y preparar el sistema

Primero, actualiza y actualiza los paquetes de tu sistema:

```bash
sudo apt update && sudo apt upgrade -y
```

Luego, instala los paquetes necesarios:

```bash
sudo apt install -y curl git docker.io docker-compose
```

Asegúrate de que Docker esté habilitado y en ejecución:

```bash
sudo systemctl enable docker
sudo systemctl start Docker
```

## Paso 2: Revisar e instalar cfssl

cfssl es una herramienta utilizada para manejar certificados en el flujo de instalación de The Things Stack. Sigue estos pasos para descargar e instalar cfssl y cfssljson:

```bash
curl -L https://pkg.cfssl.org/R1.2/cfssl_linux-arm -o cfssl
curl -L https://pkg.cfssl.org/R1.2/cfssljson_linux-arm -o cfssljson
chmod +x cfssl cfssljson
sudo mv cfssl cfssljson /usr/local/bin/
```

Verifica la instalación de cfssl:

```bash
cfssl version
cfssljson -version
```

## Paso 3: Crear directorios y archivos de configuración

Crea los directorios necesarios para la configuración:

```bash
cd /home/pi
mkdir the-thing-stack
mkdir config
cd config/
mkdir stack
cd ..
cd the-thing-stack
```

Dentro del directorio `the-thing-stack`, crea el archivo `docker-compose.yml` (plantilla).

```bash
cd config/stack/
nano ttn-lw-stack-docker.yml
```

## Paso 4: Configurar los certificados

En el directorio `the-things-stack/`, crea y edita el archivo `ca.json` para generar la clave de la autoridad certificadora (CA):

```bash
nano ca.json
```

Contenido de `ca.json`:

```json
{
  "names": [
    {
      "C": "NL",
      "ST": "Noord-Holland",
      "L": "Amsterdam",
      "O": "The Things Demo"
    }
  ]
}
```

Genera la clave de la CA:

```bash
cfssl genkey -initca ca.json | cfssljson -bare ca
```

Luego, crea el archivo `cert.json` para el certificado:

```bash
nano cert.json
```

Contenido de `cert.json`:

```json
{
  "hosts": ["192.168.1.12"],
  "names": [
    {
      "C": "NL",
      "ST": "Noord-Holland",
      "L": "Amsterdam",
      "O": "The Things Demo"
    }
  ]
}
```

Genera el certificado utilizando la clave de la CA:

```bash
cfssl gencert -ca ca.pem -ca-key ca-key.pem cert.json | cfssljson -bare cert
```

Renombra el archivo de clave y configura los permisos:

```bash
mv cert-key.pem key.pem
sudo chown 886:886 ./cert.pem ./key.pem
```

## Paso 5: Ejecutar Docker Compose

Primero, descarga las imágenes necesarias con docker-compose:

```bash
sudo docker-compose pull
```

Ejecuta las migraciones de la base de datos:

```bash
sudo docker-compose run --rm stack is-db migrate
```

Crea el usuario administrador:

```bash
sudo docker-compose run --rm stack is-db create-admin-user --id admin --email your@email.com
```

Si ocurre un error, intenta nuevamente.

Crea el cliente OAuth para la interfaz de línea de comandos (CLI):

```bash
sudo docker-compose run --rm stack is-db create-oauth-client --id cli --name "Command Line Interface" --owner admin --no-secret --redirect-uri "local-callback" --redirect-uri "code"
```

Finalmente, crea el cliente OAuth para la consola web:

```bash
SERVER_ADDRESS=http://192.168.1.12
ID=console
NAME=Console
CLIENT_SECRET=console
REDIRECT_URI=${SERVER_ADDRESS}/console/oauth/callback
REDIRECT_PATH=/console/oauth/callback
LOGOUT_REDIRECT_URI=${SERVER_ADDRESS}/console
LOGOUT_REDIRECT_PATH=/console

sudo docker-compose run --rm stack is-db create-oauth-client   --id ${ID}   --name "${NAME}"   --owner admin   --secret "${CLIENT_SECRET}"   --redirect-uri "${REDIRECT_URI}"   --redirect-uri "${REDIRECT_PATH}"   --logout-redirect-uri "${LOGOUT_REDIRECT_URI}"   --logout-redirect-uri "${LOGOUT_REDIRECT_PATH}"
```

## Paso 6: Iniciar The Things Stack

Por último, levanta el servicio con Docker Compose:

```bash
sudo docker-compose up
```

Con esto, The Things Stack debería estar funcionando correctamente en tu Raspberry Pi.

## Notas

- Asegúrate de reemplazar las direcciones IP, correos electrónicos y contraseñas según sea necesario.
- Si encuentras problemas con el acceso, verifica los logs de Docker para obtener más información.
