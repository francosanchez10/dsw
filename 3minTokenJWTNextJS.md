Entiendo que estás utilizando **Next.js** para desarrollar tu aplicación.

A continuación, te proporcionaré una guía adaptada para implementar este sistema utilizando Next.js, incluyendo ejemplos de código específicos para este framework.

---

### **1. Generación de Enlaces Temporales en Next.js**

#### a. Generación del Token con JWT

Utilizaremos el paquete `jsonwebtoken` para generar tokens que expiran en 3 minutos.

Instala el paquete:

```bash
npm install jsonwebtoken
```

#### b. Creación de una API Route para Generar el Token

En Next.js, puedes crear rutas de API en la carpeta `pages/api`.

Crea un archivo `pages/api/generar-token.js`:

```javascript
// pages/api/generar-token.js
import jwt from 'jsonwebtoken';

export default function handler(req, res) {
  const { obraId } = req.query;

  const payload = { obraId };
  const opciones = { expiresIn: '3m' }; // Expira en 3 minutos
  const token = jwt.sign(payload, process.env.JWT_SECRET, opciones);

  res.status(200).json({ token });
}
```

Asegúrate de definir `JWT_SECRET` en tus variables de entorno.

### **2. Generación y Actualización del QR Code**

#### a. Generar el QR Code en el Servidor

Instala el paquete `qrcode`:

```bash
npm install qrcode
```

Crea una API Route para generar el QR code:

```javascript
// pages/api/generar-qr.js
import jwt from 'jsonwebtoken';
import QRCode from 'qrcode';

export default async function handler(req, res) {
  const { obraId } = req.query;

  const payload = { obraId };
  const opciones = { expiresIn: '3m' };
  const token = jwt.sign(payload, process.env.JWT_SECRET, opciones);

  const url = `${process.env.NEXT_PUBLIC_BASE_URL}/votar?token=${token}`;

  try {
    const qrCodeDataURL = await QRCode.toDataURL(url);
    res.status(200).json({ qrCodeDataURL });
  } catch (error) {
    res.status(500).json({ error: 'Error al generar el QR code' });
  }
}
```

Define `NEXT_PUBLIC_BASE_URL` en tus variables de entorno para apuntar a tu dominio.

#### b. Página en la Tableta para Mostrar el QR Code

Crea una página en `pages/mostrar-qr.js`:

```jsx
// pages/mostrar-qr.js
import { useEffect, useState } from 'react';

export default function MostrarQR({ obraId }) {
  const [qrCodeDataURL, setQrCodeDataURL] = useState('');

  const fetchQRCode = async () => {
    const response = await fetch(`/api/generar-qr?obraId=${obraId}`);
    const data = await response.json();
    setQrCodeDataURL(data.qrCodeDataURL);
  };

  useEffect(() => {
    fetchQRCode();
    const interval = setInterval(fetchQRCode, 180000); // Cada 3 minutos
    return () => clearInterval(interval);
  }, []);

  return (
    <div>
      <h1>Escanea para votar por la obra {obraId}</h1>
      {qrCodeDataURL && <img src={qrCodeDataURL} alt="QR Code" />}
    </div>
  );
}

// Si estás usando getServerSideProps para obtener obraId
export async function getServerSideProps(context) {
  const { obraId } = context.query;
  return { props: { obraId } };
}
```

### **3. Verificación de la Validez del Enlace al Votar**

Crea una página para manejar la votación en `pages/votar.js`:

```jsx
// pages/votar.js
import jwt from 'jsonwebtoken';

export default function Votar({ valido, obraId }) {
  if (!valido) {
    return <p>El enlace ha expirado o es inválido. Por favor, escanea el QR nuevamente.</p>;
  }

  return (
    <div>
      <h1>Votando por la obra {obraId}</h1>
      {/* Aquí puedes colocar tu formulario de votación */}
    </div>
  );
}

export async function getServerSideProps(context) {
  const { token } = context.query;

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    return {
      props: { valido: true, obraId: decoded.obraId },
    };
  } catch (error) {
    return {
      props: { valido: false },
    };
  }
}
```

### **4. Prevención de Votaciones Múltiples**

Para evitar que un usuario vote varias veces, puedes:

- **Usar Cookies**: Establece una cookie cuando el usuario vote.
- **Registro por IP**: Registra la dirección IP, aunque no es 100% confiable.
- **Autenticación**: Solicita que los usuarios se autentiquen antes de votar.

Ejemplo usando cookies:

```jsx
// Dentro de la función que maneja el envío del voto
export default function Votar({ valido, obraId }) {
  // ...código anterior

  const handleVotar = () => {
    // Verifica si la cookie 'haVotado' existe
    if (document.cookie.includes(`haVotadoObra${obraId}=true`)) {
      alert('Ya has votado por esta obra.');
      return;
    }

    // Procesa el voto y luego establece la cookie
    document.cookie = `haVotadoObra${obraId}=true; max-age=86400; path=/`;

    alert('¡Gracias por tu voto!');
  };

  // ...resto del componente
}
```

### **5. Sincronización de Tiempo**

- **Servidor y Cliente en Sincronía**: Asegúrate de que el servidor donde se generan y verifican los tokens tiene la hora correcta.
- **Uso de UTC**: Trabaja con tiempos en UTC para evitar problemas de zona horaria.

### **6. Seguridad Adicional**

- **HTTPS**: Asegúrate de que tu sitio está sirviendo contenido a través de HTTPS.
- **Protección de Rutas**: Las rutas de API que generan tokens deben estar protegidas si es necesario.
- **Validación en el Servidor**: Siempre valida los tokens en el servidor utilizando `getServerSideProps` o APIs.

### **7. Variables de Entorno**

En Next.js, las variables de entorno se manejan en archivos `.env.local`.

Ejemplo de un archivo `.env.local`:

```
JWT_SECRET=tu_clave_secreta
NEXT_PUBLIC_BASE_URL=https://tusitio.com
```

Recuerda no exponer `JWT_SECRET` en el cliente. Solo utiliza variables que comiencen con `NEXT_PUBLIC_` en el cliente.

### **8. Manejo de Errores y Experiencia de Usuario**

- **Mensajes Claros**: Informa al usuario si el enlace ha expirado.
- **Redirección**: Opcionalmente, puedes redirigir al usuario a una página donde pueda solicitar un nuevo QR.

### **9. Optimización y Buenas Prácticas**

- **Cacheo**: Considera cachear los QR codes durante su periodo de validez para reducir la carga del servidor.
- **Optimización de Imágenes**: Utiliza el componente `next/image` para servir el QR code si lo guardas como imagen.

### **10. Resumen de Pasos**

1. **Generar Tokens Temporales**: Utiliza JWT en una API Route para generar tokens que expiran en 3 minutos.
2. **Generar QR Codes**: Crea una API Route que genere el QR code con el enlace que incluye el token.
3. **Actualizar el QR Code en la Tableta**: Implementa una página en Next.js que actualice el QR code cada 3 minutos utilizando `useEffect` y `setInterval`.
4. **Verificar Tokens al Votar**: En la página de votación, utiliza `getServerSideProps` para verificar el token antes de renderizar la página.
5. **Prevenir Votaciones Múltiples**: Implementa lógica en el frontend y/o backend para evitar que un usuario vote más de una vez.

---

Espero que esta guía adaptada a Next.js te sea de ayuda para implementar el sistema de votación con QR codes que necesitas. Si tienes más preguntas o necesitas más detalles sobre algún paso, no dudes en consultarme.
