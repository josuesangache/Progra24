#include <WiFi.h>
#include <WebServer.h>
#include <DHT.h>
#include <ESP32Servo.h>

// Configuración del sensor DHT11
#define DHTPIN 18     // Pin digital donde está conectado el DHT11
#define DHTTYPE DHT11 // Definir tipo de sensor DHT

DHT dht(DHTPIN, DHTTYPE);

// Crear instancia del servomotor
Servo myservo;

// Configuración de red Wi-Fi
const char* ssid = "nombredered";
const char* password = "contraseñadered";

// Crear instancia del servidor web
WebServer server(80);

void setup() {
  Serial.begin(115200);
  dht.begin();
  myservo.attach(13); // Pin digital donde está conectado el servomotor

  // Conectarse a la red Wi-Fi
  WiFi.begin(ssid, password);
  Serial.println("Conectando a WiFi...");
  while (WiFi.status() != WL_CONNECTED) {
    delay(1000);
    Serial.println("Intentando conectar...");
  }
  Serial.println("Conectado a WiFi");

  // Imprimir la dirección IP
  Serial.print("Dirección IP: ");
  Serial.println(WiFi.localIP());

  // Iniciar el servidor
  server.on("/", handleRoot);
  server.on("/humidity", handleHumidity); // Nueva ruta para la humedad
  server.begin();
  Serial.println("Servidor iniciado");
}

void loop() {
  server.handleClient();

  // Leer la humedad
  float humidity = dht.readHumidity();

  // Comprobar si la lectura del sensor ha fallado
  if (isnan(humidity)) {
    Serial.println("¡Error al leer el sensor DHT!");
    return;
  }

  Serial.print("Humedad: ");
  Serial.print(humidity);
  Serial.println(" %");

  // Mover el servomotor basado en la humedad
  int angle = map(humidity, 0, 100, 0, 180); // Mapeo de la humedad al ángulo del servomotor (0-180 grados)
  myservo.write(angle); // Mover el servomotor al ángulo calculado

  delay(2000); // Esperar 2 segundos antes de la siguiente lectura
}

void handleRoot() {
  String html = "<html><body><h1>Control de Humedad y Servomotor</h1>";
  html += "<p>Humedad: <span id='humidity'>Cargando...</span> %</p>";
  html += "<script>";
  html += "setInterval(function() {";
  html += "  fetch('/humidity').then(response => response.text()).then(data => {";
  html += "    document.getElementById('humidity').innerText = data;";
  html += "  });";
  html += "}, 1000);";  // Actualizar cada 1 segundo
  html += "</script>";
  html += "</body></html>";

  server.send(200, "text/html", html);
}

void handleHumidity() {
  float humidity = dht.readHumidity();

  // Comprobar si la lectura del sensor ha fallado
  if (isnan(humidity)) {
    Serial.println("¡Error al leer el sensor DHT!");
    server.send(500, "text/plain", "Error al leer el sensor DHT");
    return;
  }

  // Enviar datos al cliente
  server.send(200, "text/plain", String(humidity));
}
