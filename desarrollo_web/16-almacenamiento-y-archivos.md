# Módulo 16: Almacenamiento y Archivos

## Objetivos de aprendizaje

Al finalizar este módulo, el estudiante será capaz de:

- Subir archivos desde el cliente al servidor de forma segura.
- Manejar `multipart/form-data` y `FormData`.
- Procesar y comprimir imágenes.
- Servir archivos localmente y decidir cuándo migrar a la nube.
- Subir archivos a S3-compatible (AWS S3, MinIO, R2).
- Optimizar entrega con CDNs y cacheo.

---

## 1. Upload de archivos (general)

### Anatomía

```
Cliente                 Server                   Almacenamiento
(FormData)── POST ───▶ (Multer/parse) ───▶ (Save to disk/S3)
```

### Cliente: FormData

```html
<form id="upload">
  <input type="file" name="avatar" accept="image/*" multiple>
  <button>Subir</button>
</form>
```

```js
const form = document.querySelector("#upload");

form.addEventListener("submit", async (e) => {
  e.preventDefault();
  const fd = new FormData(form);
  // Validación cliente (no suficiente)
  for (const file of fd.getAll("avatar")) {
    if (file.size > 5 * 1024 * 1024) {
      alert("Demasiado grande");
      return;
    }
  }

  const res = await fetch("/api/upload", {
    method: "POST",
    body: fd,                       // sin Content-Type
    headers: { Authorization: `Bearer ${token}` },
  });

  if (!res.ok) throw new Error("Error al subir");
  const json = await res.json();
  console.log(json.urls);
});
```

> El navegador pone automáticamente `Content-Type: multipart/form-data; boundary=...`. **No lo pongas tú**.

### Subida múltiple y progresiva

```js
for (const file of files) {
  const fd = new FormData();
  fd.append("file", file);
  await fetch("/api/upload", { method: "POST", body: fd });
}
```

### Barra de progreso con `fetch`

```js
async function uploadWithProgress(url, file, onProgress) {
  return new Promise((resolve, reject) => {
    const xhr = new XMLHttpRequest();
    xhr.open("POST", url);
    xhr.upload.onprogress = (e) => {
      if (e.lengthComputable) onProgress(e.loaded / e.total);
    };
    xhr.onload = () => (xhr.status < 300 ? resolve(xhr.response) : reject(xhr.response));
    xhr.onerror = reject;
    const fd = new FormData();
    fd.append("file", file);
    xhr.send(fd);
  });
}

await uploadWithProgress("/api/upload", file, (p) => {
  console.log(`${(p * 100).toFixed(0)}%`);
});
```

### Drag & Drop

```html
<div id="drop" class="drop">
  Arrastra archivos aquí
</div>
```

```js
const drop = document.querySelector("#drop");

["dragover", "dragenter"].forEach((ev) => drop.addEventListener(ev, (e) => {
  e.preventDefault();
  drop.classList.add("drag");
}));

drop.addEventListener("dragleave", () => drop.classList.remove("drag"));

drop.addEventListener("drop", (e) => {
  e.preventDefault();
  drop.classList.remove("drag");
  for (const file of e.dataTransfer.files) {
    uploadFile(file);
  }
});
```

### Paste de imágenes

```js
document.addEventListener("paste", async (e) => {
  for (const item of e.clipboardData.items) {
    if (item.type.startsWith("image/")) {
      const file = item.getAsFile();
      uploadFile(file);
    }
  }
});
```

### Actividad práctica

- Implementa un formulario con drag & drop, paste y barra de progreso usando `XMLHttpRequest`.

---

## 2. Multipart `FormData`

### Estructura del estándar

Cada parte puede tener su propio `Content-Type` y cabeceras (`Content-Disposition`).

```
--BOUNDARY
Content-Disposition: form-data; name="campo"; filename="avatar.png"
Content-Type: image/png

(binary data)
--BOUNDARY
Content-Disposition: form-data; name="email"

ana@example.com
--BOUNDARY--
```

### Métodos clave

