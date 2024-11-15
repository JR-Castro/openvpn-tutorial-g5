# openvpn-tutorial-g5

OpenVPN tutorial para conexion client-site and site-site.

**Redes de Informacion - 2024-2Q**

Grupo 5:

- Castro, Juan Ramiro
- Ricarte, Matias Agustin

# CA Set-up

Primero necesitamos un servidor que sirva como CA para firmar nuestros certificados. Para esto vamos a usar easy-rsa.

```bash
sudo apt install easy-rsa
make-cadir ~/easy-rsa
cd ~/easy-rsa
./easyrsa init-pki
```

También vamos a usar una key para HMAC para usar con TLS/SSL. Esto hace que cualquier paquete UDP que no tenga el tag HMAC correcto sea droppeado sin procesar, protegiendonos de ataques DoS, port flooding, port scanning, vulnerabilidades de buffer overflow en la implementación de TLS/SSL, y previniendo TLS/SSL handshakes de clientes sin permisos, descartandolos antes de que inicien. Para generar la clave vamos a usar el siguiente comando:

```bash
openvpn --genkey secret ta.key
```

Editar el archivo vars para que tenga la siguiente configuración. Tambien para mejorar la seguridad vamos a hacer uso de criptografía basada en curvas elipticas y usando SHA512 para hashear.

```
set_var EASYRSA_REQ_COUNTRY    "COUNTRYCODE"
set_var EASYRSA_REQ_PROVINCE   "PROVINCE"
set_var EASYRSA_REQ_CITY       "CITY"
set_var EASYRSA_REQ_ORG        "ORGANIZATION"
set_var EASYRSA_REQ_EMAIL      "admin@example.com"
set_var EASYRSA_REQ_OU         "ORGANIZATIONAL_UNIT"
set_var EASYRSA_ALGO           "ec"
set_var EASYRSA_DIGEST         "sha512"
```

El proximo comando va a pedir una passphrase que luego debe ser usada para firmar los certificados, así que se recomienda recordarla o agregar el parametro `nopass`.

```bash
./easy-rsa build-ca
```

Luego, para crear un y firmar un certificado solo es necesario seguir los siguientes pasos, donde $CERT_NAME es el nombre del certificado que queramos generar, para poder distinguirlos.

```bash
./easyrsa gen-req $CERT_NAME nopass
```

Una vez generado un certificado, para firmarlo en caso de ser para el servidor podemos usar:

```bash
./easy-rsa sign-req server $CERT_NAME
```

Luego, para cada cliente podemos generar las claves necesarias utilizando los siguientes comandos:

```bash
mkdir -p ~/client-configs/keys
chmod -R 700 ~/client-configs
./easy-rsa gen-req $CLIENT_NAME nopass
./easy-rsa sign-req client $CLIENT_NAME
```

Y luego habra que copiar los archivos `~/easy-rsa/pki/ca.crt`, `~/easy-rsa/pki/issued/$CERT_NAME.crt`, `~/easy-rsa/private/$CERT_NAME.key` y `~/easy-rsa/ta.key` a cada maquina correspondiente.

# OpenVPN Client-Site Setup

## TAP

TAP nos permite realizar un bridge de Ethernet a traves del tunel VPN, lo cual nos permitiria hacer como si el cliente estuviera en la misma LAN que el servidor, puediendo recibir mensajes desde otros dispositivos del lado del server sin problema alguno.

### Server Set-Up

Primero tenemos que asegurarnos que nuestro servidor tenga OpenVPN, net-tools y bridge-utils

```bash
sudo apt update && sudo apt install -y openvpn net-tools
sudo cp /usr/share/doc/openvpn/examples/sample-config-files/server.conf /etc/openvpn/server-tap.conf
```

#### Bridge

