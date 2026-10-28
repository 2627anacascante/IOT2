Institut TIC de Barcelona
Activitat 02 - Dispositius IoT
Raspberry PI
Enunciat de l’activitat i documentació de la pràctica
Alumna: Ana Cascante Rodriguez
Data: octubre de 2026
Word amb les fotos-> veure arxiu Activitat02_Raspberry_IOT_acascante.docx
 
1. Enunciat de l'activitat
Activitat 02 - Dispositius IoT. Raspberry PI
Objectiu
L'objectiu principal de l'activitat és introduir-nos en un dispositiu més complex, la Raspberry PI, que carrega tot un sistema operatiu Linux complert, però alhora disposa d'entrades i sortides digitals i analògiques, que el fan adequat per a utilitzar-lo com a dipositiu d'IoT.
Utilitzarem un framework de programació en javascript, que està construït sobre Node.js, i s'anomena Node-Red. Aprendrem el seu funcionament bàsic, com es programa, i quines possibilitats té com a "Middleware", és a dir, servir de pont o intercanviador de missatges entre diferents serveis (port serie, I2C, http, tcp/udp, websockets, mqtt, bbdd, etc).
També aprendrem com connectar la sonda de temperatura pel bus OneWire en Linux.
Tasques
Posa a punt una placa Raspberry PI, instal·la Node-Red.
1.	Configura Onewire en Linux per llegir la sonda b18d20 i fes les proves pertinents.
2.	Llegeix el valor de temperatura de la sonda amb Node-Red.
3.	Configura un Dashboard en Node-Red per visualitzar els canvis.
4.	Envia les dades a un endpoint http. (Get o Post). Podem utilitzar el mateix que en l'activitat anterior.
5.	Envia les dades a una base de dades relacional. Taula logs: (data, usuari, sensor i temperatura).
Entrega
Haureu de pujar el codi a una tasca de Git Classroom, crear un README.md amb la documentació de les tasques i mostrar el funcionament del dispositiu a classe.
Recursos
Configuració del bus Onewire en Raspberry:
•	https://www.raspberrypi.com/documentation/computers/configuration.html
•	https://www.waveshare.com/wiki/Raspberry_Pi_Tutorial_Series:_1-Wire_DS18B20_Sensor
Instal·lació node-red en raspberry:
•	https://nodered.org/docs/getting-started/raspberrypi
Node-red:
•	https://node-red-academy.learnworlds.com/
 
2. Documentació de la pràctica
En aquest apartat s’explica, pas a pas, com s’ha dut a terme l’activitat: des de la preparació de la Raspberry Pi fins a l’enviament de les dades de temperatura a un tauler de control, a un endpoint HTTP i a una base de dades relacional. Cada bloc inclou les ordres utilitzades i les captures del procés.
2.1. Material i entorn
Element	Detall
Placa	Raspberry Pi 4 amb sistema operatiu Linux (Debian)
Sensor	Sonda de temperatura DS18B20 (bus 1-Wire), identificador 28-00000054ff31
Red	Router MikroTik-IOT, que assigna una IP a la Raspberry (192.168.88.206)
Programari	Node-RED, node-red-dashboard y MariaDB 11.8.6
Accés	SSH des de l’ordinador (ssh usuari@ip) i navegador web (port 1880)

2.2. Preparació de la Raspberry Pi
1.	Configurar la imatge del sistema operatiu de la Raspberry Pi.
2.	Connectar l’ordinador a la xarxa del router MikroTik-IOT.
3.	El router assigna una adreça IP a la Raspberry.
4.	Accedir a la Raspberry per SSH amb l’adreça IP assignada:
ssh usuario@ip

2.3. Tasca 1: configurar 1-Wire a Linux i provar la sonda
Habilitar la interfície 1-Wire
La interfície 1-Wire s’habilita amb l’eina de configuració de la Raspberry (Interface Options, 1-Wire) i es reinicia la placa:
sudo raspi-config
Connexió de la sonda als pins
La sonda es connecta als pins de la Raspberry segons el material de l’assignatura: la línia de dades al GPIO4 (1-Wire), l’alimentació a 3,3 V i la massa a GND.
Cable de la sonda	Pin de la Raspberry	Observacions
DATA	GPIO4 (pin físico 7)	Bus 1-Wire por defecto
VCC	3,3 V (pin 1)	Alimentació
GND	GND (pin 6)	Massa

