# Socket Server Multi (Chat + Usuarios + Gráfica en tiempo real)

Servidor **Node.js + Express + Socket.IO** en **TypeScript** que expone:

- **Chat** en tiempo real (broadcast y privado por `id`)
- **Lista de usuarios activos** (con nombre configurado)
- **Gráfica de ventas** en tiempo real (vía WebSocket y REST)

> Probado con **Node 14+** (funciona en versiones más recientes).

---

## 🚀 Inicio rápido

```bash
# 1) Instalar dependencias
npm install

# 2) Instalar TypeScript si no lo tienes en dev
npm i -D typescript @types/node

# 3) Compilar a dist/
npx tsc

# 4) Ejecutar el servidor
node dist/index.js
# o en desarrollo (autoreload) si tienes nodemon:
npx nodemon dist/index.js
```

El servidor arranca por defecto en **http://localhost:5000**  
(Puerto configurable con la variable de entorno `PORT`).

---

## 📁 Estructura relevante

```
.
├─ classes/
│  ├─ grafica.ts               # Lógica de la gráfica (meses/valores)
│  ├─ server.ts                # Clase Server (Express + Socket.IO)
│  ├─ usuario.ts               # Entidad Usuario
│  └─ usuarios-lista.ts        # Gestión de usuarios activos
├─ sockets/
│  └─ socket.ts                # Eventos Socket.IO (conectar, mensaje, etc.)
├─ routes/
│  └─ router.ts                # Endpoints REST (/grafica, /usuarios...)
├─ global/
│  └─ environment.ts           # SERVER_PORT
├─ index.ts                    # Bootstrap del servidor
├─ tsconfig.json               # Config TS (outDir=dist/)
└─ package.json
```

---

## 🔌 Endpoints REST

### 1) Obtener datos de la gráfica
**GET** `/grafica`

**Respuesta (ejemplo):**
```json
[
  { "data": [10, 5, 7, 3], "label": "Ventas" }
]
```

---

### 2) Incrementar valor de un mes
**POST** `/grafica`

**Body:**
```json
{ "mes": "enero", "unidades": 5 }
```

**Respuesta:** mismo formato que `/grafica` (datos actualizados)  
**Side-effect:** emite **`cambio-grafica`** por Socket.IO a todos los clientes.

**Curl:**
```bash
curl -X POST http://localhost:5000/grafica   -H "Content-Type: application/json"   -d '{"mes":"enero","unidades":5}'
```

---

### 3) Enviar mensaje **privado** a un cliente (por ID de socket)
**POST** `/mensajes/:id`

**Body:**
```json
{ "de": "Alice", "cuerpo": "Hola Bob!" }
```

**Respuesta:**
```json
{ "ok": true, "cuerpo": "...", "de": "...", "id": "SOCKET_ID" }
```

**Side-effect:** emite **`mensaje-privado`** al socket con ese `id`.

**Curl:**
```bash
curl -X POST http://localhost:5000/mensajes/SOCKET_ID_AQUI   -H "Content-Type: application/json"   -d '{"de":"Alice","cuerpo":"Hola Bob!"}'
```

---

### 4) Listar **IDs** de sockets conectados
**GET** `/usuarios`

**Respuesta:**
```json
{ "ok": true, "clientes": ["id1","id2","..."] }
```

---

### 5) Listar usuarios con **detalle** (id / nombre / sala)
**GET** `/usuarios/detalle`

**Respuesta:**
```json
{
  "ok": true,
  "clientes": [
    { "id": "...", "nombre": "Alice", "sala": "sin-sala" }
  ]
}
```

---

## 🧠 Eventos Socket.IO

### Cliente ➜ Servidor
- **`configurar-usuario`**  
  **payload:** `{ nombre: string }`  
  **callback:** `{ ok: boolean, mensaje: string }`  
  **efectos:** asigna nombre al socket y **broadcast** `usuarios-activos`.

- **`mensaje`**  
  **payload:** `{ de: string, cuerpo: string }`  
  **efectos:** **broadcast** `mensaje-nuevo` a todos.

- **`obtener-usuarios`**  
  **efectos:** responde SOLO al cliente emisor con **`usuarios-activos`**.

### Servidor ➜ Clientes
- **`usuarios-activos`** → lista de usuarios (al entrar/salir/configurar)
- **`mensaje-nuevo`** → nuevo mensaje público
- **`mensaje-privado`** → solo al `id` indicado (vía `/mensajes/:id`)
- **`cambio-grafica`** → cuando se actualiza la gráfica por `/grafica`

---

## 🔄 Ciclo de vida de conexión

1. Al **conectar**, se crea un `Usuario` con `id` de socket y `nombre = 'sin-nombre'`.
2. Tras **`configurar-usuario`**, se guarda el nombre y se **notifica** `usuarios-activos`.
3. Al **desconectar**, se elimina el usuario y se **notifica** `usuarios-activos`.

> La lista devuelta por `UsuariosLista.getLista()` **excluye** usuarios con `nombre = 'sin-nombre'`.

---

## 🔧 Configuración

- **Puerto** en `global/environment.ts`:
  ```ts
  export const SERVER_PORT: number = Number(process.env.PORT) || 5000;
  ```
- Arrancar en otro puerto:
  ```bash
  PORT=3000 node dist/index.js
  ```

---

## 📦 Scripts recomendados

Añade a `package.json`:

```json
{
  "scripts": {
    "build": "tsc",
    "start": "node dist/index.js",
    "dev": "nodemon dist/index.js",
    "watch": "tsc -w"
  }
}
```

Uso:
```bash
npm run build   # compila a dist/
npm run start   # producción
npm run watch   # compilación en caliente
npm run dev     # autoreload con nodemon
```

---

## 🔗 Integración con cliente (ejemplo mínimo)

```ts
import io from 'socket.io-client';

const socket = io('http://localhost:5000');

// Configurar usuario
socket.emit('configurar-usuario', { nombre: 'Alice' }, (resp: any) => {
  console.log('Config usuario:', resp);
});

// Escuchar lista de usuarios
socket.on('usuarios-activos', (lista: any[]) => {
  console.log('Usuarios:', lista);
});

// Escuchar mensajes públicos
socket.on('mensaje-nuevo', (msg: any) => {
  console.log('Mensaje:', msg);
});

// Enviar mensaje público
socket.emit('mensaje', { de: 'Alice', cuerpo: 'Hola a todos!' });

// Escuchar actualización de gráfica
socket.on('cambio-grafica', (data: any) => {
  console.log('Grafica:', data);
});
```

---

## 📝 Notas

- Los meses válidos en la gráfica están definidos en `classes/grafica.ts`:
  ```ts
  private meses: string[] = ['enero', 'febrero', 'marzo', 'abril'];
  ```
- El endpoint `POST /grafica` **suma** unidades al mes dado y dispara `cambio-grafica`.
- `GET /usuarios` devuelve **IDs de sockets** (útil para enviar privados con `/mensajes/:id`).
- `GET /usuarios/detalle` devuelve **id/nombre/sala** de cada usuario con nombre configurado.

---

## 📄 Licencia

MIT (o la que prefieras).