```js
const fd = new FormData();
fd.append("campo", "valor");
fd.append("campo", "otro");      // múltiples con el mismo nombre

fd.set("campo", "nuevo valor");  // reemplaza

fd.append("archivo", file);     // File / Blob
fd.append("archivo", file, "foto.png");

fd.get("campo");
fd.getAll("campo");
fd.has("campo");
fd.delete("campo");

for (const [k, v] of fd.entries()) console.log(k, v);

const obj = Object.fromEntries(fd.entries());
```

### En backend con Express

```bash
npm install multer
```

```js
import multer from "multer";

const storage = multer.diskStorage({
  destination: "uploads/",
  filename: (req, file, cb) => cb(null, `${Date.now()}-${file.originalname}`),
});

const upload = multer({
  storage,
  limits: { fileSize: 5 * 1024 * 1024 }, // 5MB
  fileFilter: (req, file, cb) => {
    const allowed = ["image/png", "image/jpeg", "image/webp"];
    if (!allowed.includes(file.mimetype)) return cb(new Error("Tipo no permitido"));
    cb(null, true);
  },
});

router.post("/upload", authMiddleware, upload.single("file"), (req, res) => {
  console.log(req.file, req.body);
  res.json({ url: `/uploads/${req.file.filename}` });
});
```

### Memory storage + procesamiento

```js
const upload = multer({ storage: multer.memoryStorage(), limits: { fileSize: 5 * 1024 * 1024 } });

router.post("/upload", upload.single("file"), async (req, res) => {
  const buffer = req.file.buffer;
  const processed = await sharp(buffer).resize(800).webp({ quality: 80 }).toBuffer();
  await uploadToS3(processed, `${req.user.id}.webp`);
  res.json({ ok: true });
});
```

### `busboy` (más flexible)

```js
import busboy from "busboy";

router.post("/upload", (req, res) => {
  const bb = busboy({ headers: req.headers, limits: { files: 5, fileSize: 5 * 1024 * 1024 } });
  let totalBytes = 0;

  bb.on("file", (name, file, info) => {
    file.on("data", (data) => (totalBytes += data.length));
    file.on("limit", () => res.status(413).end("Demasiado grande"));
    file.on("end", () => console.log(`File ${name} ended`));
    file.pipe(fs.createWriteStream(`uploads/${Date.now()}-${info.filename}`));
  });

  bb.on("field", (name, val) => console.log(`Field ${name}: ${val}`));
  bb.on("close", () => res.json({ totalBytes }));
  req.pipe(bb);
});
```

### Validaciones críticas (backend)

- **Tipo MIME** real (no el nombre del archivo).
- **Magic numbers** (`file-type`).
- Tamaño.
- Cantidad de archivos.
- Rate limiting.

```js
import { fileTypeFromBuffer } from "file-type";

const buffer = req.file.buffer;
const type = await fileTypeFromBuffer(buffer);
if (!type || !["image/png", "image/jpeg"].includes(type.mime)) {
  throw new Error("Tipo no permitido");
}
```

### Actividad práctica

- Implementa un endpoint con `multer`, validación de tipo MIME real y límite de tamaño.

---

## 3. Imágenes (procesamiento)

### Sharp (Node)

```bash
npm install sharp
```

```js
import sharp from "sharp";

// Convertir y comprimir
await sharp("input.png")
  .resize(800, 600, { fit: "inside", withoutEnlargement: true })
  .webp({ quality: 80 })
  .toFile("output.webp");

// Metadata
const meta = await sharp("input.png").metadata();
console.log(meta.width, meta.height, meta.format);
```

### Variantes (responsive)

```js
const sizes = [
  { suffix: "small",  width: 320 },
  { suffix: "medium", width: 640 },
  { suffix: "large",  width: 1280 },
];

for (const { suffix, width } of sizes) {
  await sharp(buffer)
    .resize(width)
    .webp({ quality: 80 })
    .toFile(`public/img/${name}-${suffix}.webp`);
}
```

### Crop, rotar, blur

```js
await sharp(buffer)
  .resize(400, 300, { fit: "cover", position: "attention" })
  .rotate(90)
  .blur(2)
  .toBuffer();
```

### Texto, marca de agua

