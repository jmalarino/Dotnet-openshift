# Dotnet-openshift

Para buscar y utilizar un certificado específico por su thumbprint, después de cargarlo desde un archivo PFX que se ha montado en un contenedor de OpenShift.

### 1. Crear y montar el Secret en OpenShift

Primero, crear un Secret en OpenShift que contenga tu archivo .pfx:

```bash
oc create secret generic my-pfx-secret --from-file=certificado.pfx=/ruta/al/certificado.pfx
```

Luego, actualizar tu Deployment para montar el Secret en el contenedor:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-container
        image: my-app-image:latest
        volumeMounts:
        - name: pfx-secret
          mountPath: /etc/ssl/certs
          readOnly: true
      volumes:
      - name: pfx-secret
        secret:
          secretName: my-pfx-secret
```

### 2. Configurar la aplicación .NET para buscar el certificado por su thumbprint

Modifica tu aplicación .NET para cargar el archivo PFX, buscar el certificado específico por su thumbprint y luego usarlo para configurar el `HttpClient`.

#### Ejemplo de configuración en .NET:

```csharp
using System;
using System.Linq;
using System.Net.Http;
using System.Security.Cryptography.X509Certificates;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;

var builder = WebApplication.CreateBuilder(args);

// Ruta al certificado y contraseña (si aplica)
var certPath = "/etc/ssl/certs/certificado.pfx";
var certPassword = "tu_contraseña";

// Thumbprint del certificado que quieres usar
var certThumbprint = "YOUR_CERT_THUMBPRINT";

// Cargar el certificado desde el archivo PFX
var certCollection = new X509Certificate2Collection();
certCollection.Import(certPath, certPassword, X509KeyStorageFlags.DefaultKeySet);

// Buscar el certificado por thumbprint
var clientCertificate = certCollection
    .OfType<X509Certificate2>()
    .FirstOrDefault(cert => cert.Thumbprint.Equals(certThumbprint, StringComparison.OrdinalIgnoreCase));

if (clientCertificate == null)
{
    throw new Exception("Certificado no encontrado.");
}

// Configurar HttpClient para usar el certificado
builder.Services.AddHttpClient("ClientWithCert")
                .ConfigurePrimaryHttpMessageHandler(() =>
                {
                    var handler = new HttpClientHandler();
                    handler.ClientCertificates.Add(clientCertificate);
                    handler.ServerCertificateCustomValidationCallback = HttpClientHandler.DangerousAcceptAnyServerCertificateValidator; // Ajusta según tus necesidades de validación
                    return handler;
                });

var app = builder.Build();

app.MapGet("/", () => "Hello World!");

app.Run();
```

### 3. Uso del `HttpClient` configurado con el certificado

Para realizar solicitudes HTTP usando el cliente configurado con el certificado, crea un servicio que utilice `IHttpClientFactory`.

#### Ejemplo de servicio:

```csharp
public class MyService
{
    private readonly IHttpClientFactory _httpClientFactory;

    public MyService(IHttpClientFactory httpClientFactory)
    {
        _httpClientFactory = httpClientFactory;
    }

    public async Task<string> GetDataAsync()
    {
        var client = _httpClientFactory.CreateClient("ClientWithCert");
        var response = await client.GetAsync("https://api.endpoint/protected/resource");
        response.EnsureSuccessStatusCode();

        return await response.Content.ReadAsStringAsync();
    }
}
```

### 4. Desplegar y ejecutar la aplicación en OpenShift

Aplica el archivo de Deployment YAML:

```bash
oc apply -f deployment.yaml
```

### Nota sobre la validación de certificados del servidor

En el código, se usa `HttpClientHandler.DangerousAcceptAnyServerCertificateValidator` para aceptar cualquier certificado del servidor, lo cual no es seguro en un entorno de producción. Implementa una validación adecuada del certificado del servidor según tus necesidades de seguridad:

```csharp
handler.ServerCertificateCustomValidationCallback = (sender, cert, chain, sslPolicyErrors) =>
{
    return sslPolicyErrors == SslPolicyErrors.None;
};
```

### Resumen

Este enfoque permite montar un archivo .pfx desde un Secret en OpenShift, cargarlo en tu aplicación .NET y buscar el certificado correcto por su thumbprint para usarlo en las solicitudes HTTP. Esta metodología es útil cuando el archivo PFX contiene múltiples certificados y necesitas asegurarte de que estás usando el correcto.