Primero hay que crear un ethernet-bridge en el servidor entre la interfaz TAP y la interfaz que use el servidor para conectarse a la LAN que se quiere compartir. Hay que obtener primero el nombre de la interfaz de ethernet (`eth0`), la IP local (`192.168.0.35`), la mascara local (`255.255.255.0`), la dirección de broadcast (`192.168.0.255`), el default gateway (`192.168.0.1`), las direcciones que se le van a asignar a los clientes (`192.168.0.100` a `192.168.0.124`), la interfaz de bridge (`br0`) y la interfaz TAP (`tap0`).

Editar el script [bridge-start](./config-files/server/bridge-start.sh) con los valores acordes a nuestra red, y luego correr los siguientes comandos, avisamos que probablemente perdera la conexión a internet por unos momentos.

```bash
sudo ./bridge-start.sh
sudo route add default gw 192.168.0.1 br0
# Hay que resetear los caches de DNS por que si no nos quedamos sin acceso a internet
sudo sudo systemd-resolve --flush-caches
sudo iptables -A INPUT -i tap0 -j ACCEPT
sudo iptables -A INPUT -i br0 -j ACCEPT
sudo iptables -A FORWARD -i br0 -j ACCEPT
```

Para más información recomendamos leer la documentación de [Ethernet Bridging](https://openvpn.net/community-resources/ethernet-bridging/)

#### OpenVPN

```bash
# Tambien nos tenemos que asegurar de copiar los archivos ca.crt, server1.crt, server1.key y ta.key a la carpeta de configuración de openvpn, en nuestro caso se los madnamos al server usando scp y los dejamos en el home del usuario ubuntu.
sudo cp ~/ca.crt ~/server1.crt ~/server1.key ~/ta.key /etc/openvpn
sudo vim /etc/openvpn/server-tap.conf
```

Pueden encontrar un archivo de configuración de ejemplo en [server-tap](./config-files/server/server-tap.conf), pero los siguientes valores son los más importantes a la hora de configurar:

```conf
port 1194
proto udp

dev tap0

ca /etc/openvpn/ca.crt
cert /etc/openvpn/server1.crt
key /etc/openvpn/server1.key
dh none
data-ciphers AES-256-GCM
auth SHA256


# El primer valor es la IP del OpenVPN server, despúes la mascara de la red (en nuestro caso es 192.168.0.0/24) y despues el rango de IPs que se les quieren asignar a los clientes
server-bridge 192.168.0.38 255.255.255.0 192.168.0.100 192.168.0.124
# Para poder asignar una IP especifica a cada cliente se puede crear un archivo <cname> por cliente en el client-config-dir para asignarlas.
client-config-dir ccd-client-site


# Si lo que se quiere es que reciban la configuración por medio de un servidor DHCP del lado del servidor
;server-bridge

tls-auth /etc/openvpn/ta.key 0
```

Una vez hecho esto, se puede configurar para cada cliente en el `ccd` una IP estatica dentro del sitio, como se ve en [/config-files/server/ccd-client-site/client1](https://github.com/JR-Castro/openvpn-tutorial-g5/blob/main/config-files/server/ccd-client-site/client1) de la siguiente manera:

```bash
ifconfig-push {ip_in_pool} {eth_netmask}
```

Por ejemplo, `ip_in_pool = 192.168.0.101`, `eth_netmash=255.255.255.0`.

Para levantar el servidor se usa:

```bash
sudo systemctl start openvpn@server-tap
```

Y también hay que asegurarse de encender el forwarding de paquetes:

```bash
sudo vim /etc/sysctl.conf
# Desomentar la linea
#net.ipv4.ip_forward=1
sudo sysctl -p /etc/sysctl.conf
```

### Client Set-Up

Del lado del cliente la configuración es más facíl, solo necesitamos indicar que se va a usar la interfaz TAP, el protocolo, la IP y puerto del servidor y los certificados y claves a utilizar. Se puede encontrar el ejemplo completo en [client-tap.conf](./config-files/client/client-tap.conf).

```bash
# Obtenemos el archivo de ejemplo de configuración
sudo cp /usr/share/doc/openvpn/examples/sample-config-files/client.conf /etc/openvpn/client-tap.conf
sudo vim /etc/openvpn/client-tap.conf
```

Los siguientes son los valores más relevantes a modificar en el archivo de configuración:

```conf
client
dev tap
proto udp
remote <public_server_ip> <port>
ca /etc/openvpn/client/ca.crt
cert /etc/openvpn/client/client1.crt
key /etc/openvpn/client/client1.key
data-ciphers AES-256-GCM
auth SHA256
tls-auth /etc/openvpn/client/ta.key 1
```

Luego, para levantar el cliente y verificar su funcionamiento podemos realizar los siguientes comandos:

```bash
sudo systemctl start openvpn@client-tap
ifconfig # Deberiamos ver una interfaz tap con IP de la LAN del servidor
ping 192.168.0.1 # Deberia funcionar
```

## TUN

TUN, en cambio, tiene una configuración más facíl que nos permite asignarle una IP de la subred del túnel (10.8.0.0/24) a cada cliente, y solo podemos enviar paquetes IP a través de esta interface.

### Server Set-Up

Ahora en el servidor, podemos empezar a realizar nuestra configuración de OpenVPN.

```bash
sudo apt update
sudo apt install openvpn net-tools
sudo cp /usr/share/doc/openvpn/examples/sample-config-files/server.conf /etc/openvpn/server1.conf
# Tambien nos tenemos que asegurar de copiar los archivos ca.crt, server1.crt, server1.key y ta.key a la carpeta de configuración de openvpn, en nuestro caso se los madnamos al server usando scp y los dejamos en el home del usuario ubuntu.
sudo cp ~/ca.crt ~/server1.crt ~/server1.key ~/ta.key /etc/openvpn
sudo vim /etc/openvpn/server1.conf
```

En `config-files/server.conf` se puede encontrar el archivo de configuración entero, pero las partes más relevantes son estas:

```conf
port 1194
proto udp

dev tun

ca /etc/openvpn/ca.crt
cert /etc/openvpn/server1.crt
key /etc/openvpn/server1.key

dh none

# Vamos a usar AES-256-GCM junto a SHA256 para más seguridad, usando claves de mayor tamaño (default es Blowfish, de 128 bits).
# Esto es importante despues replicarlo en el cliente.
# En caso de tener clientes más viejos puede llegar a generar problemas de incompatibilidad
data-ciphers AES-256-GCM
auth SHA256

topology subnet

server 10.8.0.0 255.255.255.0

push "route 172.31.0.0 255.255.240.0"
# Redirecciona el trafico de todos los cleintes a traves de la VPN, incluido DNS
# En el caso de que un cliente sea una instancía EC2, para poder conectarte vas a tener que realizar SSH a traves del servidor OpenVPN.
;push "redirect-gateway def1 bypass-dhcp"

# Esto permite que varios clientes usen el mismo certificado, NO USAR EN PRODUCCIÓN, solo para testing.
;duplicate-cn

tls-auth /etc/openvpn/ta.key 0

# Archivo que muestra el estado de las conexiónes cada minuto
status /var/log/openvpn/openvpn-status.log
# Archvio de logs
log /var/log/openvpn/openvpn.log
# Si queres que los logs se appendeen en vez de truncarse cuando se reinicia el servicio.
;log-append /var/log/openvpn/openvpn.log

# Nivel de logs, [0-9]
verb 3
```

Tambien tenemos que permitir el forwarding de paquetes en el servidor:

```bash
# Descomentar la siguiente linea:
#net.ipv4.ip_forward=1
sudo vim /etc/sysctl.conf
sudo sysctl -p /etc/sysctl.conf
```

Y para poder permitir que las maquinas en la LAN del server respondan a lo que reciben, hay que agregar una regla al gateway de la LAN para que rutee lo que viene por la subred del VPN (10.8.0.0/24) a la IP privada del servidor de OpenVPN

Otra posibilidad es hacer NAT con `sudo iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o enX0 -j MASQUERADE`, pero es temporal, hay que reconfigurarlo en cada reinicio del servidor o hacerlo permanente.

Finalmente podemos empezar el servicio de OpenVPN:

```bash
sudo systemctl start openvpn@server
#ifconfig deberia mostrar una interfaz tun
```

### Autenticación Externa

Por ejemplo, si somos una compañía, y queremos verificar que los que se conectan a nuestro VPN pertenecen a nuestra organización mediante el uso de un servidor RADIUS o algún otro lugar donde queramos verificar las credenciales, podemos hacerlo mediante la siguiente configruación.

Para comenzar este ejemplo, primero vamos a levantar un freeradius en el webserver-1, que esta en la misma red que el servidor OpenVPN

```bash
sudo apt update && sudo apt install -y freeradius
```

Luego tenemos que editar el archivo `/etc/freeradius/3.0/clients.conf` y agregar un permiso para que nuestro servidor OpenVPN se pueda conectar. Por defecto también esta permitido que `localhost` se conecte con password `testing123`.

```conf
client openvpn-server {
    ipaddr          = 10.0.5.106
    secret          = totallySecret
    require_message_authenticator = no
    shortname       = specific-client
}
```

Ahora vamos a agregar al menos un cliente en el archivo `/etc/freeradius/3.0/users`

```conf
testuser Cleartext-Password := "1234567890"
```

Ahora, reiniciando el servicio de freeradius y testeando con el siguiente comando, deberiamos obtener el resultado de que las credenciales son correctas:

```bash
sudo systemctl restart freeradius
radtest testuser 1234567890 127.0.0.1 0 testing123
# Sent Access-Request Id 59 from 0.0.0.0:37807 to 127.0.0.1:1812 length 78
# 	User-Name = "testuser"
# 	User-Password = "1234567890"
# 	NAS-IP-Address = 10.0.14.27
# 	NAS-Port = 0
# 	Message-Authenticator = 0x00
# 	Cleartext-Password = "1234567890"
# Received Access-Accept Id 59 from 127.0.0.1:1812 to 127.0.0.1:37807 length 38
# 	Message-Authenticator = 0x03479d6753a071ff0d8d88d1eecf0734
```

Ahora, en el servidor de OpenVPN:

```bash
sudo apt update && sudo apt install -y freeradius-utils
sudo touch /etc/openvpn/auth.sh
sudo chmod +x /etc/openvpn/auth.sh
sudo vim /etc/openvpn/auth.sh
```

```bash
#!/bin/bash
x="radtest $username $password 10.0.14.27 1812 totallySecret"
y=$(eval "$x")

if [[ $y == *"Access-Accept"* ]]; then
  exit 0
        else
  exit 1
fi
```

Ahora vamos a editar `/etc/openvpn/server.conf` para hacer uso del script para verificar el usuario con RADIUS:

```conf
# Agregamos esta linea
auth-user-pass-verify /etc/openvpn/auth.sh via-env
script-security 3
```

Y ahora iniciamos el servidor

```bash
sudo systemct start openvpn@server
```

Luego en el cliente hay que poner la siguiente configuración:

```conf
; Asi pide las credenciales
auth-user-pass
;auth-user-pass /etc/openvpn/auth.txt
```

Donde `/etc/openvpn/auth.txt` tiene las siguientes dos lienas:

```
testuser
1234567890
```

## Client Set-Up

```bash
sudo apt update && sudo apt install -y openvpn
# Necesitamos mandar estos archivos por alguna forma segura (como SCP) al client
sudo cp ~/ca.crt ~/client1.key ~/client1.crt ~/ta.key /etc/openvpn/client
sudo cp /usr/share/doc/openvpn/examples/sample-config-files/client.conf /etc/openvpn/client1.conf
sudo vim /etc/openvpn/client1.conf
```

El archivo de configuración usado esta en [client-site-site.conf](./config-files/client/client-site-site.conf) pero estos son los valores relevantes:

```ovpn
client
dev tun
proto udp
data-ciphers AES-256-GCM
auth SHA256
remote <public_server_ip> <port>
ca /etc/openvpn/client/ca.crt
cert /etc/openvpn/client/client1.crt
key /etc/openvpn/client/client1.key
tls-auth /etc/openvpn/client/client1.key 1
verb 3
```

Iniciamos la conexión del lado del cliente:

```bash
sudo systemctl start openvpn@client-site-site
```

## Proxy Set-Up

### Proxy

```bash
sudo apt install squid
sudo vim /etc/squid/squid.conf
```

Tenemos que descomentar la linea

```
http_access allow localnet
```

### Server

Si se desea que ademas de que puedan acceder al proxy, se puede forzar a redirigir el default gateway por el VPN, es decir que el trafico web normal (DNS y HTTP) ahora van a pasar por el server VPN. En el archivo de configuración cambiar el valor deseado:

```
# Esto redirige el default gateway del cliente por le VPN, causando que todo el tráfico IP como web browsing pase por el VPN
;push "redirect-gateway def1 bypass-dhcp"
# Esto permite indicarle a los clientes que deben uar un proxy http
;push "dhcp-option PROXY_HTTP 10.144.5.14 3128"
;push "dhcp-option PROXY_HTTPS 10.144.5.14 3128"
;push "dhcp-option DNS 1.2.3.4"
```

### Client

También es posible redirigir este trafico por el proxy que levantamos en la instancia, seteandolo como un proxy en cada cliente, sin forzar el cambio:

```bash
curl -x http://<proxy-ip>:3128 ipinfo.io

export http_proxy="http://<proxy-ip>:3128"
export https_proxy="http://<proxy-ip>:3128"
export HTTP_PROXY=${http_proxy}
export HTTPS_PROXY=${https_proxy}
```

# OpenVPN Site to Site Set-Up

## Server

```bash
sudo cp /usr/share/doc/openvpn/examples/sample-config-files/server.conf /etc/openvpn/server-site-to-site.conf
sudo vim /etc/openvpn/server-site-to-site.conf
```

Es muy similar a la conexión mediante TUN realizada anteriormente, y se puede ver el archivo de configuración usado en [server-site-to-site.conf](./config-files/server/server-site-to-site.conf), pero basicamnte solo cambian estas lineas en comparación:

```ovpn
server 10.8.0.0 255.255.255.0
push "route 10.0.0.0 255.255.0.0"

client-config-dir ccd-site-site
route 10.1.0.0 255.255.0.0
```

Creamos el directorio `ccd-site-site`, y por cada cliente creamos un archivo con su `cname`.

```ovpn
# client1
iroute 10.1.0.0 255.255.0.0
```

Y también hay que asegurarse de encender el forwarding de paquetes:

```bash
sudo vim /etc/sysctl.conf
# Desomentar la linea
#net.ipv4.ip_forward=1
sudo sysctl -p /etc/sysctl.conf
```

## Client

El cliente se conecta de la misma forma que en el formato Cliente - Sitio, pero ahora hay que encender el forwarding de paquetes de su lado también:

```bash
sudo vim /etc/sysctl.conf
# Desomentar la linea
#net.ipv4.ip_forward=1
sudo sysctl -p /etc/sysctl.conf
```

## Routing

Para asegurarse que puedan alcanzarse, hay que agregar las siguientes reglas a los default gateways de ambas redes:

- Agregar que el tráfico dirigido a 10.1.0.0/16 y 10.8.0.0/24 sea enviado a traves del servidor OpenVPN en la red 10.0.0.0/16
- Agregar que el tráfico dirigido a 10.0.0.0/16 y 10.8.0.0/24 sea enviado a través del cliente OpenVPN en la red 10.1.0.0/16

En caso de que el servidor y cliente sean instancias de EC2, hay que apagar el checkeo de entrada/salida de paquetes: Acciones > Redes > Cambiar comprobación de origen y destino > Detener > Guardar.