```js
const svg = `<svg width="800" height="600"><text x="20" y="580" fill="white" font-size="24">@username</text></svg>`;

await sharp(buffer)
  .composite([{ input: Buffer.from(svg), gravity: "southwest" }])
  .toFile("output.jpg");
```

### Placeholder LQIP

```js
const lqip = await sharp(buffer).resize(20).blur(5).toBuffer();
const dataUri = `data:image/jpeg;base64,${lqip.toString("base64")}`;
```

### `<picture>` y `srcset` (HTML)

```html
<picture>
  <source type="image/webp" srcset="img-320.webp 320w, img-640.webp 640w" sizes="(max-width: 600px) 100vw, 50vw">
  <img src="img-640.jpg" alt="" loading="lazy" width="640" height="480">
</picture>
```

### AVIF

```js
await sharp(buffer).resize(800).avif({ quality: 60 }).toBuffer();
```

Mejor compresión que WebP, ~20% más pequeño, pero más lento de codificar.

### Actividad práctica

- Implementa un endpoint que reciba una imagen, genere 3 tamaños en WebP, los suba a S3 y devuelva las URLs.

---

## 4. Compresión de imágenes

### Configuraciones típicas

| Formato | Calidad típica | Uso                      |
| ------- | -------------- | ------------------------ |
| JPEG    | 75-85          | Fotos                    |
| WebP    | 75-85          | Fotos + gráficos         |
| AVIF    | 60-75          | Fotos modernas           |
| PNG     | lossless       | Logos, capturas          |
| SVG     | texto          | Iconos, logos            |

### Recompresión con Sharp

```js
const targetSize = 200 * 1024; // 200 KB

for (let q = 80; q >= 30; q -= 5) {
  const out = await sharp(buffer).webp({ quality: q }).toBuffer();
  if (out.length <= targetSize) break;
}
```

### Compresión en el cliente (antes de subir)

```js
async function compress(file, maxW = 1600, quality = 0.8) {
  const img = await createImageBitmap(file);
  const scale = Math.min(1, maxW / img.width);
  const canvas = new OffscreenCanvas(img.width * scale, img.height * scale);
  const ctx = canvas.getContext("2d");
  ctx.drawImage(img, 0, 0, canvas.width, canvas.height);
  const blob = await canvas.convertToBlob({ type: "image/webp", quality });
  return new File([blob], file.name, { type: blob.type });
}

const compressed = await compress(file);
const fd = new FormData();
fd.append("file", compressed);
await fetch("/api/upload", { method: "POST", body: fd });
```

### Compresión con `<canvas>` clásico

```js
function compressImage(file, quality = 0.7, maxWidth = 1280) {
  return new Promise((resolve) => {
    const img = new Image();
    img.onload = () => {
      const scale = Math.min(1, maxWidth / img.width);
      const canvas = document.createElement("canvas");
      canvas.width  = img.width  * scale;
      canvas.height = img.height * scale;
      const ctx = canvas.getContext("2d");
      ctx.drawImage(img, 0, 0, canvas.width, canvas.height);
      canvas.toBlob(resolve, "image/jpeg", quality);
    };
    img.src = URL.createObjectURL(file);
  });
}
```

### Buenas prácticas

- Redimensionar antes de comprimir (subir 4K para mostrar 600px es desperdicio).
- Servir `image/webp` con fallback JPEG.
- `loading="lazy"` y `decoding="async"`.
- `width`/`height` para evitar layout shift.

### Actividad práctica

- Comprime una imagen en el navegador a WebP antes de subirla y compara el tamaño.

---

## 5. Almacenamiento local (filesystem)

Útil en desarrollo y apps pequeñas. No recomendado para producción distribuida.

### Servir en Express

```js
import path from "node:path";

app.use("/static", express.static("public", {
  maxAge: "1y",
  immutable: true,
  fallthrough: true,
}));
```

### Subir a filesystem

```js
import fs from "node:fs/promises";
import path from "node:path";

async function saveToDisk(buffer, name) {
  const fileName = `${Date.now()}-${name}`;
  await fs.mkdir("uploads", { recursive: true });
  await fs.writeFile(path.join("uploads", fileName), buffer);
  return fileName;
}
```

### URL pública

