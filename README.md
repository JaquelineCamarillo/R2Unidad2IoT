# R2Unidad2IoT

# Curso JavaScript Essentials 2 de Cisco NetAcad.
Capturas 
<img width="959" alt="netacat5" src="https://github.com/user-attachments/assets/80553424-4489-462e-8dbb-9a8eb4a77d8c" />
<img width="954" alt="netacat1" src="https://github.com/user-attachments/assets/254697c8-3045-40b8-8d98-25be69a95e54" />
<img width="959" alt="netacat2" src="https://github.com/user-attachments/assets/24087ddb-5086-4414-85ea-be6109a23249" />
<img width="954" alt="netacat3" src="https://github.com/user-attachments/assets/103882f2-30da-4ee4-b321-e615bb62470a" />
<img width="952" alt="netacat4" src="https://github.com/user-attachments/assets/160129cd-3da1-4507-ad83-00b3e181db13" />

# Ejercicio 1: Comunicación Serial con Conversión de Datos
En lugar de solo enviar datos, el ESP32 deberá convertir la señal analógica de
un sensor en datos legibles en la PC.
1. Implementa una comunicación serial donde el ESP32 convierta la señal analógica de
un sensor en datos de temperatura o humedad y los envíe a la PC.

# Codigo:
python
from machine import ADC, Pin
from time import sleep

# Configuramos el pin 13 como entrada analógica
sensor = ADC(Pin(34))
sensor.atten(ADC.ATTN_11DB)   # Para medir hasta 3.3V
sensor.width(ADC.WIDTH_10BIT)  # Rango de 0-1023

while True:
    valor = sensor.read()  # Leemos el valor ADC (0-1023)
    voltaje = valor * (3.3 / 1023.0)  # Convertimos a voltaje
    temperatura = voltaje * 100  # 10mV = 1°C para el LM35

    print("Temperatura: {:.2f} °C".format(temperatura))
    sleep(1)


# Diagrama de conexion:
<img width="195" alt="Captura de pantalla 2025-05-06 155011" src="https://github.com/user-attachments/assets/d862c9d3-3ac3-4bf1-8930-80facfcbb69b" />

# Video Demostrativo:

TU VIDEO 

////////////////////////////////////////////////////////////////////////////

# Ejercicio 2: Comunicación Bluetooth con Control desde un Chatbot
En lugar de encender un LED, se usará un chatbot en Telegram o WhatsApp
para controlar el Bluetooth.
1. Configura un bot que, al recibir un mensaje en Telegram o WhatsApp, envíe
comandos a un ESP32 para encender/apagar un LED o mover un servo.
# Codigo:

# servidor.py 
python

import asyncio
import telebot
from bleak import BleakClient, BleakScanner
import threading

TOKEN = 'token del chat bot'
bot = telebot.TeleBot(TOKEN)

UART_SERVICE_UUID = "6E400001-B5A3-F393-E0A9-E50E24DCCA9E"
UART_RX_CHAR_UUID = "6E400002-B5A3-F393-E0A9-E50E24DCCA9E"
UART_TX_CHAR_UUID = "6E400003-B5A3-F393-E0A9-E50E24DCCA9E"

ble_client = None
device_name = "ESP32-BLE"

reconnect_loop_running = True

def notification_handler(sender, data):
    message = data.decode()
    print(f"Mensaje recibido del ESP32: {message}")

async def connect_ble():
    global ble_client
    print(f"Buscando dispositivo BLE: {device_name}...")
    try:
        device = await BleakScanner.find_device_by_name(device_name)
        if not device:
            print(f"No se encontró el dispositivo: {device_name}")
            return False

        client = BleakClient(device.address)
        await client.connect()

        if client.is_connected:
            print(f"Conectado a {device.name}")
            await client.start_notify(UART_TX_CHAR_UUID, notification_handler)
            ble_client = client
            return True
        else:
            print("No se pudo conectar al dispositivo")
            return False
    except Exception as e:
        print(f"Error al conectar: {e}")
        return False

async def disconnect_ble():
    global ble_client
    if ble_client and ble_client.is_connected:
        await ble_client.disconnect()
        print("Desconectado del dispositivo BLE")

