# Securización de Servidores DNS con DNSSEC

## 1. El problema del DNS original

El protocolo DNS se diseñó asumiendo que la red era de confianza. Al utilizar UDP en el puerto 53 y transmitir las consultas en texto claro, es vulnerable a diferentes ataques. No dispone de mecanismos propios para verificar si una respuesta es auténtica o si un intermediario ha modificado los datos (Man-in-the-Middle).

Además, la falsificación de la IP de origen (IP Spoofing) puede utilizarse en ataques DDoS por amplificación cuando los servidores están mal configurados. Las consultas tradicionales también permiten la inspección del tráfico (eavesdropping) y pueden verse afectadas por el secuestro de consultas DNS (DNS hijacking).

## 2. Cómo securizamos el servicio DNS

Para desplegar un servicio más seguro, he aplicado las siguientes medidas:

- **Aislamiento de roles:** Desactivé la recursión mediante `recursion no` en ambos servidores. De esta forma, los servidores autoritativos solo responden por las zonas que administran y no actúan como resolutores recursivos.
- **Restricción de transferencias:** Utilicé `allow-transfer` para permitir que únicamente la IP del servidor esclavo pueda solicitar una transferencia de zona al maestro. Esto reduce el riesgo de que terceros obtengan información de la zona mediante consultas AXFR o IXFR.
- **Firmas digitales:** Configuré DNSSEC para añadir firmas digitales a los registros DNS. Esto permite comprobar la autenticidad e integridad de los datos mediante una cadena de confianza, siempre que la validación esté correctamente configurada.

Es importante tener en cuenta que DNSSEC no cifra las consultas ni oculta las direcciones IP consultadas. Su objetivo principal es proteger la autenticidad e integridad de los datos DNS, no proporcionar confidencialidad.

## 3. Aspectos que hay que tener en cuenta

- **Consumo de recursos:** DNSSEC añade operaciones criptográficas y aumenta el tamaño de algunas respuestas DNS. Esto puede incrementar el consumo de CPU y el tráfico de red.
- **Riesgo de expiración:** Si las firmas RRSIG caducan o las claves no se gestionan correctamente, las respuestas pueden dejar de validarse. Para reducir este riesgo, configuré la política `dnssec-policy default`, que permite a BIND gestionar automáticamente las claves y su renovación según la política aplicada.
- **Aislamiento:** Se mantiene separada la función autoritativa de la recursiva para reducir la superficie de ataque.
- **Transferencias de zona:** Restringirlas por IP reduce el acceso no autorizado, aunque para una protección adicional se puede utilizar TSIG, que autentica las transferencias entre servidores.
- **Validación DNSSEC:** Ver una firma RRSIG en una respuesta no demuestra por sí solo que la cadena de confianza completa se haya validado correctamente. Para comprobarlo, es necesario realizar pruebas adicionales con un resolutor validador.

## 4. Configuración y pruebas

**Topología de red:**

- Servidor maestro: Ubuntu en `192.168.6.25`.
- Servidor esclavo: Debian 13 en `192.168.6.1`.
- Dominio de pruebas: `empresa.local`.

El servidor maestro administra la zona DNS y genera las firmas digitales. El servidor esclavo recibe una copia de la zona mediante una transferencia autorizada. Finalmente, se realizan consultas desde el cliente para comprobar las respuestas DNS y las firmas.

**Nota:** `empresa.local` se utiliza como dominio de laboratorio. Para un despliegue real con validación DNSSEC pública, se recomienda utilizar un dominio registrado y una cadena de confianza correctamente establecida.

### A. Configuración del maestro (Ubuntu)

Configuro la zona autoritativa y aplico la política DNSSEC automática. También compruebo la sintaxis de los archivos antes de reiniciar el servicio.

```bash
cat /etc/bind/named.conf.local
cat /var/lib/bind/db.empresa.local
sudo named-checkconf
sudo named-checkzone empresa.local /var/lib/bind/db.empresa.local
sudo systemctl restart bind9
sudo systemctl status bind9 --no-pager
```

![Captura 1 - Maestro Ubuntu](img1.png)

Explicación: En esta captura se observa la configuración de la zona, la directiva `dnssec-policy default` y la restricción de transferencias mediante `allow-transfer`. El comando `named-checkconf` comprueba la sintaxis de la configuración general, mientras que `named-checkzone` verifica la estructura de la zona.