```
http://mi-servidor.com/static/avatar.png
http://mi-servidor.com/api/files/123
```

### Desventajas

- **No escala** entre múltiples instancias (no hay compartida).
- **Backups** manuales.
- **Disco limitado**.
- **No hay CDN**.

### Cuándo sí

- Apps monolito, único servidor.
- Devs o demos.
- Archivos privados (PDFs generados, backups).

### Actividad práctica

- Implementa un endpoint que suba a `uploads/` y los sirva por una ruta protegida con auth.

---

## 6. S3 Compatible (AWS S3, MinIO, R2)

### Proveedores

| Servicio  | Notas                                  |
| --------- | -------------------------------------- |
| AWS S3    | El estándar, $ por GB + transfer       |
| MinIO     | Self-hosted, S3-compatible             |
| R2        | Sin egress (Cloudflare)                |
| Backblaze | Barato, S3-compatible                  |
| DigitalOcean Spaces | S3-compatible                  |
| Wasabi    | Sin egress fees hasta cierto límite    |

### Conceptos S3

- **Bucket**: contenedor.
- **Object**: archivo (key).
- **Key**: ruta (`users/42/avatar.webp`).
- **Versioning**: opcional.
- **ACL**: pública/privada.
- **CORS**: configurado en bucket.
- **Lifecycle rules**: archival.
- **Storage Classes**: Standard, IA, Glacier.

### SDK

```bash
npm install @aws-sdk/client-s3 @aws-sdk/lib-storage
```

### Cliente

```js
// src/lib/s3.js
import { S3Client } from "@aws-sdk/client-s3";

export const s3 = new S3Client({
  region: process.env.AWS_REGION,
  credentials: {
    accessKeyId: process.env.AWS_ACCESS_KEY,
    secretAccessKey: process.env.AWS_SECRET_KEY,
  },
});
```

### Subir archivo simple

```js
import { PutObjectCommand } from "@aws-sdk/client-s3";
import { randomUUID } from "node:crypto";

const key = `users/${req.user.sub}/${randomUUID()}.webp`;

await s3.send(new PutObjectCommand({
  Bucket: process.env.S3_BUCKET,
  Key: key,
  Body: buffer,
  ContentType: "image/webp",
  ACL: "public-read", // o privada + signed URLs
}));

const url = `https://${process.env.S3_BUCKET}.s3.${process.env.AWS_REGION}.amazonaws.com/${key}`;
```

### Subida con progreso (multipart)

```bash
npm install @aws-sdk/lib-storage
```

```js
import { Upload } from "@aws-sdk/lib-storage";

const upload = new Upload({
  client: s3,
  params: {
    Bucket: process.env.S3_BUCKET,
    Key: key,
    Body: buffer,
    ContentType: "image/webp",
  },
});

upload.on("httpUploadProgress", (e) => {
  console.log(`${e.loaded}/${e.total}`);
});

await upload.done();
```

### URLs prefirmadas (privadas)

```js
import { GetObjectCommand } from "@aws-sdk/client-s3";
import { getSignedUrl } from "@aws-sdk/s3-request-presigner";

const url = await getSignedUrl(
  s3,
  new GetObjectCommand({ Bucket, Key }),
  { expiresIn: 3600 }
);
```

### Upload directo desde el cliente

```js
// Backend: genera URL pre-firmada para PUT
import { PutObjectCommand } from "@aws-sdk/client-s3";

app.post("/api/upload/sign", authMiddleware, async (req, res) => {
  const key = `users/${req.user.sub}/${randomUUID()}.webp`;
  const cmd = new PutObjectCommand({
    Bucket: process.env.S3_BUCKET,
    Key: key,
    ContentType: req.body.contentType,
  });
  const url = await getSignedUrl(s3, cmd, { expiresIn: 60 });
  res.json({ url, key });
});

// Cliente: PUT directo
const { url } = await fetch("/api/upload/sign", { method: "POST", body: JSON.stringify({ contentType: "image/webp" }) }).then(r => r.json());

await fetch(url, { method: "PUT", body: file });
```

### MinIO self-hosted

```bash
docker run -p 9000:9000 -p 9001:9001 \
  -e MINIO_ROOT_USER=admin -e MINIO_ROOT_PASSWORD=secret \
  minio/minio server /data --console-address ":9001"
