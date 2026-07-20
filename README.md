# Pre-Compiled Docker Base Images (apigo)

Repositori ini berisi blueprint Dockerfile untuk membuat base image **Builder (Compile)** dan **Runtime (Final)** yang dioptimalkan untuk project Golang.

## Mengapa Menggunakan Pre-Compiled Image?
1. **Bebas Timeout Build**: Paket compiler seperti `gcc` dan dependency `swag` sudah di-bake langsung ke dalam base image. Pipeline CI/CD tidak perlu melakukan download berulang-ulang dari mirror eksternal.
2. **Setup Timezone Global**: Mengatur zona waktu default secara konsisten ke `Asia/Jakarta` (WIB) baik pada tahap build maupun runtime aplikasi.
3. **Lebih Cepat**: Menghemat waktu build pipeline CI/CD rata-rata 3-5 menit per build.

---

## 1. Dockerfile.base (Builder Image)
Digunakan di stage pertama (`AS builder`) untuk memproses CGO, kompilasi binary, dan generasi dokumen Swagger.

* **Base OS**: `golang:1.26-alpine3.23`
* **Fitur**: Pre-installed `gcc`, `libc-dev`, `make`, `git`, `curl`, `swag CLI v1.16.4`, dan Timezone `Asia/Jakarta`.

### Cara Build & Push:
```bash
docker build -f Dockerfile.base -t <registry>/<image>:<tag> .
docker push <registry>/<image>:<tag>
```

---

## 2. Dockerfile.runtime (Runtime Image)
Digunakan di stage akhir (`AS final`) untuk menjalankan Go binary aplikasi.

* **Base OS**: `alpine:3.21`
* **Fitur**: Pre-installed `ca-certificates`, `libc6-compat` (untuk program Go), dan Timezone `Asia/Jakarta`.

### Cara Build & Push:
```bash
docker build -f Dockerfile.runtime -t <registry>/<image>:<tag> .
docker push <registry>/<image>:<tag>
```

---

## Cara Implementasi di Dockerfile Project (Contoh)
Ubah Dockerfile di project apigo kamu menjadi seperti di bawah ini:

```dockerfile
# ==========================================
# Stage 1: Build (Sudah include gcc, swag, tz)
# ==========================================
FROM <registry>/<image>:<tag> AS builder

WORKDIR /projects/
COPY go.mod go.sum ./
RUN go mod download

COPY . .

# ==========================================
# Stage 2: Runtime (Sudah include tz, certs)
# ==========================================
FROM <registry>/<image>:<tag> AS final

# (Jika ada dependencies tambahan per project, contoh: LibreOffice)
RUN apk add --no-cache openjdk11-jre libreoffice qpdf python3 py3-pip && \
    pip3 install --no-cache-dir unoserver --break-system-packages

COPY --from=builder /projects/config-*.json /
COPY --from=builder /projects/build/ /
COPY --from=builder /projects/export/ /export
COPY --from=builder /projects/logs/ /logs
COPY --from=builder /projects/temp/ /temp
COPY entrypoint.sh /entrypoint.sh

RUN chmod +x /main /entrypoint.sh && \
    chmod -R 777 /logs /temp /export

ENTRYPOINT ["/entrypoint.sh"]
EXPOSE 6060
```