Si ambos comandos finalizan correctamente, no se han detectado errores de sintaxis en los elementos comprobados. La comprobación del estado de `bind9` permite verificar que el servicio se ha iniciado correctamente.

La política DNSSEC permite que BIND gestione las claves y las firmas de la zona. La configuración debe ser compatible con la versión instalada de BIND y con el tipo de zona utilizado.

### B. Configuración del esclavo (Debian)

Configuro el servidor esclavo para que obtenga la zona desde el maestro `192.168.6.25`. La transferencia debe estar autorizada en el maestro y la configuración del esclavo debe identificar correctamente la zona y su servidor principal.

```bash
cat /etc/bind/named.conf.local
sudo named-checkconf
sudo systemctl restart bind9
sudo systemctl status bind9 --no-pager
ls -l /var/cache/bind/
```

![Captura 2 - Esclavo Debian](img2.png)

Explicación: En esta captura se muestra la configuración del esclavo y los archivos almacenados en su directorio de zonas. Si aparece el archivo de zona transferido, se dispone de una evidencia de que la copia local se ha generado.

La transferencia de zona permite que el esclavo mantenga una copia de los registros del maestro. Para confirmar que se ha completado correctamente, también se puede revisar el registro del servicio:

```bash
sudo journalctl -u bind9 -n 50 --no-pager
```

La presencia de un archivo con extensión `.signed` puede ser una evidencia del proceso de firma, pero no demuestra por sí sola que la transferencia se haya completado correctamente. Esto depende de cómo se haya configurado la firma y de qué archivos gestione BIND en cada servidor.

### C. Pruebas desde el cliente (Debian)

Desde el cliente consulto al servidor maestro para comprobar que responde con los registros de la zona y solicitar la información DNSSEC.

```bash
dig @192.168.6.25 www.empresa.local +dnssec
```

![Captura 3 - Cliente dig](img3.png)

Explicación: En la respuesta de `dig`, la sección `ANSWER SECTION` muestra los registros devueltos por el servidor. Si aparece un registro `RRSIG`, significa que se ha recibido información de una firma digital asociada a los datos DNS.

El indicador `+dnssec` solicita que el servidor incluya los registros relacionados con DNSSEC cuando corresponda. Sin embargo, no obliga al servidor consultado a validar la respuesta.

Por tanto, la presencia de un registro RRSIG permite comprobar que se está proporcionando información de firma, pero no garantiza por sí sola que la respuesta haya superado una validación completa. Para demostrar la validación criptográfica también sería necesario realizar una consulta a través de un resolutor que valide DNSSEC.

### D. Verificación final en el maestro

Compruebo el estado del proceso de firma de la zona en el servidor maestro.

```bash
sudo rndc signing -list empresa.local
```

![Captura 4 - Verificación rndc](img4.png)

Explicación: Este comando permite consultar el estado de las operaciones de firma pendientes de la zona. Si no aparecen operaciones pendientes, puede indicar que no queda trabajo de firma en curso.

El comando no garantiza por sí solo que todas las claves se hayan generado correctamente ni que la zona esté validándose desde el cliente. Para completar la comprobación, hay que revisar también los registros de BIND, los archivos generados y las respuestas DNS.

Si se necesita comprobar las claves DNSSEC disponibles, se pueden revisar los archivos de claves y los registros de la zona según la configuración utilizada.

## 5. Conclusión

Con esta práctica he configurado un entorno DNS con un servidor maestro y otro esclavo, restringiendo las transferencias de zona y aplicando una política de firma DNSSEC.

Las comprobaciones de configuración permiten detectar errores de sintaxis, mientras que las consultas con `dig` permiten observar los registros DNS y las firmas RRSIG. La revisión del estado de BIND y del proceso de firma proporciona información adicional sobre el funcionamiento del servicio.

DNSSEC mejora la seguridad del DNS al permitir verificar la autenticidad e integridad de los datos, pero debe configurarse y validarse correctamente. No sustituye otras medidas de seguridad, como el control de acceso, la supervisión de registros, la protección contra ataques de denegación de servicio o el cifrado del tráfico cuando sea necesario.

## 6. Documentación y entrega

Todo el procedimiento queda documentado en este informe. Debajo de cada captura se incluye una explicación de los comandos ejecutados, los resultados esperados y la función de cada medida de seguridad.

**Enlace a la entrega:** [Pega aquí tu enlace de GitBook o Notion]
