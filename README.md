# Limitar a los usuarios del sistema la conexión
Autor: eskan
Fecha: 3 Feb 2004

Introducción:
Nota: Basado en FreeBSD + ipfw, para otras posibilidades consultar los manuales

En este documento se explicará como limitar el ancho de banda y el tráfico de un usuario, lo cual no implica que una configuración similar a ésta, no puedas limitar o administrarlos en otras situaciones.

1º Paso.- Instalación de los elementos necesarios
ipfw
Lo primero que deberemos hacer es tener el firewall, ipfw, en perfecto funcionamiento, para ello configuramos y recompilamos el kernel:

    eskan# cd /usr/src/sys/i386/conf
    eskan# cp GENERIC LIMITADO (que sera el nombre de nuestro kernel)
    eskan# ee LIMITADO
Y le metemos las siguientes lineas:

    options DUMMYNET
    options IPDIVERT
    options IPFIREWALL
    options IPFIREWALL_VERBOSE
    options	IPFIREWALL_VERBOSE_LIMIT=# (# = numero de veces que ipfw loguee a 
    syslogd los paquetes de una determinada regla) 
recompilando:

    eskan# config LIMITADO
    eskan# cd ../compile/LIMITADO
    eskan# make depend && make && make install
Ya está el kernel apunto.

    rc.conf
    firewall_enable="YES"
    firewall_type="/etc/firewall.rules"
    firewall_script="/etc/rc.firewall"
Donde decimos que se active el firewall y le especificamos la ruta donde configuraremos nuestro firewall

ipa
ipa es el software que usaremos para limitar el tráfico, de los usuarios en nuestro caso.

eskan# cd /usr/ports/sysutils/ipa/ && make install clean
Ya lo tenemos instalado, el archivo de configuración está localizado en "/usr/local/etc/" con el nombre "ipa.conf"

Para iniciar ipa al iniciar el sistema:

eskan# mv /usr/local/etc/rc.d/ipa.sh.sample /usr/local/etc/rc.d/ipa.sh 
2º Paso.- Configurar ipfw con las reglas adecuadas
Para esta tarea simplemente tendrás que pensar como quieres asegurar el sistema según tu política de seguridad, no tienes más que consultar las páginas del manual ipfw(8)

En nuestro caso, limitaremos a los usuarios pedro y luis con 60kBps/100kBps y 25kBps/10kBps [bajada/subida] respectivamente para todas las conexiones

    add 2020 pipe 1 ip from any to any in uid pedro
    add 2030 pipe 2 ip from any to any out uid pedro
    add 3020 pipe 3 ip from any to any in uid luis
    add 3030 pipe 4 ip from any to any out uid luis
    pipe 1 config bw 60KByte/s
    pipe 2 config bw 100KByte/s
    pipe 3 config bw 25KByte/s
    pipe 4 config bw 10KByte/s
con estas simples reglas ya tendremos a los usuarios pedro y luis con el ancho de banda limitado, sin poder en ningú caso superarlo.

3º Paso.- Configurar ipa para limitar el tráfico
Con ipa podremos limitar el tráfico, en nuestro caso de un usuario, y consultar los límites establecidos y el progreso del usuario.

Antes de nada, tendremos que establecer unas reglas en el firewall para poder contar el trafico usado:

    add 2021 count tcp from any to any in uid pedro
    add 2031 count tcp from any to any out uid pedro
    add 3021 count tcp from any to any in uid luis
    add 3031 count tcp from any to any out uid luis
Nos dirigimos al archivo de configuración "/usr/local/etc/ipa.conf" donde estableceremos que el usuario pepe tenga un tráfico de y ponemos las sucesiva configuración para que pedro tenga un máximo de 500MBytes/2000MBytes [Bajada/Subida] cada 15 días y a luis uno de 10GBytes/5GBytes [Bajada/Subida] al mes.

    global {
        update_db_time = 1m 15s
    }
