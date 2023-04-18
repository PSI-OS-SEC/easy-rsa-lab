# Prerequisitos

1. FQDN configurado en sus sistema
2. Acceso a Internet en una VM ubuntu 20.04
3. Paquete instalado de git
4. FQDN Accesible desde su estación de trabajo (puede temporalmente configurar en /etc/hosts o similar)


# Generar CA y Certificados utilizando Easy-rsa-lab 

1. Clonar el repositorio 

```
git clone https://github.com/OpenVPN/easy-rsa.git
```

2. Ingresar al directorio de easyrsa3

```
cd easy-rsa/easyrsa3
```

3. Inicializar

```
./easyrsa init-pki
```

4. Personalizar configuración

Modificar archivo ```pki/vars```

```
set_var EASYRSA_DN      "org"

set_var EASYRSA_REQ_COUNTRY     "GT"
set_var EASYRSA_REQ_PROVINCE    "Guatemala"
set_var EASYRSA_REQ_CITY        "Guatemala"
set_var EASYRSA_REQ_ORG         "PSI"
set_var EASYRSA_REQ_EMAIL       "email"
set_var EASYRSA_REQ_OU          "OSSEC2"

set_var EASYRSA_ALGO            rsa
set_var EASYRSA_NS_COMMENT      "CA PoC"

```

5. Generar CA

Common Name: Test-PSI CA

```
./easyrsa build-ca
```


6. Consultar CA Creado

```
./easyrsa show-ca

```

7. Consultar Certificado CA utilizando openssl

Issuer:
```
openssl x509 -in pki/ca.crt -issuer -noout
```
Subject:
```
openssl x509 -in pki/ca.crt -subject -noout
```
EndDate:
```
openssl x509 -in pki/ca.crt -enddate -noout
```

8. Generar CSR (EasyRSA)

```
./easyrsa gen-req FQDN
```
Por defecto la llave esta encriptada, si desea generarla sin password ejecutar:

```
./easyrsa gen-req FQDN nopass
```

9. Consultar Llave Privada

```
head -n1 pki/private/FQDN.key 
```
Mostrará: ```-----BEGIN ENCRYPTED PRIVATE KEY-----```, si está encriptada.

Puede eliminar el password utilizando:
```
openssl rsa  -in pki/private/FQDN.key -out pki/private/FQDN-no-pass.key
```
Mostrará: ```-----BEGIN RSA PRIVATE KEY-----```, si no está encriptada


10. Consultar CSR

```
./easyrsa show-req FQDN
```

11. Firmar Certificado para WebServer TLS

```
./easyrsa sign-req server FQDN
```

12. Consultar Certificado

```
./easyrsa show-cert FQDN
```

13. Validar firma de Certificado por el CA

```
openssl verify -CAfile pki/ca.crt pki/issued/FQDN.crt
```
Debe mostrar: ```pki/issued/FQDN.crt: OK```

# Utilizar el Certificado en Un WebServer

1. Instalar apache2
```
sudo apt-get install apache2
```
2. Activar modulo de ssl y sitio por defecto ssl
```
sudo a2enmod ssl
sudo a2ensite default-ssl
sudo systemctl restart apache2
```
3. Consultar Certificado por defecto Apache

```
openssl s_client -connect localhost:443 -showcerts
```
Buscar en la salida el texto:
* verify return
* issuer
* subject
* ```-----BEGIN CERTIFICATE-----```
* ```-----END CERTIFICATE-----```

Puede observar que no existe un Certificado que valide al emisor (Issuer), en este caso es un certificado autofirmado.

4. Consultar el servicio web en HTTPS

```
curl -I -v  https://FQDN
```

Este comando devuelve el resultado:

```
curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the web page mentioned above.
```

Al ser un certificado no confiado por algún CA, cuyo certificado se encuentre en el sistema, no es posibe conectarnos sin desactvar la verificación del certificado.

```
curl -I -v  https://FQDN -k
```
5. Remplazar certificado por defecto por el certificado firmado por el nuevo CA.

```
sudo cp pki/private/FQDN.key /etc/ssl/private/ 
sudo cp pki/issued/FQDN.crt /etc/ssl/certs/
sudo sed -i 's/ssl-cert-snakeoil/FQDN/g' /etc/apache2/sites-available/default-ssl.conf
sudo sed -i 's/FQDN.pem/FQDN.crt/g' /etc/apache2/sites-available/default-ssl.conf
sudo systemctl restart apache2
```

6. Repita el paso 3

Buscar en la salida el texto:
* verify return
* issuer
* subject
* ```-----BEGIN CERTIFICATE-----```
* ```-----END CERTIFICATE-----```

7. Repita el paso 4 y 5

El Certificado ha sido actualizado, pero aun no se valida el CE, ya que este no es parte de los CAs reconocidos por el sistema.

8. Agregar CAs al sistema (Linux Shell)

```
sudo cp pki/ca.crt /usr/local/share/ca-certificates/
sudo update-ca-certificates
```
9. Repita el paso 4

Deberá reconocer el certificado emitido por un CA reconocido por el Sistema Operativo.

10. Agregue el CA a su Cliente (Firefox)

Settings -> Certificates -> View Certificates -> Authorities -> Import

![Importar](images/import.png)

Verificar Certificado

![Verificar](images/valid.png)
