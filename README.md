Hola,

Claro, puedo ayudarte a generar un token que expira en 3 minutos utilizando **Next.js** en un **componente de servidor** (Server Component). A continuación, te mostraré cómo hacerlo paso a paso.

---

### **Paso 1: Configurar un Proyecto de Next.js**

Si aún no tienes un proyecto de Next.js, puedes crear uno ejecutando:

```bash
npx create-next-app@latest mi-aplicacion
```

### **Paso 2: Instalar las Dependencias Necesarias**

Necesitarás instalar el paquete `jsonwebtoken` para generar y firmar los tokens JWT.

```bash
npm install jsonwebtoken
```

### **Paso 3: Configurar Variables de Entorno**

Crea un archivo `.env.local` en la raíz de tu proyecto para almacenar tu clave secreta:

```
SECRET_KEY=tu_clave_secreta_segura
```

**Nota**: Asegúrate de que `.env.local` está en tu `.gitignore` para que no se suba al repositorio.

### **Paso 4: Crear un Componente de Servidor que Genere el Token**

En Next.js 13 y versiones superiores, puedes utilizar la carpeta `app` para crear rutas y componentes de servidor.

#### **a. Estructura de Archivos**

Crea un endpoint en `/api/generar-token`:

```
mi-aplicacion/
├── app/
│   └── api/
│       └── generar-token/
│           └── route.js
```

#### **b. Código del Endpoint `route.js`**

```javascript
// app/api/generar-token/route.js
import jwt from 'jsonwebtoken';

export async function GET(request) {
  const SECRET_KEY = process.env.SECRET_KEY;
  const ALGORITHM = 'HS256';

  // Datos a incluir en el token
  const data = {
    // Por ejemplo, el ID de la obra o cualquier información relevante
    obraId: 123,
  };

  // Configuración de expiración a 3 minutos
  const expiresIn = '3m';

  // Generar el token
  const token = jwt.sign(data, SECRET_KEY, {
    algorithm: ALGORITHM,
    expiresIn: expiresIn,
  });

  return new Response(JSON.stringify({ token }), {
    status: 200,
    headers: { 'Content-Type': 'application/json' },
  });
}
```

#### **c. Explicación del Código**

- **Importación de `jsonwebtoken`**: Para generar y firmar el token JWT.
- **Configuración de Clave Secreta y Algoritmo**: Se utiliza la clave secreta de las variables de entorno y el algoritmo HS256.
- **Datos del Token**: Incluimos datos como `obraId` en el token.
- **Establecer la Expiración**: Usamos `'3m'` para que el token expire en 3 minutos.
- **Generación del Token**: Usamos `jwt.sign()` para crear el token.
- **Respuesta HTTP**: Devolvemos el token en formato JSON.

### **Paso 5: Acceder al Endpoint para Obtener el Token**

Puedes probar el endpoint accediendo a `http://localhost:3000/api/generar-token` en tu navegador o mediante herramientas como `curl` o Postman.

La respuesta será:

```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

### **Paso 6: Generar un Enlace con el Token**

Ahora, genera un enlace que incluya el token como parámetro para que pueda ser escaneado mediante un QR.

#### **a. Modificar el Endpoint para Incluir el Enlace**

```javascript
// app/api/generar-token/route.js
import jwt from 'jsonwebtoken';

export async function GET(request) {
  const SECRET_KEY = process.env.SECRET_KEY;
  const ALGORITHM = 'HS256';

  const data = {
    obraId: 123,
  };

  const expiresIn = '3m';

  const token = jwt.sign(data, SECRET_KEY, {
    algorithm: ALGORITHM,
    expiresIn: expiresIn,
  });

  // Generar la URL que contiene el token
  const urlConToken = `https://tu-dominio.com/votar?token=${token}`;

  return new Response(JSON.stringify({ urlConToken }), {
    status: 200,
    headers: { 'Content-Type': 'application/json' },
  });
}
```

### **Paso 7: Generar el Código QR**

Puedes generar un código QR que contenga el enlace con el token.

#### **a. Instalar una Biblioteca para Generar QR Codes**

```bash
npm install qrcode
```

#### **b. Modificar el Endpoint para Incluir el QR Code**

```javascript
// app/api/generar-token/route.js
import jwt from 'jsonwebtoken';
import QRCode from 'qrcode';

