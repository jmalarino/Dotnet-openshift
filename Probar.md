Para reemplazar el contenido de `appsettings.json` en tu ConfigMap usando `yq`, primero necesitas asegurarte de tener `yq` instalado. `yq` es una herramienta de línea de comandos para procesar archivos YAML y JSON.

Supongamos que tienes dos archivos:
- `configmap.yaml` que contiene tu ConfigMap actual.
- `new-appsettings.json` que contiene el contenido actualizado de `appsettings.json`.

Aquí te explico cómo puedes hacer esto:

1. **Instalar yq**:
   Si no tienes `yq` instalado, puedes instalarlo con `pip` (si estás usando `yq` versión 4.x):

   ```sh
   pip install yq
   ```

   O puedes instalar la versión autónoma usando `wget`:

   ```sh
   wget https://github.com/mikefarah/yq/releases/download/v4.6.0/yq_linux_amd64 -O /usr/bin/yq && chmod +x /usr/bin/yq
   ```

2. **Reemplazar el contenido**:
   Utiliza el siguiente comando para actualizar el contenido de `appsettings.json` dentro de `configmap.yaml`:

   ```sh
   yq eval '.data["appsettings.json"] = load("new-appsettings.json")' configmap.yaml > updated-configmap.yaml
   ```

   Este comando hace lo siguiente:
   - `yq eval`: Ejecuta una evaluación de yq.
   - `.data["appsettings.json"] = load("new-appsettings.json")`: Carga el contenido de `new-appsettings.json` y lo asigna a `appsettings.json` dentro del `ConfigMap`.
   - `configmap.yaml > updated-configmap.yaml`: Escribe el resultado en un nuevo archivo llamado `updated-configmap.yaml`.

### Ejemplo Paso a Paso:

1. **Contenido original de `configmap.yaml`**:

   ```yaml
   kind: ConfigMap
   apiVersion: v1
   metadata:
     name: garantias-monitor-config
     namespace: dev-garantias-monitor
     immutable: false
   data:
     appsettings.json: |-
       {
         "Logging": {
           "LogLevel": {
             "Default": "Information",
             "Microsoft.AspNetCore": "Warning"
           }
         },
         ...
       }
   ```

2. **Contenido de `new-appsettings.json`** (actualizado):

   ```json
   {
     "Logging": {
       "LogLevel": {
         "Default": "Warning",
         "Microsoft.AspNetCore": "Error"
       }
     },
     "Paths": {
       "Conciliacion": "D:\\New Path\\Conciliacion\\",
       "UploadGarantias": "D:\\New Path\\Garantias\\"
     },
     "AppEnvironment": "PROD",
     "ValidIssuerToken": "NewIssuerToken",
     ...
   }
   ```

3. **Comando para actualizar**:

   ```sh
   yq eval '.data["appsettings.json"] = load("new-appsettings.json")' configmap.yaml > updated-configmap.yaml
   ```

4. **Contenido actualizado de `updated-configmap.yaml`**:

   ```yaml
   kind: ConfigMap
   apiVersion: v1
   metadata:
     name: garantias-monitor-config
     namespace: dev-garantias-monitor
     immutable: false
   data:
     appsettings.json: |-
       {
         "Logging": {
           "LogLevel": {
             "Default": "Warning",
             "Microsoft.AspNetCore": "Error"
           }
         },
         "Paths": {
           "Conciliacion": "D:\\New Path\\Conciliacion\\",
           "UploadGarantias": "D:\\New Path\\Garantias\\"
         },
         "AppEnvironment": "PROD",
         "ValidIssuerToken": "NewIssuerToken",
         ...
       }
   ```

Ahora, `updated-configmap.yaml` contiene el contenido de `appsettings.json` actualizado desde `new-appsettings.json`. Puedes aplicar este ConfigMap actualizado a tu clúster de Kubernetes usando:

```sh
kubectl apply -f updated-configmap.yaml
```

Esto asegurará que tu ConfigMap tenga la configuración más reciente.