```

```js
const s3 = new S3Client({
  endpoint: "http://localhost:9000",
  region: "us-east-1",
  credentials: { accessKeyId: "admin", secretAccessKey: "secret" },
  forcePathStyle: true, // importante para MinIO
});
```

### Cloudflare R2

```js
new S3Client({
  region: "auto",
  endpoint: `https://${R2_ACCOUNT_ID}.r2.cloudflarestorage.com`,
  credentials: { accessKeyId, secretAccessKey },
});
```

### Multi-part upload

```bash
# CreateMultipartUpload → UploadPart* → CompleteMultipartUpload
```

Recomendado para archivos > 100MB.

### Presigned POST (subida con restricciones)

```js
import { createPresignedPost } from "@aws-sdk/s3-presigned-post";

const { url, fields } = await createPresignedPost(s3, {
  Bucket: process.env.S3_BUCKET,
  Key: key,
  Conditions: [
    ["content-length-range", 0, 5 * 1024 * 1024],
    ["eq", "$Content-Type", "image/webp"],
  ],
  Expires: 600,
});

res.json({ url, fields });
```

### Actividad práctica

- Sube una imagen desde el cliente directamente a S3 con una URL pre-firmada.

---

## 7. Cloud Storage (alternativas)

### Google Cloud Storage

```bash
npm install @google-cloud/storage
```

```js
import { Storage } from "@google-cloud/storage";

const storage = new Storage({ projectId: process.env.GCP_PROJECT });
const bucket   = storage.bucket(process.env.GCS_BUCKET);

const file = bucket.file(`avatars/${name}`);
await file.save(buffer, { contentType: "image/webp", resumable: false });

const url = `https://storage.googleapis.com/${bucket.name}/${file.name}`;
```

### Azure Blob Storage

```bash
npm install @azure/storage-blob
```

```js
import { BlobServiceClient } from "@azure/storage-blob";

const client = BlobServiceClient.fromConnectionString(process.env.AZURE_CONN);
const container = client.getContainerClient("media");
const blob = container.getBlockBlobClient(`users/${userId}/avatar.webp`);

await blob.uploadData(buffer, { blobHTTPHeaders: { blobContentType: "image/webp" } });
```

### Cuándo usar cada uno

- **S3 / R2 / Spaces**: mejor ecosistema, fácil de integrar.
- **GCS**: integración con BigQuery, Firebase.
- **Azure**: integración con Azure Functions.
- **MinIO**: self-hosted, control total.

### Actividad práctica

- Compara costos y throughput de S3 vs R2 vs MinIO para 1TB/mes.

---

## 8. CDN (Content Delivery Network)

Una red de servidores distribuidos geográficamente que entrega contenido **más cerca** del usuario.

### Beneficios

- Latencia reducida.
- Menor carga al origen.
- Tolerancia a picos.
- Mitigación de DDoS.
- Cache en borde.

### Proveedores

- **Cloudflare**: gratis para muchos casos, network global enorme.
- **CloudFront** (AWS): integración con S3.
- **Fastly**: usado por Vercel/Netlify, edge computing.
- **Vercel Edge Network** / **Netlify Edge**.
- **Bunny.net**: muy económico.
- **Akamai**: enterprise.

### Cloudflare

```
Origen: S3 / R2
    │
    ▼
Cloudflare edge (caching)
    │
    ▼
Usuario
```

Setup básico:
1. Crear cuenta y añadir dominio.
2. Cambiar DNS al proxy de Cloudflare (nube naranja).
3. Configurar cache rules:
   - `*.webp, *.jpg, *.png` → Edge cache TTL 1 año.
   - Resto: cache por defecto.

### `Cache-Control`

```js
res.set("Cache-Control", "public, max-age=31536000, immutable"); // archivos versionados
res.set("Cache-Control", "public, max-age=300");                 // 5 min
res.set("Cache-Control", "public, max-age=0, must-revalidate");  // siempre revalidar
```

### Versionado de assets

```
app.js?v=42     → cache forever
api/data        → cache 0
```

```js
// Generar en build
const v = process.env.npm_package_version;
`<script src="/app.js?v=${v}">`
```

### Firma de query para invalidar

```
?v=1234567890
```

### CloudFront con S3

```
S3 bucket privado
   │
   ▼ (OAI/OAC)
