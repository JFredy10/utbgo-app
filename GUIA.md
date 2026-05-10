# 📖 Guía de Configuración y Despliegue — UTBGo

Esta guía explica cómo poner en marcha el ecosistema de UTBGo en dos escenarios: **Local (con Docker)** y **Producción (en Azure)**.

---

## 🛠️ Escenario 1: Ejecución Local (Sandbox)
Ideal para desarrolladores que solo quieren probar la lógica, el diseño y las funcionalidades base sin usar recursos de la nube.

### 1. Preparar el Entorno
Crea un archivo `.env` dentro de la carpeta `api-service/`. Puedes usar estos valores genéricos:

```bash
# Servidor y Entorno
PORT=8080
GIN_MODE=debug # 👈 CAMBIAR A 'debug' PARA DESARROLLO

# Base de Datos y Caché (Docker se encarga)
DB_CONNECTION_STRING="host=db user=postgres password=postgres dbname=utbgo port=5432 sslmode=disable"
REDIS_URL="redis:6379"

# Seguridad
JWT_SECRET_KEY="clave_local_muy_secreta"

# Microservicios (Nombres de red de Docker)
RECOMMENDATIONS_SERVICE_URL="http://utbgo-recommendations:8000"
TRACKING_SERVICE_URL="http://utbgo-tracking:8000"
RECOMMENDATIONS_API_KEY="test_key"
TRACKING_API_KEY="test_key"
```

### 2. Google Services
Para que la app de Flutter no de errores al iniciar, asegúrate de colocar tu archivo `google-services.json` (Android) y `GoogleService-Info.plist` (iOS) en las carpetas correspondientes:
*   `android/app/google-services.json`
*   `ios/Runner/GoogleService-Info.plist`

### 3. Levantar con Docker
Desde la raíz del proyecto, ejecuta el siguiente comando. Esto construirá y levantará la API de Go y los 3 microservicios de Python automáticamente:

```bash
docker-compose up -d --build
```

### 4. Lanzar Flutter (Comando Local)
Para correr la app sin IDs reales de Azure o Google, usa este comando simplificado:
```bash
flutter run --dart-define=AZURE_CLIENT_ID=null --dart-define=AZURE_TENANT_ID=common --dart-define=GOOGLE_CLIENT_ID=null
```

---

## 🚀 Escenario 2: Despliegue a Producción (Azure)
UTBGo utiliza un flujo de **DevSecOps** automatizado mediante **GitHub Actions**.

### 1. Configurar Secretos en GitHub
Para que el pipeline funcione, debes ir a **Settings > Secrets and variables > Actions** en tu repositorio y agregar:

| Secreto | Descripción | Valor Ejemplo |
| :--- | :--- | :--- |
| `AZURE_AD_CLIENT_ID` | ID del Service Principal (Admin) | `59f582b9-...` |
| `AZURE_AD_TENANT_ID` | ID del Directorio de Azure | `27a3436b-...` |
| `AZURE_AD_CLIENT_SECRET` | Secreto del Service Principal | `****` |
| `AZURE_SUBSCRIPTION_ID` | ID de tu suscripción de Azure | `...` |
| `TF_VAR_DB_CONNECTION_STRING` | URL de tu BD en la nube | `postgres://...` |

### 2. Flujo del Pipeline
Una vez configurados los secretos, el flujo es automático al hacer `git push origin main`:
1.  **Fase CI:** Ejecuta tests de Go, Python y Flutter + Escaneo de seguridad CodeQL.
2.  **Fase CD:** 
    *   Se loguea en Azure.
    *   Construye las imágenes Docker y las sube al **Azure Container Registry (ACR)**.
    *   Aplica **Terraform** para actualizar la infraestructura.
    *   Despliega las nuevas versiones en **Azure Container Apps**.

### 3. Cambio de Entorno en el Código
Antes de desplegar a producción, asegúrate de que en `api-service/main.go` o en tus variables de entorno, el `GIN_MODE` esté en **`release`** para activar las optimizaciones de rendimiento y seguridad.

---

**Nota:** Si vas a usar el Login de Microsoft en producción, el `AZURE_CLIENT_ID` que usa la App (definido en el pipeline) debe tener configurado el Redirect URI como `msauth://com.example.flutter_practica/callback` en el Portal de Azure.