donde especificamos que se actualice la base de datos cada minuto y 15 segundos

    rule pedro_in {
        ipfw = 2021
        info = Trafico de entrada de pedro
        startup {
            if_limit_is_reached {
                exec = /sbin/ipfw add 2015 deny tcp from any to any in uid pedro
            }
        }
        limit 500m {
            byte_limit = 500m
            info = 500 Mbytes cada 15dias
            zero_time = +15d
            reach {
                exec = /sbin/ipfw add 2015 deny tcp from any to any in uid pedro
            }
            expire {
                expire_time = +15d
                exec = /sbin/ipfw del 2015
            }
        }
    }
aqui hemos puesto un límite de 500MBytes de entrada cada 15 días que si es alcanzado dejara sin conexiones TCP de entrada al usuario pedro y donde a los 15 días se reinicia de nuevo la cuenta se haya o no superado el límite

    rule pedro_out {
        ipfw = 2031
        info = Trafico de pedro de salida
        startup {
            if_limit_is_reached {
                exec = /sbin/ipfw add 2025 deny tcp from any to any out uid pedro
            }
        }
        limit 2000m {
            byte_info = 2000m
            info = 2000 MBytes cada 15 dias
            zero_time = +15d
            reach {
                exec = /sbin/ipfw add 2025 deny tcp from any to any out uid pedro
            }
            expire {
                expire_time = +15d
                exec = /sbin/ipfw del 2025
            }
        }
    }
en esta hemos establecido la regla similar a la anterior, pero especificando la salida y el límite de 2000MBytes de salida.

ahora pondre las reglas para el usuario luis, no las comentare por la similitud a las de pedro.

        rule luis_in {
                ipfw = 3021
                info = Trafico de entrada de luis
                startup {
                        if_limit_is_reached {
                            exec = /sbin/ipfw add 3015 deny tcp from any to any in uid luis
                    }
            }
            limit 10g {
                    byte_limit = 10g
                    info = 10 Gbytes cada mes
                    zero_time = +M
                    reach {
                            exec = /sbin/ipfw add 3015 deny tcp from any to any in uid luis
                    }
                    expire {
                            expire_time = +M
                            exec = /sbin/ipfw del 3015
                    }
            }
    }

    rule luis_out {
            ipfw = 3031
            info = Trafico de salida de luis
            startup {
                    if_limit_is_reached {
                            exec = /sbin/ipfw add 3025 deny tcp from any to any out uid luis
                    }
            }
            limit 5g {
                    byte_limit = 5g
                    info = 5 Gbytes cada mes
                    zero_time = +M
                    reach {
                            exec = /sbin/ipfw add 3025 deny tcp from any to any out uid luis
                    }
                    expire {
                            expire_time = +M
                            exec = /sbin/ipfw del 3025
                    }
            }
    }
Bueno con todo esto ya tendriamos a nuestros 2 usuarios limitados, para má posibilidades, miraros el manual ipa.conf(5) y para el uso del ipa leeros el ipa(8)

Finalizando
En cualquier momento podeis ver el estado del tráfico de cada usuario con el comando ipastat

eskan# ipastat -R luis_in

        +---------------------+---------------------+
        | From                | To                  |
        +---------------------+---------------------+
        | 2004.02.01/00:00:00 | 2004.02.29/24:00:00 |
        +---------------------+---------------------+

    +----------+----------------------------+-------+--------+
    | Rule     | Info                       | Bytes | Mbytes |
    +----------+----------------------------+-------+--------+
    | luis_in  | Trafico de entrada de luis |    25 |      0 |
    +----------+----------------------------+-------+--------+
eskan#
Como en este ejemplo, podeis ver las características de cada regla y de sus límites, miraros la página del manual ipastat(8) que tiene unas opciones muy completas.

Recordad, leeros los manuales que son muy útiles, aqui teneis una web relacionada con el ipa http://ipa-system.sourceforge.net/