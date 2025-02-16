# bind-DNS
Un DNS configurado para un servidor Linux RHEL/CentOS/Fedora
Hecho en Fedora server (misma sintaxis que CentOS y RHEL)

Pasos a seguir:

1) sudo su
2) yum update
3) yum install bind bind-utils
5) netstat -antp

Debería verse el "named" en el puerto 53 de nuestra interfaz loopback.

Ahora especificamos la dirección del servidor PRINCIPAL (.85.131 de ahora en adelante).

6) nano named.conf y:
   
                     reemplazar ---> listen-on port 53 { 127.0.0.1; };
                            por ---> listen-on port 53 { 127.0.0.1; 192.168.85.131; };

                     reemplazar ---> allow-query { localhost; }
                            por ---> allow-query { localhost; 192.168.85.0/24; }

9) systemctl restart named

Verificar que el puerto 53 está escuchando con la IP de nuestro servidor DNS primario (.131)
8) netstat -antp 

Para configurar el dominio de nuestro nuestro servidor vamos al directorio donde indicamos los dominios.
9) cd /var/named
10) cp -rf named.localhost fedoraserver.com.db
11) nano fedoraserver.com y:

                            reemplazar ---> IN SOA rname.invalid. (
                                   por ---> IN SOA ns1.fedoraserver.com. admin.fedoraserver.com. (

También agregar las siguientes líneas:

                         --->           IN	NS	ns1.fedoraserver.com.
                         ---> ns1	IN	A	192.168.85.131 
                         ---> pc1	IN	A	192.168.85.130

Donde ns1 es la IP de nuestro servidor y pc1 es la dirección de un cliente de prueba. Estos nombres de ns1 y pc1 en esta configuración son importantes porque ahora las direcciones .131 y .130 se pueden manejar como ns1.fedoraserver.com y pc1.fedoraserver.com respectivamente. Cosa que es muy conveniente para un entorno empresarial donde hay muchos hosts. Un ejemplo interesante sería:

                         ---> www	IN	A	x.x.x.x

Y así tendríamos el dominio `www.fedoraserver.com` dentro de nuestra red. Hay una alternativa más adelante para el reverse DNS que sirve un propósito similar. Para que se efectúen los cambios:

12) chown named:named `fedoraserver.com`
13) systemctl restart named
14) netstat -antu 

Revisar que en el puesto 53 nuestro servidor esté presente.

Finalmente, para que nuestro servidor DNS principal reconozca estos dominios.
15) nano /etc/named.conf y agregar:

 					zone "fedoraserver.com" {
        					type master;
        					file "fedoraserver.com.db";
        					allow-transfer { none; };
					};

Para crear uno secundario solo se repiten los pasos anteriores en OTRA MÁQUINA y ahora:

16) nano /etc/named.conf y agregar:

 					zone "fedoraserver.com" {
        					type slave;
        					file "slaves/fedoraserver.com.db";
        					masters { 192.168.85.131; };
					};

De esta forma, esta segunda máquina será un servidor DNS secundario/slave. Completar esta configuración haciendo el siguiente cambio en el DNS primario de nuestra red.

17) nano /etc/named.conf y:

                           reemplazar ---> allow-transfer { none; };
                                  por ---> allow-transfer { 192.168.85.132; };

Suponiendo que la .132 es la dirección del DNS secundario/slave. Podemos agregar también un DNS "superior" que sirva como reespaldo adicional al servidor principal. Este servidor "superior" puede verse como el IPS.

18) nano /etc/named.conf y agregar:
    
					forwarders { 8.8.8.8; };

Es importante tener funciones de reverse DNS para poder identificar los dominios asociados a cada dirección IP en ese orden y no al revés. Para esta configuración debemos dirigirnos al servidor DNS principal.
19) nano /etc/named.conf y agregar:

 					zone "0.168.192.in-addr.arpa" {
        					type master;
        					file "0.168.192.db";
        					allow-transfer { 192.168.85.132; };
					};
20) cd /var/named
21) cp -rf named.loopback 0.168.192.db
22) nano 0.168.192.db y:

                        reemplazar ---> IN SOA rname.invalid. (
                               por ---> IN SOA ns1.fedoraserver.com. admin.fedoraserver.com. (

También agregamos:

	--->    	IN	NS	ns1.fedoraserver.com.
	---> 131	IN	PTR	ns1.fedoraserver.com.
	---> 130	IN	PTR	pc1.fedoraserver.com.

Con esta configuración ahora podemos buscar el dominio de una máquina utilizando su IP.

Para que el DNS secundario/slave reconozca estas peticiones hay que indicarle el archivo. 
23) nano /etc/named/ (del servidor secundario) y agregar:

 					zone "0.168.192.in-addr.arpa" {
        					type slave;
        					file "slaves/0.168.192.db";
        					masters { 192.168.85.131; };
					};

Aplicar systemctl restart named en ambos servidor para que hagas las pruebas que necesites.