export async function GET(request) {
  const SECRET_KEY = process.env.SECRET_KEY;
  const ALGORITHM = 'HS256';

  const data = {
    obraId: 123,
  };

  const expiresIn = '3m';

  const token = jwt.sign(data, SECRET_KEY, {
    algorithm: ALGORITHM,
    expiresIn: expiresIn,
  });

  const urlConToken = `https://tu-dominio.com/votar?token=${token}`;

  // Generar el QR Code como Data URL
  const qrCodeDataURL = await QRCode.toDataURL(urlConToken);

  return new Response(JSON.stringify({ qrCodeDataURL }), {
    status: 200,
    headers: { 'Content-Type': 'application/json' },
  });
}
```

### **Paso 8: Mostrar el QR Code en el Cliente**

En un componente de cliente, puedes mostrar el QR code y actualizarlo cada 3 minutos.

#### **a. Componente de Cliente para Mostrar el QR Code**

```javascript
// app/page.js
'use client';

import { useEffect, useState } from 'react';

export default function QRCodePage() {
  const [qrCodeDataURL, setQrCodeDataURL] = useState('');

  useEffect(() => {
    const fetchQrCode = () => {
      fetch('/api/generar-token')
        .then((res) => res.json())
        .then((data) => {
          setQrCodeDataURL(data.qrCodeDataURL);
        })
        .catch((error) => {
          console.error('Error al obtener el QR Code:', error);
        });
    };

    fetchQrCode(); // Obtener el QR code al montar el componente

    const intervalId = setInterval(fetchQrCode, 180000); // Actualizar cada 3 minutos

    return () => clearInterval(intervalId); // Limpiar el intervalo al desmontar
  }, []);

  return (
    <div>
      <h1>Escanea el QR para votar por la obra</h1>
      {qrCodeDataURL ? (
        <img src={qrCodeDataURL} alt="Código QR" />
      ) : (
        <p>Cargando...</p>
      )}
    </div>
  );
}
```

### **Paso 9: Crear la Ruta para Procesar el Voto**

Necesitas una ruta que verifique el token y permita al usuario votar.

#### **a. Crear el Endpoint `/votar`**

Crea un archivo en `app/votar/page.js`.

```javascript
// app/votar/page.js
import jwt from 'jsonwebtoken';
import { redirect } from 'next/navigation';

export default async function VotarPage({ searchParams }) {
  const { token } = searchParams;

  const SECRET_KEY = process.env.SECRET_KEY;
  const ALGORITHM = 'HS256';

  try {
    const decoded = jwt.verify(token, SECRET_KEY, { algorithms: [ALGORITHM] });

    const obraId = decoded.obraId;

    // Aquí puedes renderizar la página de votación
    return (
      <div>
        <h1>Votando por la obra {obraId}</h1>
        {/* Formulario o contenido de votación */}
      </div>
    );
  } catch (error) {
    // Si el token es inválido o ha expirado, redirigir o mostrar un mensaje
    return (
      <div>
        <h1>El enlace ha expirado o es inválido.</h1>
        <p>Por favor, escanea el QR nuevamente.</p>
      </div>
    );
  }
}
```

#### **b. Explicación del Código**

- **Lectura del Token**: Obtenemos el token de los parámetros de la URL.
- **Verificación del Token**: Usamos `jwt.verify()` para verificar la validez y la firma del token.
- **Manejo de Errores**: Si el token es inválido o ha expirado, mostramos un mensaje al usuario.

### **Paso 10: Consideraciones de Seguridad**

- **Validación en el Servidor**: Siempre valida el token en el servidor para evitar manipulaciones.
- **HTTPS**: Utiliza HTTPS para asegurar la comunicación.
- **Prevención de Votaciones Múltiples**: Implementa lógica para evitar que un usuario vote varias veces.

### **Paso 11: Sincronización de Tiempo**

Asegúrate de que el servidor y los dispositivos de los usuarios estén sincronizados en cuanto a la hora, para evitar problemas con la expiración del token.

---

Espero que esta guía te sea útil para generar un token que expira en 3 minutos en Next.js utilizando un componente de servidor. Si tienes más preguntas o necesitas ayuda adicional, ¡no dudes en consultarme!
