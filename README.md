# Practica-11-BASE-DE-DATOS

Dentro de este repositorio se muestra la manera de programar una ESP32 con el sensor DHT11, el sensor Ultrasonico y además, que los datos obtenidos se vean reflejados en el programa Node-RED y se vacien a una base de datos con el progama XAMPP a través de una conexión WIFI.

## Introducción

### Descripción

La ```Esp32``` la utilizamos en un entorno de adquision de datos, lo cual en esta practica ocuparemos un sensor ```DTH11``` para adquirir temperatura y humedad del entorno, el sensor ```HC-SR04``` para el registro de distancia. Tambien utilizaremos el programa ```Node-red``` para trabajar de manera visual y esquematica la programación grafica del flujo de la información, la cual podremos observar los resultados en la interfaz de ```Node-Red``` al igual que podremos gestionar los resultados en una base de datos a traves de un servidor *WEB* con el *software* libre de ```XAMPP```; cabe aclarar que en esta practica se usara un simulador llamado [WOKWI](https://https://wokwi.com/), un programa llamado [Node-RED](http://localhost:1880/) y un *software* llamado [XAMPP](http://localhost/phpmyadmin/index.php?route=/sql&db=diplomado&table=diplomado+2024&pos=0)

## Material Necesario

Para realizar esta practica necesitas lo siguiente:
- [WOKWI](https://https://wokwi.com/)
- Programa [Node-RED](http://localhost:1880/)
- Software [XAMPP](http://localhost/phpmyadmin/index.php?route=/sql&db=diplomado&table=diplomado+2024&pos=0)
- Tarjeta ESP 32
- Sensor DHT11
- Sensor HC-SR04

## Instrucciones

### Requisitos previos

Para poder usar este repositorio necesitas entrar a la plataforma [WOKWI](https://https://wokwi.com/), tener instalado correctamente el programa [Node-RED](http://localhost:1880/) y haber instalado de manera correcta el software [XAMPP](http://localhost/phpmyadmin/index.php?route=/sql&db=diplomado&table=diplomado+2024&pos=0).

### Instrucciones de preparación de entorno 

1. Abrir la terminal de programación y colocar la siguente programación:

```
#include <ArduinoJson.h>
#include <WiFi.h>
#include <PubSubClient.h>
#define BUILTIN_LED 2
#include "DHTesp.h"
const int DHT_PIN = 15;
const int Trigger = 4;   //Pin digital 2 para el Trigger del sensor
const int Echo = 2;   //Pin digital 3 para el Echo del sensor
DHTesp dhtSensor;
// Update these with values suitable for your network.

const char* ssid = "Wokwi-GUEST";
const char* password = "";
const char* mqtt_server = "18.193.219.109";
String username_mqtt="educatronicosiot";
String password_mqtt="12345678";

WiFiClient espClient;
PubSubClient client(espClient);
unsigned long lastMsg = 0;
#define MSG_BUFFER_SIZE  (50)
char msg[MSG_BUFFER_SIZE];
int value = 0;

void setup_wifi() {

  delay(10);
  // We start by connecting to a WiFi network
  Serial.println();
  Serial.print("Connecting to ");
  Serial.println(ssid);

  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);

  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }

  randomSeed(micros());

  Serial.println("");
  Serial.println("WiFi connected");
  Serial.println("IP address: ");
  Serial.println(WiFi.localIP());
}

void callback(char* topic, byte* payload, unsigned int length) {
  Serial.print("Message arrived [");
  Serial.print(topic);
  Serial.print("] ");
  for (int i = 0; i < length; i++) {
    Serial.print((char)payload[i]);
  }
  Serial.println();

  // Switch on the LED if an 1 was received as first character
  if ((char)payload[0] == '1') {
    digitalWrite(BUILTIN_LED, LOW);   
    // Turn the LED on (Note that LOW is the voltage level
    // but actually the LED is on; this is because
    // it is active low on the ESP-01)
  } else {
    digitalWrite(BUILTIN_LED, HIGH);  
    // Turn the LED off by making the voltage HIGH
  }

}

void reconnect() {
  // Loop until we're reconnected
  while (!client.connected()) {
    Serial.print("Attempting MQTT connection...");
    // Create a random client ID
    String clientId = "ESP8266Client-";
    clientId += String(random(0xffff), HEX);
    // Attempt to connect
    if (client.connect(clientId.c_str(), username_mqtt.c_str() , password_mqtt.c_str())) {
      Serial.println("connected");
      // Once connected, publish an announcement...
      client.publish("outTopic", "hello world");
      // ... and resubscribe
      client.subscribe("inTopic");
    } else {
      Serial.print("failed, rc=");
      Serial.print(client.state());
      Serial.println(" try again in 5 seconds");
      // Wait 5 seconds before retrying
      delay(5000);
    }
  }
}

void setup() {
  pinMode(BUILTIN_LED, OUTPUT);     // Initialize the BUILTIN_LED pin as an output
  Serial.begin(115200);
  setup_wifi();
  client.setServer(mqtt_server, 1883);
  client.setCallback(callback);
  dhtSensor.setup(DHT_PIN, DHTesp::DHT22);
  pinMode(Trigger, OUTPUT); //pin como salida
  pinMode(Echo, INPUT);  //pin como entrada
  digitalWrite(Trigger, LOW);//Inicializamos el pin con 0
}

void loop() {


delay(1000);
TempAndHumidity  data = dhtSensor.getTempAndHumidity();
long t; //timepo que demora en llegar el eco
long d; //distancia en centimetros

digitalWrite(Trigger, HIGH);
delayMicroseconds(10);          //Enviamos un pulso de 10us
digitalWrite(Trigger, LOW);
  
t = pulseIn(Echo, HIGH); //obtenemos el ancho del pulso
d = t/59;             //escalamos el tiempo a una distancia en cm
  
Serial.print("Distancia: ");
Serial.print(d);      //Enviamos serialmente el valor de la distancia
Serial.print("cm");
Serial.println();
  if (!client.connected()) {
    reconnect();
  }
  client.loop();

  unsigned long now = millis();
  if (now - lastMsg > 2000) {
    lastMsg = now;
    //++value;
    //snprintf (msg, MSG_BUFFER_SIZE, "hello world #%ld", value);

    StaticJsonDocument<128> doc;

    doc["DEVICE"] = "ESP32";
    //doc["Anho"] = 2022;
    doc["DISTANCIA"] = String(d);
    doc["TEMPERATURA"] = String(data.temperature, 1);
    doc["HUMEDAD"] = String(data.humidity, 1);
   

    String output;
    
    serializeJson(doc, output);

    Serial.print("Publish message: ");
    Serial.println(output);
    Serial.println(output.c_str());
    client.publish("CRISTEMP", output.c_str());
  }
}
```

2. Instalar las librerias de **DHT sensor library for ESPx**, **PubSubClient** y **ArduinoJson**  como se muestra en la siguente imagen.

![](https://github.com/Cris9901/Practica-11-BASE-DE-DATOS/blob/main/IMAGEN%201.jpg)

3. Hacer la conexion de **DHT11** con la **ESP32** como se muestra en la siguente imagen.

![](https://github.com/Cris9901/Practica-11-BASE-DE-DATOS/blob/main/IMAGEN%202.jpg)

4. Hacer la conexion de **HC-SR04** con la **ESP32** como se muestra en la siguente imagen.

![](https://github.com/Cris9901/Practica-11-BASE-DE-DATOS/blob/main/IMAGEN%203.jpg)

5. Cambiando al programa de **Node-RED** primero tendran que instalar los nodos ```node-red-dashboard``` y ```node-red-node-mysql```, que se encuentran en la opcion de **Manage palette**.

![](https://github.com/Cris9901/Practica-11-BASE-DE-DATOS/blob/main/imagen%20instalacion%20de%20nodos.jpg)

6. Colocar el bloque de ```mqtt in```.

![](https://github.com/Cris9901/Practica-11-BASE-DE-DATOS/blob/main/IMAGEN%204.jpg)

7. Configurar el bloque con el puerto mqtt con el ip ```18.193.219.109``` como se muestra en la imagen.

![](https://github.com/Cris9901/Practica-11-BASE-DE-DATOS/blob/main/IMAGEN%205.jpg)

8. Colocar el bloque ```json``` y configurarlo como se muestra en la imagen.

![](https://github.com/Cris9901/Practica-11-BASE-DE-DATOS/blob/main/IMAGEN%206.jpg)

9. Configurar el bloque con la acción de ```Always convert to JavaScript Object```  como se muestra en la imagen.

![](https://github.com/Cris9901/Practica-11-BASE-DE-DATOS/blob/main/IMAGEN%207.jpg)

10. Colocamos cuatro bloques ```function```.

![](https://github.com/Cris9901/Practica-11-BASE-DE-DATOS/blob/main/IMAGEN%208.jpg)

11. Los configuramos con el siguiente codigo, uno para cada funcion.


```
msg.payload = msg.payload.TEMPERATURA;
msg.topic = "TEMPERATURA";
return msg;
```

![](https://github.com/Cris9901/Practica-11-BASE-DE-DATOS/blob/main/IMAGEN%209.jpg)


```
msg.payload = msg.payload.HUMEDAD;
msg.topic = "HUMEDAD";
return msg;
```

![](https://github.com/Cris9901/Practica-11-BASE-DE-DATOS/blob/main/IMAGEN%2010.jpg)


```
msg.payload = msg.payload.DISTANCIA;
msg.topic = "DISTANCIA";
return msg;
```

![](https://github.com/Cris9901/Practica-11-BASE-DE-DATOS/blob/main/IMAGEN%2011.jpg)


```
var query = "INSERT INTO `diplomado 2024`(`ID`, `FECHA`, `DEVICE`, `TEMPERATURA`, `HUMEDAD`) VALUES (NULL, current_timestamp(), '";
query = query+msg.payload.DEVICE + "','";
query = query+msg.payload.TEMPERATURA + "','";
query = query+msg.payload.HUMEDAD + "');'";
msg.topic=query;
return msg;
```

![](https://github.com/Cris9901/Practica-11-BASE-DE-DATOS/blob/main/IMAGEN%20CODIGO%20DE%20FUNCION.jpg)

12. Colocamos los bloques ```Chart``` y ```Guage``` a las funciones de **TEMPERATURA**, **HUMEDAD** y **DISTANCIA**.

![](https://github.com/Cris9901/Practica-11-BASE-DE-DATOS/blob/main/IMAGEN%2012.jpg)

13. Los que estan conectados a la funcion de temperatura los configuramos de la siguiente manera.

*Gauge*

![](https://github.com/Cris9901/Practica-11-BASE-DE-DATOS/blob/main/IMAGEN%2013.jpg)

*Chart*

![](https://github.com/Cris9901/Practica-11-BASE-DE-DATOS/blob/main/IMAGEN%2014.jpg)

14. Los que estan conectados a la funcion de humedad los configuramos de esta manera.

*Gauge*

![](https://github.com/Cris9901/Practica-11-BASE-DE-DATOS/blob/main/IMAGEN%2015.jpg)

*Chart*

![](https://github.com/Cris9901/Practica-11-BASE-DE-DATOS/blob/main/IMAGEN%2016.jpg)

15. Los que estan conectados a la funcion de distancia los configuramos de esta manera.

*Gauge*

![](https://github.com/Cris9901/Practica-11-BASE-DE-DATOS/blob/main/IMAGEN%2017.jpg)

*Chart*

![](https://github.com/Cris9901/Practica-11-BASE-DE-DATOS/blob/main/IMAGEN%2018.jpg)

16. Colocamos el bloque ```mysql```.

![](https://github.com/Cris9901/Practica-11-BASE-DE-DATOS/blob/main/IMAGEN%20BLOQUE%20mysql.jpg)

17. Lo configuramos de la siguiente manera.

![](https://github.com/Cris9901/Practica-11-BASE-DE-DATOS/blob/main/IMAGEN%20CONF%20MYSQL.jpg)

18. En la pestaña de de *Layout* crearemos otro tabulador llamado **Sensor DHT22 con HC-SR04**, dentro de el añadiremos dos grupos uno para los indicadores y otro para las graficas; de igual manera colocaremos tres espaciadores de temperatura,humedad y distancia, los pondremos segun sea el caso y la especificación.

![](https://github.com/Cris9901/Practica-11-BASE-DE-DATOS/blob/main/IMAGEN%2019.jpg)

19. Abrir el *software* **XAMPP**, en el panel de control debemos activar las opciones **Apache** y **MySQL** dando *click izquierdo* sobre el boton *start* (Imagen 1) y despues dar doble *click izquierdo* el boton **Admin** (Imagen 2).

Imagen 1

![](https://github.com/Cris9901/Practica-11-BASE-DE-DATOS/blob/main/Panel%20de%20control%20XAMPP%20uno.jpg)

Imagen 2

![](https://github.com/Cris9901/Practica-11-BASE-DE-DATOS/blob/main/panel%20de%20control%20XAMPP%20dos.jpg)

20. Ya que nos haya redirigido a la pagina de **phpMyAdmin**, crear una nueva tabla con los siguientes criterios.

![]()

21. Para que nuestra programación en **WOKWI**, el flujo de datos en **Node-Red** y la base de datos en **XAMPP** sirvan de manera correcta; tenemos que nombrar nuestra base de datos y el proyecto de la siguiente manera.

![]()

### Instrucciónes de operación

1. Iniciar simulador en [WOKWI](https://https://wokwi.com/).
2. Visualizar los datos en el monitor serial.
3. Colocar la temperatura y humedad dando *doble click* al sensor **DHT11**.
4. Colocar la distancia dando *doble click* al sensor **HC-SR04**.
5. Iniciar el simulador en [Node-RED](http://localhost:1880/) dando *click izquierdo* en el botón **Deploy** y despues abrir la interfaz dando *click izquierdo* en el boton de exportar.
6. Visualizar la interfaz.
7. Dar *click izquierdo* en la pestaña **Examinar** de la base de datos **MySQL**.
8. Visualisar los datos en la tabla creada.

## Resultados

Cuando haya funcionado, verás los valores dentro del monitor serial, la interfaz y la base de datos como se muestra en las siguentes imagenes.

Resultados en **Node-red**

![]()

![]()

Resultados en **WOKWI**

![]()

Resultados en **XAMPP**

![]()

# Créditos
Desarrollado por Ing. Montañez Mejia Cristian.






