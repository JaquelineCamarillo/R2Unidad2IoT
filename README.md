# R2Unidad2IoT
## Juana Jaqueline Camarillo Olaez
# Curso JavaScript Essentials 2 de Cisco NetAcad.
Capturas 
![image](https://github.com/user-attachments/assets/2e32c06c-884c-4ce7-b771-f88598d8e6ae)
![image](https://github.com/user-attachments/assets/a380681d-3860-4840-98a8-f4a9d2cf94d5)
![image](https://github.com/user-attachments/assets/1f4d848e-ad16-4ac5-a9ca-3119bbd9d4fc)
![image](https://github.com/user-attachments/assets/4c5e40d0-206a-4d4a-862d-f59747f4a50f)
![image](https://github.com/user-attachments/assets/387e048f-5e16-468f-91de-4fe5332a9839)


# Ejercicio 1: Comunicación Serial con Conversión de Datos
En lugar de solo enviar datos, el ESP32 deberá convertir la señal analógica de
un sensor en datos legibles en la PC.
1. Implementa una comunicación serial donde el ESP32 convierta la señal analógica de
un sensor en datos de temperatura o humedad y los envíe a la PC.

# Codigo:

    from machine import ADC, Pin
    from time import sleep

    # Configurar pin analógico (GPIO34)
    sensor = ADC(Pin(34))
    sensor.atten(ADC.ATTN_11DB)  # Rango de 0 a ~3.3V
    sensor.width(ADC.WIDTH_12BIT)  # Resolución de 12 bits (0-4095)

    while True:
        raw = sensor.read()  # Leer valor crudo
        voltage = raw * 3.3 / 4095  # Convertir a voltaje (0 - 3.3V)
        temperature = voltage * 100  # LM35 da 10mV/°C → multiplicamos por 100
    
        print("Temperatura: {:.2f} °C".format(temperature))
        sleep(1)




# Diagrama de conexion:
![image](https://github.com/user-attachments/assets/8475ea14-65a7-485a-840c-825666d60e82)

https://app.cirkitdesigner.com/project/e167f63f-3f83-4efb-8715-f4efb6f4180d
# Video Demostrativo:

https://drive.google.com/file/d/1ppaDaq1yY8j01wml8BvXqY_G36V2DAr6/view?usp=sharing

////////////////////////////////////////////////////////////////////////////

# Ejercicio 2: Comunicación Bluetooth con Control desde un Chatbot
En lugar de encender un LED, se usará un chatbot en Telegram o WhatsApp
para controlar el Bluetooth.
1. Configura un bot que, al recibir un mensaje en Telegram o WhatsApp, envíe
comandos a un ESP32 para encender/apagar un LED o mover un servo.
# Codigo:
## ESP32
    import bluetooth
    from ble_simple_peripheral import BLESimplePeripheral
    import time
    from machine import Pin, PWM


    servo_pin = 13  # Cambia al pin donde conectaste tu servo
    servo = PWM(Pin(servo_pin), freq=50)  # Frecuencia estándar para servos


    def set_servo_angle(angle):
        # Convertir de 0-180 grados a duty cycle (aprox. 40-115)
        # Los valores exactos pueden variar según el servo
        min_duty = 40
        max_duty = 115
        duty = min_duty + (max_duty - min_duty) * (angle / 180)
        servo.duty(int(duty))
        return angle


        ble = bluetooth.BLE()
        
        sp = BLESimplePeripheral(ble)
        
        
        led = Pin(2, Pin.OUT)  # El pin puede variar según tu placa ESP32
        
        def on_rx(data):
            """Función que se ejecuta cuando se reciben datos"""
            data_str = data.decode().strip()
            print("Datos recibidos:", data_str)
    
            # Procesar comando para el servo
            if data_str.startswith("SERVO:"):
                try:
                    # Extraer el ángulo del comando (SERVO:90)
                    angle = int(data_str.split(":")[1])
                    if 0 <= angle <= 180:
                        # Mover el servo al ángulo especificado
                        actual_angle = set_servo_angle(angle)
                        
                        # Parpadear LED para indicar que se recibió comando
                        led.value(1)
                        time.sleep(0.1)
                        led.value(0)
                        
                        # Enviar confirmación
                        if sp.is_connected():
                            sp.send(f"Servo movido a: {actual_angle} grados")
                    else:
                        if sp.is_connected():
                            sp.send("Error: El ángulo debe estar entre 0 y 180")
                except Exception as e:
                    if sp.is_connected():
                        sp.send(f"Error: {str(e)}")


        set_servo_angle(90)
        
        
        print("Esperando conexión Bluetooth...")
        while True:
            # Si está conectado, el método 'advertise' no hace nada
            if not sp.is_connected():
                # Si no está conectado, comenzar a anunciarse de nuevo
                sp.advertise()
                print("Esperando conexión...")
    
            # Revisar si se han recibido datos
            if sp.is_connected():
                # El callback 'on_rx' manejará los datos recibidos
                sp.on_write(on_rx)
                
                # Enviar un mensaje cada 3 segundos
                sp.send(f"ESP32 Servo Control - {time.ticks_ms()}")
            
            # Esperar un poco antes de la siguiente iteración
            time.sleep(3)
## servidor.py
        import asyncio
        import telebot
        from bleak import BleakClient, BleakScanner
        import threading


        TOKEN = '7930355394:AAFA0FyniOMS-Mbl-4bbIJdVw2jQ9QZeaz8'
        bot = telebot.TeleBot(TOKEN)

        UART_SERVICE_UUID = "6E400001-B5A3-F393-E0A9-E50E24DCCA9E"
        UART_RX_CHAR_UUID = "6E400002-B5A3-F393-E0A9-E50E24DCCA9E"  # Para escribir en ESP32
        UART_TX_CHAR_UUID = "6E400003-B5A3-F393-E0A9-E50E24DCCA9E"  # Para leer de ESP32
        
        
        ble_client = None
        device_name = "ESP32-BLE"  # Debe coincidir con el nombre en tu código ESP32
        
        
        reconnect_loop_running = True
        
        def notification_handler(sender, data):
            message = data.decode()
            print(f"Mensaje recibido del ESP32: {message}")
            # Aquí puedes procesar los mensajes recibidos del ESP32


        async def connect_ble():
            global ble_client
            
            print(f"Buscando dispositivo BLE: {device_name}...")
            
            try:
                # Buscar dispositivo por nombre
                device = await BleakScanner.find_device_by_name(device_name)
                
                if not device:
                    print(f"No se encontró el dispositivo: {device_name}")
                    return False
                
                print(f"Dispositivo encontrado. Intentando conectar a {device.name} [{device.address}]")
                
                # Conectar al dispositivo
                client = BleakClient(device.address)
                await client.connect()
        
                if client.is_connected:
                    print(f"Conectado a {device.name}")
                    
                    # Configurar para recibir notificaciones
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
                # Enviar comando a la característica RX
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
                await asyncio.sleep(5)  # Revisar cada 5 segundos


        def run_ble_loop():
            loop = asyncio.new_event_loop()
            asyncio.set_event_loop(loop)
            
            # Conectar por primera vez
            loop.run_until_complete(connect_ble())
            
            # Iniciar bucle de reconexión
            loop.run_until_complete(reconnection_loop())
        
        
        ble_thread = threading.Thread(target=run_ble_loop)
        ble_thread.daemon = True
        ble_thread.start()
        
        @bot.message_handler(commands=['start'])
        def send_welcome(message):
            bot.reply_to(message, "Hola! Envíame comandos como SERVO:90 para mover el servo a 90 grados.")

        @bot.message_handler(commands=['status'])
        def check_status(message):
            if ble_client and ble_client.is_connected:
                bot.reply_to(message, "✅ Conectado al ESP32")
            else:
                bot.reply_to(message, "❌ No hay conexión con el ESP32")
        
        @bot.message_handler(commands=['reconnect'])
        def force_reconnect(message):
            bot.reply_to(message, "Intentando reconectar con el ESP32...")
            # La reconexión se maneja automáticamente en el bucle de reconexión
        
        @bot.message_handler(func=lambda message: True)
        def handle_message(message):
            command = message.text.strip().upper()
    
            # Validar si el comando es tipo SERVO:90
            if command.startswith("SERVO:"):
                try:
                    angle = int(command.split(":")[1])
                    if 0 <= angle <= 180:
                        # En lugar de usar bluetooth.write, usamos asyncio para enviar el comando BLE
                        loop = asyncio.new_event_loop()
                        success = loop.run_until_complete(send_command(command))
                        
                        if success:
                            bot.reply_to(message, f"Ángulo enviado: {angle}°")
                        else:
                            bot.reply_to(message, "No se pudo enviar el comando. Verifica la conexión BLE.")
                    else:
                        bot.reply_to(message, "El ángulo debe estar entre 0 y 180.")
                except ValueError:
                    bot.reply_to(message, "Formato incorrecto. Usa SERVO:90, SERVO:45, etc.")
            else:
                bot.reply_to(message, "Comando no válido. Usa SERVO:[0-180]")
        
        try:
            print("Iniciando bot de Telegram...")
            bot.polling()
        except Exception as e:
            print(f"Error al iniciar el bot: {e}")
        finally:
            # Detener el bucle de reconexión y desconectar BLE
            reconnect_loop_running = False
            loop = asyncio.new_event_loop()
            loop.run_until_complete(disconnect_ble())
            
            input("Presiona Enter para salir...")

# Diagrama de conexion:
![image](https://github.com/user-attachments/assets/114cf3dc-eff2-4d66-a0e0-ca315e12db76)
https://app.cirkitdesigner.com/project/1ef0d36e-f27e-43f4-8a82-1f50b2a3a1cf

# Video Demostrativo:
https://drive.google.com/file/d/1N_5uSYCn2mDcxZRihNJNQ8O-zFBSUmbL/view?usp=sharing
https://drive.google.com/file/d/1ZJ3z1LFTrvK3uDXKCEAtmASMCrj8OZ0x/view?usp=sharing

////////////////////////////////////////////////////////////////////////////

# Ejercicio 3: Comunicación TCP/IP con Base de Datos:
En lugar de solo enviar datos, estos deben almacenarse en una base de datos
(SQLite, Firebase, PostgreSQL).
1. Implementa una comunicación TCP/IP donde un ESP32 registre datos en una base
de datos en la nube o local.

# Codigo:
## esp32
        from machine import ADC, Pin
        import network
        import socket
        import time

        # Conexión Wi-Fi
        ssid ='JAQUELINE'
        password ='12345678'

        sta = network.WLAN(network.STA_IF)
        sta.active(True)
        time.sleep(2)  # Espera para que se active correctamente
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
                s.connect(('192.168.209.182', 3000))  # IP del servidor
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


## server.js
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


## web.js
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

## Public/index.html
    <!DOCTYPE html>
    <html lang="es">
    <head>
      <meta charset="UTF-8">
      <title>Lecturas del Sensor</title>
      <style>
        body {
          font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
          background: #f0f2f5;
          margin: 0;
          padding: 0;
        }

        .container {
          max-width: 900px;
          margin: 40px auto;
          background: #fff;
          padding: 30px;
          border-radius: 12px;
          box-shadow: 0 4px 20px rgba(0, 0, 0, 0.1);
        }

        h1 {
          text-align: center;
          color: #333;
          margin-bottom: 30px;
        }

        table {
          width: 100%;
          border-collapse: collapse;
          background-color: #fff;
        }

        thead {
          background-color: #007BFF;
          color: white;
        }

        th, td {
          padding: 12px 15px;
          text-align: center;
          border-bottom: 1px solid #ddd;
        }

        tbody tr:hover {
          background-color: #f1f1f1;
        }

        td:nth-child(2) {
          font-weight: bold;
          color: #d63384;
        }

        @media (max-width: 600px) {
          .container {
            padding: 15px;
          }

          table, thead, tbody, th, td, tr {
            display: block;
            text-align: right;
          }

          thead {
            display: none;
          }

          tr {
            margin-bottom: 15px;
            border-bottom: 2px solid #eee;
          }

          td {
            padding: 10px;
            position: relative;
          }

          td::before {
            content: attr(data-label);
            position: absolute;
            left: 10px;
            font-weight: bold;
            color: #555;
          }
        }
      </style>
    </head>
    <body>
      <div class="container">
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
      </div>

      <script>
        fetch('http://localhost:3001/api/lecturas')
          .then(res => res.json())
          .then(data => {
            const tbody = document.querySelector('#tabla tbody');
            data.forEach(fila => {
              const tr = document.createElement('tr');
              tr.innerHTML = `
                <td data-label="ID">${fila.id}</td>
                <td data-label="Temperatura">${fila.valor} °C</td>
                <td data-label="Fecha">${fila.timestamp}</td>
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
![image](https://github.com/user-attachments/assets/8475ea14-65a7-485a-840c-825666d60e82)

https://app.cirkitdesigner.com/project/e167f63f-3f83-4efb-8715-f4efb6f4180d

# Video Demostrativo:
https://drive.google.com/file/d/1SE3qtyY9qSg3TJYmA3AfNMQ9mpdlfgq-/view?usp=sharing

////////////////////////////////////////////////////////////////////////////
# Ejercicio 4: Envío de Notificaciones con Discord Webhooks:
Se reemplaza el correo por una integración con Discord.
1. Configura un webhook de Discord para recibir notificaciones desde un ESP32
cuando un sensor detecte un evento.

# Codigo:
## esp32
        import network
        import urequests
        import time
        import json
        from machine import Pin
        import dht  # Librería para sensores DHT (DHT11 / DHT22)

        # Configuración WiFi y Webhook de Discord
        SSID = "JAQUELINE"
        PASSWORD = "12345678"
        WEBHOOK_URL = "https://discord.com/api/webhooks/1370209125182472273/8QrsDimtkimet2RnTOVVxXiV7Oppsym7b0mrsQvnXX6DrLd290Fqzrl5rqb8i4hDoKrv"

        # Sensor RQ-S003 (compatible con DHT11) en pin 14
        sensor_dht = dht.DHT11(Pin(14))

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
                mensaje = f"🌡️ Temp: {temp:.1f}°C | 💧 Humedad: {hum:.1f}%"
                print("Enviando datos:", mensaje)
                enviar_discord(mensaje)
            except Exception as e:
                print("Error leyendo sensor:", e)
            time.sleep(60)  # Espera 60 segundos antes de enviar otra lectura


# Diagrama de conexion:
![image](https://github.com/user-attachments/assets/8475ea14-65a7-485a-840c-825666d60e82)

https://app.cirkitdesigner.com/project/e167f63f-3f83-4efb-8715-f4efb6f4180d


# Video Demostrativo:

https://drive.google.com/file/d/12nBZMbUTrxo8oTNM2a1xLzAmCd0H-Uv-/view?usp=sharing

////////////////////////////////////////////////////////////////////////////

# Autor 
Camarillo Olaez Juana Jaqueline

# Autoevaluación y Coevaluación

Coevaluación
Durante el desarrollo del proyecto, observé en mí una actitud constante de esfuerzo y compromiso. A lo largo de las distintas etapas, trabajé intensamente para cumplir con los objetivos técnicos y personales del proyecto. Aunque atravesé momentos de procrastinación, algo que reconozco como una de mis principales debilidades, supe retomar el ritmo gracias a mi insistencia y perseverancia cuando las cosas no salían como esperaba.

Me caractericé por no rendirme fácilmente ante los problemas: cada vez que algo no funcionaba, busqué soluciones con insistencia, investigando y probando diferentes alternativas hasta dar con una respuesta satisfactoria. Esa actitud me permitió avanzar incluso en momentos de dificultad técnica o falta de claridad en las instrucciones.