CloudFront distribution
   │
   ▼
S3 urls firmadas o APIs del backend
```

### URL firmada a través de CDN

```js
const cdnUrl = `https://d111111abcdef8.cloudfront.net/${key}?${signature}`;
```

### Cache por extensión

```js
// Express
express.static("public", {
  maxAge: "1y",
  immutable: true,
  setHeaders: (res, path) => {
    if (path.endsWith(".html")) {
      res.setHeader("Cache-Control", "no-cache");
    }
  },
});
```

### Invalidación de cache

```bash
# Cloudflare
curl -X POST "https://api.cloudflare.com/.../zones/{id}/purge_cache" \
  -H "Authorization: Bearer ..." \
  -d '{"files":["https://app.com/static/app.42.js"]}'
```

### Cloudflare Workers y R2

```js
addEventListener("fetch", (event) => {
  event.respondWith(handle(event.request));
});

async function handle(request) {
  const url = new URL(request.url);
  if (url.pathname.startsWith("/img/")) {
    const obj = await env.R2.get(url.pathname.slice(5));
    return new Response(obj.body, { headers: { "Content-Type": obj.httpMetadata.contentType } });
  }
  return fetch(request);
}
```

### Image Resizing en CDN

- **Cloudflare Image Resizing**: `?width=320&quality=80`.
- **CloudFront Functions/Lambda@Edge**.
- **imgproxy** self-hosted.
- **Cloudinary** (todo-en-uno + ML).

```html
<img src="https://mi-cdn.com/img/avatar.webp?w=320&q=80">
```

### Actividad práctica

- Configura un bucket de S3 + CloudFront, sube una imagen y verifica el cache desde `curl -I` y `Cache-Control`.

---

## Buenas prácticas generales

### Cliente

- Valida tamaño y tipo antes de subir.
- Comprime antes (WebP/AVIF).
- Genera nombres únicos (`UUID`) o usa la fecha.
- **No guardes secretos** en URLs públicas.

### Servidor

- Verifica MIME con magic numbers (`file-type`).
- Limita tamaño total y por archivo.
- Sanitiza el nombre (`path.basename`).
- Usa HTTPS con HSTS.
- Rate limit por usuario.

### Almacenamiento

- Estructura por usuario/entidad: `users/{id}/avatar.{webp}`.
- Versionado opcional (`{id}/v1.webp`).
- Lifecycle: borrar archivos huérfanos tras N días.
- Encriptación en reposo (S3 lo hace por defecto).

### CDN

- Cache `public, max-age=...` para assets con hash.
- `Cache-Control: no-cache` para HTML.
- Headers `Vary` correctos (Accept-Encoding, Origin).
- HTTPS end-to-end.

---

## Errores comunes

- Subir al backend y guardar en disco → no escala.
- Confiar en `Content-Type` del cliente.
- Permitir uploads de HTML/SVG (XSS).
- Nombres predecibles (`avatar.png`) → colisiones.
- Olvidar `Content-Disposition: attachment` para descargas.
- No comprimir ni redimensionar → archivos enormes.
- Imágenes sin `width`/`height` → layout shift.

---

## Proyecto integrador

```
backend/
├── package.json
├── .env                  AWS_*, S3_BUCKET, CDN_URL
└── src/
    ├── config/s3.js
    ├── lib/
    │   ├── image.js      (sharp)
    │   ├── upload.js     (multipart, validación, signed POST)
    │   └── cdn.js        (URLs)
    ├── routes/upload.routes.js
    └── controllers/upload.controller.js
```

### Endpoint: pedir URL prefirmada

```js
// routes/upload.routes.js
router.post("/upload/sign", authMiddleware, uploadController.sign);

import { createPresignedPost } from "@aws-sdk/s3-presigned-post";
import { randomUUID } from "node:crypto";
import { s3 } from "../config/s3.js";