async def send_command(command):
    global ble_client
    if not ble_client or not ble_client.is_connected:
        print("No hay conexión BLE activa")
        return False
    try:
        await ble_client.write_gatt_char(UART_RX_CHAR_UUID, command.encode())
        print(f"Comando enviado: {command}")
        return True
    except Exception as e:
        print(f"Error al enviar comando: {e}")
        return False

async def reconnection_loop():
    global ble_client, reconnect_loop_running
    while reconnect_loop_running:
        if not ble_client or not ble_client.is_connected:
            print("Intentando reconectar...")
            await connect_ble()
        await asyncio.sleep(5)

def run_ble_loop():
    loop = asyncio.new_event_loop()
    asyncio.set_event_loop(loop)
    loop.run_until_complete(connect_ble())
    loop.run_until_complete(reconnection_loop())

ble_thread = threading.Thread(target=run_ble_loop)
ble_thread.daemon = True
ble_thread.start()

@bot.message_handler(commands=['start'])
def send_welcome(message):
    bot.reply_to(message, "¡Hola! Envíame comandos como LED:ON o LED:OFF para controlar el LED.")

@bot.message_handler(commands=['status'])
def check_status(message):
    if ble_client and ble_client.is_connected:
        bot.reply_to(message, "✅ Conectado al ESP32")
    else:
        bot.reply_to(message, "❌ No hay conexión con el ESP32")

@bot.message_handler(commands=['reconnect'])
def force_reconnect(message):
    bot.reply_to(message, "Intentando reconectar con el ESP32...")

@bot.message_handler(func=lambda message: True)
def handle_message(message):
    command = message.text.strip().upper()
    if command in ["LED:ON", "LED:OFF"]:
        loop = asyncio.new_event_loop()
        success = loop.run_until_complete(send_command(command))
        if success:
            bot.reply_to(message, f"Comando enviado: {command}")
        else:
            bot.reply_to(message, "No se pudo enviar el comando. Verifica la conexión BLE.")
    else:
        bot.reply_to(message, "Comando no válido. Usa LED:ON o LED:OFF")

try:
    print("Iniciando bot de Telegram...")
    bot.polling()
except Exception as e:
    print(f"Error al iniciar el bot: {e}")
finally:
    reconnect_loop_running = False
    loop = asyncio.new_event_loop()
    loop.run_until_complete(disconnect_ble())
    input("Presiona Enter para salir...")


# esp32
# Control de Servo por Bluetooth BLE para ESP32 en MicroPython
python

import bluetooth
from ble_simple_peripheral import BLESimplePeripheral
import time
from machine import Pin

ble = bluetooth.BLE()
sp = BLESimplePeripheral(ble)

led = Pin(2, Pin.OUT)  # Cambia al pin que estés usando para el LED

def on_rx(data):
    data_str = data.decode().strip().upper()
    print("Datos recibidos:", data_str)

    if data_str == "LED:ON":
        led.value(1)
        print("LED encendido")
        if sp.is_connected():
            sp.send("LED encendido")
    elif data_str == "LED:OFF":
        led.value(0)
        print("LED apagado")
        if sp.is_connected():
            sp.send("LED apagado")
    else:
        if sp.is_connected():
            sp.send("Comando no reconocido")

print("Esperando conexión Bluetooth...")
while True:
    if not sp.is_connected():
        sp.advertise()
        print("Esperando conexión...")
    if sp.is_connected():
        sp.on_write(on_rx)
        sp.send(f"ESP32 LED Control - {time.ticks_ms()}")
    time.sleep(3)