Entre la línia de dades i els 3,3 V es col·loca una resistència pull-up de 4,7 kΩ, necessària perquè el bus 1-Wire funcioni de manera estable.
Proves de funcionament
Primer es comprova que el sistema detecta el dispositiu:
ls /sys/bus/w1/devices/
Ha d’aparèixer una carpeta que comença per 28- (la sonda, en aquest cas 28-00000054ff31) i w1_bus_master1 (el bus). Després es llegeix el valor de la sonda:
cat /sys/bus/w1/devices/28-00000054ff31/w1_slave
La sortida té dues línies:
b0 01 7f 80 1f ff 10 10 25 : crc=25 YES
b0 01 7f 80 1f ff 10 10 25 t=27000
•	YES indica que la lectura és vàlida (CRC correcte).
•	t=27000 és la temperatura en mil·lèsimes de grau: es divideix entre 1000, és a dir, 27,0 °C.
2.4. Instal·lació de Node-RED
Node-RED s’instal·la a la Raspberry amb l’script oficial:
bash <(curl -sL https://github.com/node-red/linux-installers/releases/latest/download/install-update-nodered-deb)
L’instal·lador proporciona les ordres següents per treballar amb el servei:
Ordre	Funció
node-red-start	Inicia el servei i mostra el registre. Amb Ctrl-C o tancant la finestra, el servei continua funcionant en segon pla.
node-red-stop	Atura el servei de Node-RED.
node-red-restart	Reinicia el servei.
node-red-reload	Atura i torna a iniciar el servei.
node-red-log	Mostra el registre del servei.

Perquè Node-RED s’iniciï amb el sistema i per engegar-lo, s’utilitzen aquestes ordres:
sudo systemctl enable nodered.service
sudo systemctl start nodered.service
S’accedeix a l’editor des del navegador amb l’adreça IP de la Raspberry i el port 1880: http://192.168.88.206:1880.
2.5. Tasca 2: llegir la temperatura de la sonda amb Node-RED
Dins de Node-RED es crea el flux de dades. El plantejament inicial és el següent:
•	inject: envia un missatge per iniciar la lectura (es repeteix cada 5 segons).
•	sonda: llegeix la temperatura de la DS18B20.
•	function (JavaScript): llegeix la dada i construeix l’URL o el missatge que s’ha d’enviar.
•	http request: envia la petició a l’endpoint.
•	dashboard: mostra la temperatura en una pàgina web.
Instal·lar el node de la sonda
Al menú de Node-RED, a Manage palette, pestanya Install, se cerca “DS18B20”. La paleta ofereix diversos paquets per a aquesta sonda (a la captura n’apareix, entre d’altres, node-red-contrib-ds18b20-sensor).
 
Figura 1. Instal·lació del node de la sonda DS18B20 des de la paleta de Node-RED.
Primer flux
S’afegeixen els nodes inject, sonda, function i http request, i es connecten en cadena.
 
Figura 2. Primer flux: inject cada 5 segons, sonda ds18b20, function “payload” i http request.
Configuració del node inject
El node inject (“eject cada 5 segons”) envia la data i l’hora (timestamp) com a payload i deixa el topic buit. Està configurat per enviar el primer missatge 5 segons després de desplegar el flux.
 
Figura 3. Configuració del node inject.
Comprovació al panell de debug
En desplegar el flux, el node debug mostra el missatge de la sonda. El camp payload conté la temperatura (23,5 °C), file és l’identificador de la sonda (28-00000054ff31) i dir és el bus (w1_bus_master1).
 
Figura 4. Missatge del node sonda al panell de debug: payload 23.5.
2.6. Tasca 3: tauler de control per visualitzar els canvis
S’instal·la el paquet node-red-dashboard des de Manage palette i s’afegeix un node de tipus gràfic (chart) connectat a la sortida de la sonda. Configuració utilitzada:
Paràmetre	Valor
Group	[Temperatures_dash] Default
Label	dashboard
Type	Line chart
X-axis	darrera hora
X-axis Label	HH:mm:ss
Llegenda / Interpolació	Cap / lineal

 
Figura 5. Propietats del node dashboard (gràfic de línies).
El resultat es veu al navegador a http://192.168.88.206:1880/ui: el gràfic s’actualitza amb cada lectura de la sonda.
 
Figura 6. Tauler de control de Node-RED que mostra l’evolució de la temperatura (23,5 °C).
2.7. Tasca 4: enviar les dades a un endpoint HTTP
Les dades s’envien a un endpoint HTTP amb el node http request. L’endpoint d’exemple rep la temperatura com a paràmetre de l’URL (mètode GET):
http://192.168.88.245:1880/dades?temperatura=23.2&param2=0
Per rebre i comprovar aquestes peticions, el flux inclou un node [get] /dades (http in). Una funció JavaScript llegeix la temperatura i construeix l’URL amb el valor mesurat; després, el node http request fa la petició.
A la versió final, la funció “payload” també prepara un missatge per a l’endpoint amb les dades en format JSON (data, usuari, sensor i temperatura), que s’envia amb un node http request (POST). El codi complet es mostra a l’apartat 2.8.
2.8. Tasca 5: enviar les dades a una base de dades relacional
Instal·lar i iniciar MariaDB
S’instal·la el servidor MariaDB a la Raspberry:
sudo apt install mariadb-server -y
 
Figura 7. Instal·lació de MariaDB amb apt.
S’habilita perquè s’iniciï amb el sistema:
sudo systemctl enable mariadb
 
Figura 8. Habilitació del servei mariadb.
S’entra a la consola SQL com a administrador:
sudo mysql
 
Figura 9. Accés a la consola de MariaDB.
Crear la base de dades, l’usuari i la taula logs
Dins de la consola SQL es crea la base de dades temperatures, l’usuari nodered amb els seus permisos i la taula logs amb els camps que demana l’enunciat (data, usuari, sensor i temperatura):
CREATE DATABASE temperatures;
CREATE USER 'nodered'@'localhost' IDENTIFIED BY 'pw123';
GRANT ALL PRIVILEGES ON temperatures.* TO 'nodered'@'localhost';
FLUSH PRIVILEGES;
USE temperatures;
CREATE TABLE logs (
  id INT AUTO_INCREMENT PRIMARY KEY,
  data DATETIME,
  usuari VARCHAR(50),
  sensor VARCHAR(50),
  temperatura FLOAT
);
EXIT;
 
Figura 10. Script SQL per crear la base de dades, l’usuari i la taula logs.
La clàusula IDENTIFIED BY defineix la contrasenya de l’usuari. L’usuari només es pot connectar des de la mateixa Raspberry (@'localhost') i té permisos únicament sobre la base de dades temperatures.
Ajustar els permisos i comprovar la taula
Per assegurar que l’usuari nodered té la contrasenya correcta i els permisos necessaris, s’executen aquestes ordres i després es comprova l’accés des del terminal:
ALTER USER 'nodered'@'localhost' IDENTIFIED BY 'pw123';
CREATE DATABASE IF NOT EXISTS temperatures;
GRANT ALL PRIVILEGES ON temperatures.* TO 'nodered'@'localhost';
FLUSH PRIVILEGES;
USE temperatures;
CREATE TABLE IF NOT EXISTS logs (...);
SHOW TABLES;
mysql -u nodered -ppw123 temperatures -e "SHOW TABLES;"
 
Figura 11. Ajust dels permisos i comprovació que la taula logs existeix.
Node mysql a Node-RED
S’instal·la el node mysql (paquet node-red-node-mysql) i s’afegeix al flux amb el nom “Temperatures”. La connexió es configura amb aquestes dades:
Camp	Valor
Host	127.0.0.1
Port	3306
Usuari	nodered
Contrasenya	pw123
Base de dades	temperatures

Al principi, el node mostrava l’error ECONNREFUSED perquè el servidor MariaDB encara no estava en funcionament. Això s’explica a l’apartat de problemes trobats.
 
Figura 12. Flux amb el node Temperatures que mostra l’error ECONNREFUSED.
Funció “payload”: preparar les dades
La funció “payload” processa la lectura i retorna tres missatges, un per a cada sortida: l’INSERT per a la base de dades (sortida 1), el valor de temperatura per al tauler de control (sortida 2) i les dades en JSON per a l’endpoint HTTP (sortida 3).
const txt = String(msg.payload);
const m = txt.match(/t=(-?\d+)/);
if (!txt.includes("YES") || !m) return null;   // lectura no válida
 
const temp = parseInt(m[1]) / 1000;
 
// data i hora locals en format MySQL
const ahora = new Date(Date.now() - new Date().getTimezoneOffset() * 60000)
    .toISOString().slice(0, 19).replace("T", " ");
 
// Sortida 1: base de dades
const msgDB = {
    topic: "INSERT INTO logs (data, usuari, sensor, temperatura) VALUES (?,?,?,?)",
    payload: [ahora, "anacr", "DS18B20", temp]
};
 
// Sortida 2: tauler de control
const msgDash = { payload: temp, topic: "DS18B20" };
 
// Sortida 3: endpoint HTTP
const msgHTTP = {
    payload: { data: ahora, usuari: "anacr", sensor: "DS18B20", temperatura: temp }
};
 
return [msgDB, msgDash, msgHTTP];
 
Figura 13. Codi de la funció “payload” a Node-RED.
Què fa cada part
•	match(/t=(-?\d+)/) extreu el nombre que hi ha després de t=; es divideix entre 1000 per obtenir graus Celsius.
•	Si la lectura no conté YES o no es troba el valor, es descarta el missatge amb return null.
•	msg.topic del missatge de la base de dades conté l’ordre SQL, amb quatre marcadors ?; msg.payload conté els quatre valors en el mateix ordre que les columnes (data, usuari, sensor, temperatura).
•	El node mysql executa l’INSERT cada vegada que arriba un missatge, de manera que s’afegeix una fila nova a la taula logs amb cada lectura.
Comprovar les dades desades
Els registres es poden consultar des del terminal:
mysql -u nodered -ppw123 temperatures -e "SELECT * FROM logs ORDER BY id DESC LIMIT 5;"
2.9. Flux final
El flux final reuneix totes les parts: inject inicia la lectura cada 5 segons, la sonda 28-00000054ff31 obté la temperatura, el tauler de control la representa gràficament, el node debug “sonda” permet comprovar cada missatge i la funció “payload” envia les dades al node mysql “Temperatures”, que ja apareix com a connected. A més, el node [get] /dades permet rebre peticions HTTP i catch: all amb debug 1 recull els errors.
 
Figura 14. Flux final a Node-RED, amb el node Temperatures connectat a la base de dades.
2.10. Problemes trobats i solucions
Error	Causa	Solució
ECONNREFUSED 127.0.0.1:3306	El servidor MariaDB no estava instal·lat o iniciat.	Instal·lar mariadb-server i iniciar-lo amb sudo systemctl start mariadb (i enable perquè s’iniciï automàticament).
ER_ACCESS_DENIED / Access denied for user	L’usuari o la contrasenya del node no coincidien amb els de MariaDB (el node feia servir un altre usuari i no tenia contrasenya).	Crear l’usuari nodered amb ALTER USER ... IDENTIFIED BY 'pw123', concedir permisos amb GRANT i posar el mateix usuari i contrasenya al node mysql.
ER_EMPTY_QUERY	El node mysql rebia un missatge sense l’ordre SQL a msg.topic.	Preparar msg.topic amb l’INSERT i msg.payload amb la llista de valors a la funció “payload”.
Nodo dashboard con triángulo (invalid properties)	El node no tenia assignat un grup (Group) amb la seva pestanya (Tab).	Crear el grup i la pestanya (Temperatures_dash / Default) i assignar-los al node.

.
