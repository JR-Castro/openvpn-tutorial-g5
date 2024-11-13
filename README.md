# openvpn-tutorial-g5

OpenVPN tutorial for client-site and site-site connection

# CA Set-up

Primero necesitamos un servidor que sirva como CA para firmar nuestros certificados. Para esto vamos a usar easy-rsa.

```bash
sudo apt install easy-rsa
make-cadir ~/easy-rsa
cd ~/easy-rsa
./easy-rsa init-pki
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

## Server Set-Up

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

push "route 172.31.0.0 2555.255.240.0"
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

### Otorgar IP de la red interna por DHCP a Clients

Para otorgar IP por DHCP a la red interna del sitio, es necesario configurar un bridge y utilizar la interface TAP. Para crear el puente usaremos el paquete `bridge-utils`

Para llevar a cabo la configuracion debemos conocer algunos parametros:
|Setting|param name| value |
|-------|----------|-------|
|Ethernet Interface|eth|enp3s0|
|Router IP Address|r_ip|192.168.0.1|
|Local IP Address|eth_ip|192.168.0.58|
|Local Netmask|eth_netmask|255.255.255.0|
|Local Broaadcast Address|eth_broadcast|192.168.0.255|
|VPN client address pool start|pool_start|192.168.0.100|
|VPN client address pool end|pool_end|192.168.0.110|
|Virtual Bridge Interface|br|br0|
|Virtual TAP Interface|tap|tap0|

En base a estos parametros podemos crear el script `bridge-start.sh` en `/etc/openvpn/`:

```bash
#!/bin/bash

# Definimos la interface del bridge
br="br0"

# Definimos una lista de interfaces TAP a las
# cuales le hacemos bridge,
# por ej. tap="tap0 tap1 tap2"
tap="tap0"

# Definimos interfaces de ethernet fisicas a las
# cuales le hacemos bridge con la/s interface/s TAP
# definida/s anteriormente
eth="enp3s0"
eth_ip="192.168.0.58"
eth_netmask="255.255.255.0"
eth_broadcast="192.168.0.255"

for t in $tap; do
    openvpn --mktun --dev $t
done

brctl addbr $br
brctl addif $br $eth

for t in $tap; do
    brctl addif $br $t
done

for t in $tap; do
    ifconfig $t 0.0.0.0 promisc up
done

ifconfig $eth 0.0.0.0 promisc up

ifconfig $br $eth_ip netmask $eth_netmask broadcast $eth_broadcast
```

Y el script `bridge-stop.sh` para finalizar el ethernet bridging:

```bash
#!/bin/bash

# Definimos la interface del bridge
br="br0"

# Definimos una lista de interfaces TAP a las
# cuales les hicimos bridge,
# por ej. tap="tap0 tap1 tap2"
tap="tap0"

ifconfig $br down
brctl delbr $br

for t in $tap; do
    openvpn --rmtun --dev $t
done
```

Luego, podemos utilizar la configuracion de server que se encuentra en [/config-files/server/server-tap.conf](https://github.com/JR-Castro/openvpn-tutorial-g5/blob/main/config-files/server/server.conf) con la siguiente modificacion:

En la linea 139, modifiquela con los valores que correspondan:

```bash
server-bridge {eth_ip} {eth_netmask} {pool_start} {pool_end}
```

Una vez hecho esto, se puede configurar para cada cliente en el `ccd` una IP estatica dentro del sitio, como se ve en [/config-files/server/ccd-client-site/client1](https://github.com/JR-Castro/openvpn-tutorial-g5/blob/main/config-files/server/ccd-client-site/client1) con modificando el comando en este de la siguiente manera:

```bash
ifconfig-push {ip_in_pool} {eth_netmask}
```

Por ejemplo, `ip_in_pool = 192.168.0.101`.

#TODO: PROBAR USAR EN LA CONF `push "redirect-gateway def1 bypass-dhcp" PARA NO REQUERIR UNA CONF POR CADA USUARIO

Ahora ya podemos comenzar el bridge y el server de OpenVPN con los siguientes comandos:

```bash
sudo /etc/openvpn/bridge-start.sh
sudo systemctl start openvpn@server
```

Si ocurren problemas de conexion, utilice los siguientes comandos:

```bash
route add default gw {r_ip} br0
sudo systemd-resolve --flush-caches
```

Y para dar cierre se puede utilizar el script de `bridge_stop.sh`.

## Client Set-Up

```bash
sudo apt update
sudo apt install openvpn
sudo cp ~/ca.crt ~/client1.key ~/client1.crt ~/ta.key /etc/openvpn/client
sudo cp /usr/share/doc/openvpn/examples/sample-config-files/client.conf /etc/openvpn/client1.conf
sudo vim /etc/openvpn/client1.conf
```

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

```bash
sudo systemctl start openvpn
```

## Proxy Set-Up

### Proxy

```bash
sudo apt install squid
sudo vim /etc/squid/squid.conf
```

Tenemos que descomentar la linea

```
hettp_access allow localnet
```

### Server

Si se desea que ademas de que puedan acceder al proxy, se puede forzar a redirigir el default gateway por el VPN, es decir que el trafico web normal (DNS y HTTP) ahora van a pasar por el server VPN. También es posible redirigir este trafico por el proxy que levantamos en la instancia, seteandolo como un proxy en cada cliente:

```bash
curl -x http://<proxy-ip>:3128 ipinfo.io

export http_proxy="http://<proxy-ip>:3128"
export https_proxy="http://<proxy-ip>:3128"
export HTTP_PROXY=${http_proxy}
export HTTPS_PROXY=${https_proxy}
```

# OpenVPN Site to Site Set-Up

# Server

```bash
# Descomentar la linea
sudo vim /etc/openvpn/server1.conf
```

Agregamos las lienas

```ovpn
client-config-dir ccd
route 172.32.0.0 255.255.0.0
```

Creamos el directorio `ccd`, y por cada cliente creamos un archivo con su `cname`.

```ovpn
# client1
iroute 172.32.0.0 255.255.255.0
```