# Diagrama de conexion:
![image](https://github.com/user-attachments/assets/b3899046-a898-4fcd-82e3-e4e09de723ca)

# Video Demostrativo:

TU VIDEO 

////////////////////////////////////////////////////////////////////////////

# Ejercicio 3: Comunicación TCP/IP con Base de Datos:
En lugar de solo enviar datos, estos deben almacenarse en una base de datos
(SQLite, Firebase, PostgreSQL).
1. Implementa una comunicación TCP/IP donde un ESP32 registre datos en una base
de datos en la nube o local.

# Codigo:

# Esp32
python
from machine import ADC, Pin
import network
import socket
import time

# Conexión Wi-Fi
ssid = 'tu wifi'
password = 'contrase;a'

sta = network.WLAN(network.STA_IF)
sta.active(True)
sta.connect(ssid, password)

while not sta.isconnected():
    print('Conectando...')
    time.sleep(1)

print('Conectado:', sta.ifconfig())

# Sensor LM35 en GPIO34 (puedes usar otro pin ADC)
sensor = ADC(Pin(34))  # Cambia a 32, 33, 35 si prefieres
sensor.atten(ADC.ATTN_11DB)  # Para leer hasta 3.6V

# Envío TCP
while True:
    try:
        s = socket.socket()
        s.connect(('la ip de tu compu ', 3000))  # IP del servidor
        valor = sensor.read()  # 0 - 4095

        # Convertir la lectura ADC (0-4095) a voltaje (0-3.3V)
        voltaje = (valor / 4095) * 3.3

        # Convertir voltaje a temperatura en Celsius (LM35: 1°C = 10mV)
        temperatura = voltaje * 100

        mensaje = f"temperatura={temperatura:.2f}"  # Formato: temperatura=25.45
        s.send(mensaje.encode())
        print("Enviado:", mensaje)
        s.close()
    except Exception as e:
        print("Error al enviar:", e)
    time.sleep(5)



# server.js
js
const net = require('net');
const sqlite3 = require('sqlite3').verbose();
const db = new sqlite3.Database('./datos.db');

// Crear tabla si no existe
db.run(`CREATE TABLE IF NOT EXISTS lecturas (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  valor TEXT,
  timestamp DATETIME DEFAULT CURRENT_TIMESTAMP
)`);

// Servidor TCP
const server = net.createServer((socket) => {
    console.log('Cliente conectado');

    socket.on('data', (data) => {
        // Guardar solo el valor de temperatura
        const recibido = data.toString().trim(); // Ejemplo de formato: "temperatura=25.45"
        const valor = recibido.split('=')[1]; // Extrae el valor numérico
        db.run('INSERT INTO lecturas (valor) VALUES (?)', [valor], (err) => {
            if (err) {
                return console.error(err.message);
            }
            console.log('Dato guardado en SQLite');
        });
    });

    socket.on('end', () => {
        console.log('Cliente desconectado');
    });
});

// Iniciar el servidor TCP
server.listen(3000, () => {
    console.log('Servidor TCP escuchando en puerto 3000');
});


# web.js

js
const express = require('express');
const sqlite3 = require('sqlite3').verbose();
const app = express();
const port = 3001;  // Cambié el puerto a 3001
const path = require('path');  // Importar el módulo path

// Crear o abrir la base de datos
const db = new sqlite3.Database('./datos.db');
// Configurar para servir archivos estáticos (como HTML, CSS, JS)
app.use(express.static(path.join(__dirname, 'public')));

// Endpoint para obtener las lecturas desde la base de datos
app.get('/api/lecturas', (req, res) => {
  db.all('SELECT * FROM lecturas ORDER BY timestamp DESC', [], (err, rows) => {
    if (err) {
      return res.status(500).json({ error: err.message });
    }
    res.json(rows);  // Enviar los resultados como JSON
  });
});

// Iniciar el servidor Express
app.listen(port, () => {
  console.log(`Servidor HTTP corriendo en http://localhost:${port}`);
});



# Public/index.html
html
<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8">
  <title>Lecturas del Sensor</title>
  <style>
    body { font-family: Arial, sans-serif; padding: 20px; }
    table { width: 100%; border-collapse: collapse; }
    th, td { padding: 8px; border: 1px solid #ccc; text-align: left; }
  </style>
</head>
<body>
  <h1>Lecturas del Sensor</h1>
  <table id="tabla">
    <thead>
      <tr>
        <th>ID</th>
        <th>Temperatura (°C)</th>
        <th>Fecha</th>
      </tr>
    </thead>
    <tbody></tbody>
  </table>

  <script>
    // Realiza una solicitud a la API para obtener las lecturas
    fetch('http://localhost:3001/api/lecturas')
      .then(res => res.json())
      .then(data => {
        const tbody = document.querySelector('#tabla tbody');
        data.forEach(fila => {
          // Si el valor recibido es una temperatura en grados Celsius, se puede usar directamente
          const tr = document.createElement('tr');
          tr.innerHTML = `
            <td>${fila.id}</td>
            <td>${fila.valor} °C</td>  <!-- Mostrando la temperatura -->
            <td>${fila.timestamp}</td>
          `;
          tbody.appendChild(tr);
        });
      })
      .catch(err => {
        console.error('Error al obtener los datos:', err);
      });
  </script>
</body>
</html>





# Diagrama de conexion:
![image](https://github.com/user-attachments/assets/26885679-c106-4110-90e8-0c2db3cc3c69)

# Video Demostrativo:

TU VIDEO 

////////////////////////////////////////////////////////////////////////////
# Ejercicio 4: Envío de Notificaciones con Discord Webhooks:
Se reemplaza el correo por una integración con Discord.
1. Configura un webhook de Discord para recibir notificaciones desde un ESP32
cuando un sensor detecte un evento.

# Codigo:
Python
import network
import urequests
import time
import json
from machine import Pin
import dht  # Importamos sensor DHT

# Configuración
SSID = "tu wifi"
PASSWORD = "la contrase;a"
WEBHOOK_URL = "direccion que te da la api "

sensor_dht = dht.DHT22(Pin(14))

def conectar_wifi():
    wlan = network.WLAN(network.STA_IF)
    wlan.active(True)
    if not wlan.isconnected():
        print("Conectando a WiFi...")
        wlan.connect(SSID, PASSWORD)
        while not wlan.isconnected():
            time.sleep(1)
    print("Conectado a WiFi:", wlan.ifconfig())

def enviar_discord(mensaje):
    headers = {'Content-Type': 'application/json'}
    cuerpo = json.dumps({"content": mensaje})
    try:
        respuesta = urequests.post(WEBHOOK_URL, data=cuerpo.encode('utf-8'), headers=headers)
        print("Mensaje enviado. Código:", respuesta.status_code)
        print("Respuesta:", respuesta.text)
        respuesta.close()
    except Exception as e:
        print("Error al enviar:", e)

# Programa principal
conectar_wifi()

while True:
    try:
        sensor_dht.measure()
        temp = sensor_dht.temperature()
        hum = sensor_dht.humidity()
        mensaje = f" Temp: {temp:.1f}°C |  Humedad: {hum:.1f}%"
        print("Enviando datos:", mensaje)
        enviar_discord(mensaje)
    except Exception as e:
        print("Error leyendo sensor:", e)
    time.sleep(10)  

# Diagrama de conexion:
<img width="209" alt="Captura de pantalla 2025-05-06 225552" src="https://github.com/user-attachments/assets/3dff6079-871f-47f1-b0a5-01e79c6f4c69" />


# Video Demostrativo:

TU VIDEO 

////////////////////////////////////////////////////////////////////////////

# Autor 
Camarillo Olaez Juana Jaqueline

# Autoevaluación y Coevaluación

Coevaluación
Durante el desarrollo del proyecto, observé en mí una actitud constante de esfuerzo y compromiso. A lo largo de las distintas etapas, trabajé intensamente para cumplir con los objetivos técnicos y personales del proyecto. Aunque atravesé momentos de procrastinación, algo que reconozco como una de mis principales debilidades, supe retomar el ritmo gracias a mi insistencia y perseverancia cuando las cosas no salían como esperaba.

Me caractericé por no rendirme fácilmente ante los problemas: cada vez que algo no funcionaba, busqué soluciones con insistencia, investigando y probando diferentes alternativas hasta dar con una respuesta satisfactoria. Esa actitud me permitió avanzar incluso en momentos de dificultad técnica o falta de claridad en las instrucciones.

Mi participación fue activa y comprometida, tanto en el trabajo individual como en la colaboración con el equipo. Aunque reconozco que pude haber manejado mejor mi tiempo en algunos momentos, logré completar las tareas asignadas y participar en las discusiones técnicas, aportando ideas y soluciones basadas en lo que iba aprendiendo. Esta experiencia me ayudó a identificar mis áreas de mejora, especialmente en la gestión personal del tiempo, pero también a reforzar mis fortalezas como la resiliencia y la capacidad de aprendizaje autónomo.

En general, considero que mi desempeño fue positivo: aprendí mucho, enfrenté mis limitaciones y las usé como impulso para crecer. Este proyecto fue una oportunidad para reafirmar mi compromiso con el desarrollo técnico y personal, y para seguir construyendo una base sólida en el ámbito de la tecnología.