export async function sign(req, res) {
  const ext = (req.body.ext || "webp").toLowerCase();
  const key = `users/${req.user.sub}/${randomUUID()}.${ext}`;

  const { url, fields } = await createPresignedPost(s3, {
    Bucket: process.env.S3_BUCKET,
    Key: key,
    Conditions: [
      ["content-length-range", 0, 5 * 1024 * 1024],
      ["eq", "$Content-Type", "image/webp"],
    ],
    Expires: 60,
  });

  res.json({
    uploadUrl: url,
    fields,
    publicUrl: `${process.env.CDN_URL}/${key}`,
    key,
  });
}
```

### Cliente (frontend)

```jsx
// components/UploadAvatar.jsx
import { useState } from "react";
import { compressImage, getMimeType } from "../lib/image";

const CDN = import.meta.env.VITE_CDN_URL;

export default function UploadAvatar({ token, onUploaded }) {
  const [progress, setProgress] = useState(0);

  async function handleFile(file) {
    const compressed = await compressImage(file, 1280, 0.8);
    const signature = await fetch("/api/upload/sign", {
      method: "POST",
      headers: { "Content-Type": "application/json", Authorization: `Bearer ${token}` },
      body: JSON.stringify({ ext: "webp" }),
    }).then(r => r.json());

    const fd = new FormData();
    Object.entries(signature.fields).forEach(([k, v]) => fd.append(k, v));
    fd.append("file", compressed);

    await uploadWithProgress(signature.uploadUrl, fd, setProgress);
    onUploaded(signature.publicUrl);
  }

  return (
    <div>
      <input type="file" accept="image/*" onChange={(e) => handleFile(e.target.files[0])} />
      {progress > 0 && <progress value={progress} max="1">{Math.round(progress * 100)}%</progress>}
    </div>
  );
}

function uploadWithProgress(url, body, onProgress) {
  return new Promise((resolve, reject) => {
    const xhr = new XMLHttpRequest();
    xhr.open("POST", url);
    xhr.upload.onprogress = (e) => e.lengthComputable && onProgress(e.loaded / e.total);
    xhr.onload = () => (xhr.status < 300 ? resolve(xhr.response) : reject(xhr.response));
    xhr.onerror = reject;
    xhr.send(body);
  });
}
```

### Backend: servir desde CDN

```js
function publicUrl(key) {
  return `${process.env.CDN_URL}/${key}`;
}

export async function remove(key) {
  await s3.send(new DeleteObjectCommand({ Bucket: process.env.S3_BUCKET, Key: key }));
}
```

---

## Recursos complementarios

### Lecturas

- [MDN - FileReader](https://developer.mozilla.org/es/docs/Web/API/FileReader)
- [MDN - FormData](https://developer.mozilla.org/es/docs/Web/API/FormData)
- [AWS S3 Docs](https://docs.aws.amazon.com/s3/)
- [Sharp Docs](https://sharp.pixelplumbing.com/)
- [Cloudflare Caching](https://developers.cloudflare.com/cache/)
- [web.dev - Image optimization](https://web.dev/fast/#optimize-your-images)

### Herramientas

- **AWS CLI**, **s3cmd** para gestionar buckets.
- **Cloudflare** dashboard.
- **imgproxy**, **Thumbor**, **Cloudinary** para transformaciones server-side.
- **curl**, **wget** para pruebas.

### Comparativas

- [S3 vs R2 vs Spaces](https://blog.cloudflare.com/r2-vs-s3/)

---

## Evaluación sugerida

1. **Upload cliente (20%)**: drag & drop + barra de progreso + validación previa.
2. **Procesamiento (20%)**: Sharp con múltiples tamaños en WebP/AVIF.
3. **S3 (30%)**: signed POST cliente → S3, URLs públicas vía CDN.
4. **CDN (20%)**: configurar caching por tipo de archivo y versionado.
5. **Seguridad (10%)**: validar MIME real, tamaño, signed URLs privadas.

---

> **Nota para el estudiante**: en producción, **nunca dejes archivos en el servidor**. Subir a S3 y servir por CDN es el patrón estándar. La compresión en el cliente reduce ancho de banda y tiempo de espera antes de la primera visualización.
