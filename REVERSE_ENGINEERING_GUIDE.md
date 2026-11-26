# Guía Práctica de Ingeniería Inversa y Análisis de Aplicaciones Android (APK)

> **Aviso Legal y Ético**: Esta guía tiene fines exclusivamente educativos y de auditoría de seguridad autorizada. El análisis e ingeniería inversa debe realizarse únicamente en aplicaciones de tu propiedad o en aquellas sobre las cuales tengas autorización explícita para auditar.

---

## Índice
1. [Obtención del Archivo APK](#1-obtención-del-archivo-apk)
2. [Análisis Estático (Ver Código y Funciones)](#2-análisis-estático-ver-código-y-funciones)
3. [Análisis Dinámico e Instrumentación (Ver Ejecución en Tiempo Real)](#3-análisis-dinámico-e-instrumentación-ver-ejecución-en-tiempo-real)
4. [Monitoreo de Red y Logs](#4-monitoreo-de-red-y-logs)
5. [Flujo Recomendado paso a paso](#5-flujo-recomendado-paso-a-paso)

---

## 1. Obtención del Archivo APK

Si tienes un enlace a la aplicación (ej. Google Play Store) o la aplicación instalada en un dispositivo:

- **Desde un dispositivo Android conectado por ADB**:
  ```bash
  # 1. Listar los paquetes instalados para hallar el nombre exacto
  adb shell pm list packages | grep mi_app

  # 2. Obtener la ruta del archivo APK en el dispositivo
  adb shell pm path com.ejemplo.miapp

  # 3. Descargar el APK a tu computadora
  adb pull /data/app/.../base.apk mi_aplicacion.apk
  ```

- **Desde tiendas de terceros o agregadores (si se parte de un link)**:
  - Repositorios conocidos como APKMirror, APKPure (verificar integridad mediante hash).
  - Herramientas CLI como `gplaycli` u la tienda `Aurora Store` (móvil/emulador).

---

## 2. Análisis Estático (Ver Código y Funciones)

El análisis estático te permite inspeccionar la estructura, clases, métodos y recursos sin ejecutar la app.

### A. JADX / JADX-GUI (Recomendado para ver código Java/Kotlin)
[JADX](https://github.com/skylot/jadx) decompila el código de bytes Dalvik (`.dex`) a código fuente legible en Java/Kotlin.

- **Uso**:
  ```bash
  jadx-gui mi_aplicacion.apk
  ```
- **Capacidades**:
  - Buscador global de texto (`Ctrl + Shift + F`): Buscar palabras clave como `"api_key"`, `"http"`, `"login"`, `"encrypt"`, etc.
  - Árbol de paquetes: Inspeccionar clases y métodos.
  - Cross-references (`x` sobre una función/método): Ver quién llama a esa función y a qué otras funciones llama.

### B. APKTool (Para recursos y código Smali)
[APKTool](https://apktool.org/) desensambla el APK en recursos (`res/`), el manifesto (`AndroidManifest.xml`) y código en ensamblador Dalvik (`.smali`).

- **Descomprimir APK**:
  ```bash
  apktool d mi_aplicacion.apk -o mi_app_descomprimida
  ```
- **Utilidad**:
  - Editar el `AndroidManifest.xml` (por ejemplo para hacer la app *debuggable* o permitir certificados de usuario).
  - Modificar código Smali y reensamblar:
    ```bash
    apktool b mi_app_descomprimida -o mi_app_modificada.apk
    ```

### C. Ghidra / IDA Pro (Librerías Nativas `.so`)
Si la aplicación utiliza código C/C++ cargado mediante JNI (archivos `.so` dentro de `lib/`):
- Abre el archivo `.so` en **Ghidra** para decompilar a código C y analizar funciones nativas.

---

## 3. Análisis Dinámico e Instrumentación (Ver Ejecución en Tiempo Real)

Para ver **cómo** se ejecutan las funciones en tiempo real, qué argumentos reciben y qué valores devuelven, se utiliza la instrumentación dinámica.

### A. Frida
[Frida](https://frida.re/) es el toolkit estándar para la ingeniería inversa dinámica. Permite inyectar código JavaScript en el proceso en ejecución de la app Android.

1. **Requisitos**:
   - Dispositivo Android con `root` o emulador (ej. Android Studio Emulator, Genymotion).
   - Instalar `frida-server` en el dispositivo y ejecutarlo como root.
   - Instalar `frida-tools` en la computadora (`pip install frida-tools`).

2. **Hooking de Funciones (Hook Script)**:
   Crea un archivo script `hook.js` para interceptar un método específico:

   ```javascript
   Java.perform(function () {
       // Localizar la clase objetivo
       var MiClase = Java.use("com.ejemplo.miapp.MiClase");

       // Interceptar el método "miFuncion"
       MiClase.miFuncion.implementation = function (arg1, arg2) {
           console.log("[+] miFuncion fue ejecutada!");
           console.log("    Argumento 1: " + arg1);
           console.log("    Argumento 2: " + arg2);

           // Llamar al método original y obtener el resultado
           var resultado = this.miFuncion(arg1, arg2);
           console.log("    Resultado devuelto: " + resultado);

           return resultado;
       };
   });
   ```

3. **Ejecutar Frida**:
   ```bash
   frida -U -f com.ejemplo.miapp -l hook.js
   ```

### B. Objection
[Objection](https://github.com/sensepost/objection) es una herramienta basada en Frida que no requiere escribir scripts manualmente.

- **Iniciar Objection**:
  ```bash
  objection --g com.ejemplo.miapp explore
  ```
- **Comandos útiles dentro de Objection**:
  - Evitar SSL Pinning: `android sslpinning disable`
  - Evitar detección de Root: `android root disable`
  - Monitorear invocaciones a una clase: `android hooking watch class com.ejemplo.miapp.MiClase`

---

## 4. Monitoreo de Red y Logs

### A. ADB Logcat
Permite ver las impresiones de consola (`Log.d`, `Log.e`, etc.) en tiempo real:
```bash
adb logcat | grep "com.ejemplo.miapp"
```

### B. Intercepción de Tráfico de Red (Burp Suite / OWASP ZAP)
1. Configurar la computadora como Proxy HTTP/HTTPS.
2. Configurar el Wi-Fi o Proxy en el dispositivo Android para apuntar a la IP de la computadora.
3. Instalar el certificado CA del Proxy en el dispositivo Android.
4. Usar Frida/Objection para eludir **SSL Pinning** si la app valida el certificado en código.

---

## 5. Flujo Recomendado paso a paso

1. **Obtener el APK** mediante `adb pull` o descarga segura.
2. **Abrir en JADX-GUI** para explorar el código Java/Kotlin y ubicar los nombres de las clases y funciones de interés.
3. **Lanzar la app en un emulador o dispositivo de pruebas**.
4. **Usar Frida u Objection** para conectar con el proceso y aplicar *hooks* a las funciones identificadas en JADX.
5. **Interactuar con la interfaz de la app** y observar los argumentos, llamadas y respuestas impresos en la consola de Frida/Logcat/Burp Suite.
