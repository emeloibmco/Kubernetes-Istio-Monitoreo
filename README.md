# Demostración de la aplicabilidad de Istio en un Cluster K8s ☁

**Disclaimer**: Este repositorio contiene una copia de Istio, todos los archivos han sido descargados del [GitHub Oficial de Istio](https://github.com/istio/istio/releases) y pertenecen a sus respectivos autores.

Esta demo es un acercamiento al concepto de Service Mesh en un Cluster de K8s provisto en IBM Cloud usando Istio y el Dashboard Kiali.

Usaremos Istio para administrar configuraciones al Load Balancer, crear rutas entre servicios, realizar transiciones ágiles entre versiones de un servicio y visualizar nuestro Service Mesh con Kiali.
<br />

## 📑 Tabla de contenido

1. [Requisitos](#-requisitos)
2. [Hands On!](#-hands-on)
   2.1 [Configuración de Istio en IKS](#Configuración-de-Istio-en-IKS)
   2.2 [Despliegue de la aplicación](#Despliegue-de-la-aplicación)
   2.3 [Dashboard Kiali](#Dashboard-Kiali)
   2.4 [Despliegue de servicio de base de datos MongoDB](#Despliegue-de-servicio-de-base-de-datos-MongoDB)
3. [Referencias y documentación útil](#Referencias-y-documentación-útil)
<br />

## 📑 Requisitos

- Tener un servicio **[Kubernetes Cluster (IKS)](https://cloud.ibm.com/kubernetes/clusters)** disponible en la cuenta IBM Cloud.

  **Importante:** Debe ser un Cluster **pago** en plan **Standard**

- :cloud: [IBM Cloud CLI](https://cloud.ibm.com/docs/cli?topic=cloud-cli-getting-started&locale=en)
- :whale: [Docker](https://www.docker.com/products/docker-desktop)
- [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/). La version de esta herramienta debe ser compatible con la version de IKS que se desplegó en la cuenta.
- Complemento [container-service/kubernetes-service](https://cloud.ibm.com/docs/cli?topic=cli-install-devtools-manually) para ibmcloud CLI. `ibmcloud plugin install container-service/kubernetes-service`
<br />

## ✋ Hands On!

### Configuración de Istio en IKS

**Paso 1:** Clonar este repositorio y configurar las variables de entorno de nuestro ambiente.

Nos ubicamos en la carpeta del repositorio y colocamos: 

Linux o OSX: `export PATH=$PWD/bin:$PATH`

Windows: Escribimos el siguiente comando en PowerShell

```powershell
$path = [Environment]::GetEnvironmentVariable('PATH', 'User')
$ruta = $PWD
$newpath = $path + $ruta +'\bin'
[Environment]::SetEnvironmentVariable("PATH", $newpath, 'User')
```
<br />

**Paso 2:** Configuración de nuestro Cluster IKS

Recuerde llenar el campo <nombre_cluster> con el nombre de su cluster

`ibmcloud cs cluster config --cluster <nombre_cluster>`
<br />

**Paso 3:** Instalar Istio en nuestro cluster

Para efectos de esta demo definimos el perfil demo incluido en el repositorio

Usando el comando `istioctl install --set profile=demo` se instalará y configurará Istio en nuestro Cluster.

<p align=center><img src=".github/istioctl-install.png"></p>
<br />

**Paso 4:** Habilitar la inyección automática de Istio al Envoy Sidecar de nuestro cluster

Esto se realiza para un namespace determinado, en este caso usaremos el namespace por defecto

`kubectl label namespace default istio-injection=enabled`

<p align=center><img src=".github/istioctl-injection.png"></p>
<br />

### Despliegue de la aplicación

**Paso 1: aplicación bookinfo**

Vamos a desplegar la aplicación de ejemplo Bookinfo que está en la carpeta samples del repositorio, usando el comando:

`kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml`

Este comando nos creará todo el despliegue en nuestro cluster, es decir, Deployment, Service, Pods y réplicas.

Podemos comprobar la aplicación del comando anterior visualizando los servicios y los pods de nuestro cluster:

`kubectl get services`

<p align=center><img src=".github/istioctl-services.png"></p>

`kubectl get pods`

<p align=center><img src=".github/istioctl-pods.png"></p>
<br />

**Paso 2: Exponer al exterior de nuestro cluster y definición de políticas de acceso**

Ahora configuramos nuestra aplicación para aceptar trafico externo, agregando el Istio Ingress Gateway que se encargará de gestionar las rutas de nuestro Service Mesh.
Por defecto, el ingress gateway se encarga de bloquear todas las solicitudes, permitiendo únicamente las que definamos en las políticas de acceso.

`kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml`

<p align=center><img src=".github/istioctl-ingress.png"></p>

Definimos tambien la confugiración de enrutamiento, donde se especifica a que servicios se puede acceder desde el exterior, aplicando el archivo destination-rule-all.yaml

`kubectl apply -f samples/bookinfo/networking/destination-rule-all.yaml`

<p align=center><img src=".github/istioctl-routes.png"></p>
<br />

Para obtener la dirección ip y el puerto(externo) de nuestra aplicación ejecutamos los siguientes comandos:

**Dirección IP:**

`kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}'`

<p align=center><img src=".github/istioctl-ip.png"></p>
<br />

**Puerto:**

Terminal de Linux & OSX

`kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{$.spec.ports[?(@.name=="http2")].nodePort}'`

PowerShell:

`kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{$.spec.ports[?(@.name==\"http2\")].nodePort}'`

<p align=center><img src=".github/istioctl-port.png"></p>
<br />

Ahora verificamos que sea posible acceder mediante el comando:

`curl -o /dev/null -s -w "%{http_code}\n" http://169.63.6.234/productpage`

La salida debe ser 200

<p align=center><img src=".github/istioctl-status.png">

Tambien por el navegador accediendo a la dirección `http://169.64.6.235/productpage`

<p align=center><img src=".github/istioctl-web.png"></p>
<br />

## Dashboard Kiali

Istio viene por defecto con Kiali y podemos visualizar el Service Mesh utilizando el comando

`istioctl dashboard kiali`

Las credenciales para acceder, tanto usuario como contraseña es **admin**

<p align=center><img src=".github/istioctl-login.png"></p>
<br />

### Captura de datos en Kiali

Seleccionamos en el panel izquierdo Graph y filtramos por nuestro namespace, en este caso Default, sin embargo, no hemos generado solicitudes a nuestra aplicación y por eso nos mostrará **Empty Graph**

Para generar una cantidad considerable de solicitudes, y así poder visualizar el tráfico en nuestro Service Mesh, usar el comando:

**Windows PowerShell:**

```powershell
$i = 1
do
{
   $Response = Invoke-WebRequest -URI http://169.63.6.234/productpage
   $Response.StatusCode
   $i++
}
while ($i -le 10)
```

**Linux & OSX:**

```bash
for ((i = 0; i < 10; i++)); do
    curl -o /dev/null -s -w "%{http_code}\n" http://169.63.6.234/productpage
done
```

En el panel lateral izquierdo seleccionamos Graph, en la pestaña Display, sección Show Edge Labels, seleccionamos Request Percentage y en la sección show Traffic Animation.

<p align=center><img src=".github/kiali-graph.png"></p>
<br />

## Despliegue de servicio de base de datos MongoDB

Ejecutamos el comando para desplegar el servicio:

`kubectl apply -f samples/bookinfo/platform/kube/bookinfo-db.yaml`

Comprobamos que se haya creado un nuevo servicio de mongodb, con el comando:

`kubectl get services`

Despluegamos una nueva version del servicio de ratings que consume nuestro servicio de mongodb

`kubectl apply -f samples/bookinfo/platform/kube/bookinfo-ratings-v2.yaml`

Para poder visualizar en Kiali las versiones, seleccionamos la drop list que se encuentra al lado derecho del Namespace y en Versioned app graph:

<p align=center><img src=".github/kiali-mongo.png"></p>
<br />

### Definición de políticas de acceso a nuestra base de datos

Pero si vamos a la página nos mostrará un error en la sección de reviews. Tenemos que definir nuevas políticas de acceso por medio del enrutamiento del Ingress Gateway, a la nueva versión del servicio ratings y al servicio mongodb

```bash
kubectl apply -f samples/bookinfo/networking/destination-rule-all-mtls.yaml
kubectl apply -f samples/bookinfo/networking/virtual-service-ratings-db.yaml
```

Tenemos que volver a realizar peticiones a nuestra página web con el fin de recibir tráfico en Kiali [Captura de datos en Kiali](#captura-de-datos-en-kiali).

Finalmente Kiali mostrará tráfico entrante a nuestro servicio de mongodb.

<p align=center><img src=".github/kiali-final.png"></p>
<br />

## Referencias y documentación útil

- [Documentación Kiali](https://istio.io/docs/tasks/observability/kiali/)
- [Documentación Inicial Istio](https://istio.io/docs/setup/getting-started/#install)

- [IBM Cloud Docs Istio](https://cloud.ibm.com/docs/containers?topic=containers-istio-qs)

- [Manejo de Políticas con Istio](https://istio.io/docs/tasks/policy-enforcement/denial-and-list/)

- [Autorización de servicios TCP Istio](https://archive.istio.io/v1.3/docs/tasks/security/authz-tcp/)
<br />

## Autores ✒
Equipo *IBM Cloud Tech Sales Colombia*.
