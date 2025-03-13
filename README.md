# Guía de Diagnóstico de Red con Netshoot en Kubernetes

Esta guía muestra paso a paso cómo utilizar Netshoot para diagnosticar problemas de red en un clúster de Kubernetes. Los ejemplos utilizan una aplicación simple compuesta por un servidor web, una base de datos y un servicio de caché.

## Preparación del Entorno

### 1. Aplicar el Manifiesto de la Aplicación Simple

Primero, aplica el manifiesto para desplegar la aplicación de prueba:

```bash
kubectl apply -f simple-app.yaml
```

### 2. Crear Pod de Netshoot para Diagnóstico

```bash
kubectl run -n simple-app netshoot-diagnostic --image nicolaka/netshoot -- sleep infinity
```

Espera a que el pod esté listo:

```bash
kubectl wait --for=condition=Ready pod/netshoot-diagnostic -n simple-app
```

## Escenario 1: Verificar Problemas de Conectividad entre Servicios

### Verificar conectividad al servidor web

```bash
# Intentar conexión HTTP al servidor web
kubectl exec -n simple-app netshoot-diagnostic -- curl -s web-server
```

Si no hay respuesta, realiza estos pasos de diagnóstico:

```bash
# Comprobar si el servicio existe
kubectl get svc web-server -n simple-app

# Comprobar endpoints
kubectl get endpoints web-server -n simple-app

# Comprobar resolución DNS
kubectl exec -n simple-app netshoot-diagnostic -- dig web-server.simple-app.svc.cluster.local
```

### Verificar conectividad a la base de datos

```bash
# Comprobar conectividad al puerto MySQL
kubectl exec -n simple-app netshoot-diagnostic -- nc -z -v -w2 database 3306
```

Si falla, verifica:

```bash
# Comprobar servicio y pods
kubectl get svc database -n simple-app
kubectl get pods -n simple-app -l app=database
```

### Verificar conectividad al caché

```bash
# Comprobar conectividad a Redis
kubectl exec -n simple-app netshoot-diagnostic -- nc -z -v -w2 cache 6379
```

## Escenario 2: Diagnóstico de Problemas de Latencia

### Medir latencia de red básica

```bash
# Medir latencia con ping
kubectl exec -n simple-app netshoot-diagnostic -- ping -c 5 web-server
```

### Medir tiempos de respuesta HTTP

```bash
# Analizar tiempos de respuesta detallados
kubectl exec -n simple-app netshoot-diagnostic -- curl -o /dev/null -s -w "Tiempo de conexión: %{time_connect}s\nTiempo hasta primer byte: %{time_starttransfer}s\nTiempo total: %{time_total}s\n" web-server
```

Interpretación:
- Si el tiempo de conexión es alto: Posible problema de red entre pods
- Si el tiempo hasta primer byte es alto: Posible problema de rendimiento en la aplicación
- Si el tiempo total es alto pero los otros tiempos son bajos: Posible problema de transferencia

## Escenario 3: Diagnóstico de Problemas de DNS

### Verificar configuración DNS

```bash
# Revisar configuración DNS del pod
kubectl exec -n simple-app netshoot-diagnostic -- cat /etc/resolv.conf
```

### Probar resolución DNS de servicios internos

```bash
# Resolución de nombre corto
kubectl exec -n simple-app netshoot-diagnostic -- dig web-server.simple-app.svc.cluster.local +short

# Resolución de FQDN
kubectl exec -n simple-app netshoot-diagnostic -- dig database.simple-app.svc.cluster.local +short
```

### Medir tiempos de respuesta DNS

```bash
# Medir tiempo de consulta DNS
kubectl exec -n simple-app netshoot-diagnostic -- time dig web-server.simple-app.svc.cluster.local
```

Si las consultas DNS son lentas, verifica CoreDNS:

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
```

## Escenario 4: Diagnóstico de Problemas de MTU y Fragmentación

### Verificar MTU de interfaces

```bash
# Ver MTU configurado en interfaces
kubectl exec -n simple-app netshoot-diagnostic -- ip link | grep mtu
```

### Probar diferentes tamaños de paquete

```bash
# Probar con 1400 bytes (generalmente funciona bien)
kubectl exec -n simple-app netshoot-diagnostic -- ping -c 1 -M do -s 1400 web-server

# Probar con 1450 bytes
kubectl exec -n simple-app netshoot-diagnostic -- ping -c 1 -M do -s 1450 web-server

# Probar con 1472 bytes (cerca del límite común)
kubectl exec -n simple-app netshoot-diagnostic -- ping -c 1 -M do -s 1472 web-server

# Probar con 1473 bytes (puede fallar en algunas redes)
kubectl exec -n simple-app netshoot-diagnostic -- ping -c 1 -M do -s 1473 web-server
```

Si un tamaño falla pero tamaños menores funcionan, has identificado el límite de MTU efectivo.

## Escenario 5: Análisis de Tráfico con tcpdump

Estos comandos deben ejecutarse en terminales separadas para análisis en tiempo real:

### Capturar tráfico HTTP

```bash
kubectl exec -it -n simple-app netshoot-diagnostic -- tcpdump -i any -n port 80
```

### Capturar tráfico de base de datos

```bash
kubectl exec -it -n simple-app netshoot-diagnostic -- tcpdump -i any -n port 3306
```

### Capturar tráfico de un pod específico

```bash
# Obtener IP del pod
POD_IP=$(kubectl get pod web-server-xxxxxx -n simple-app -o jsonpath='{.status.podIP}')

# Capturar tráfico
kubectl exec -it -n simple-app netshoot-diagnostic -- tcpdump -i any -n host $POD_IP
```

## Escenario 6: Analizar Headers HTTP

```bash
# Capturar headers HTTP
kubectl exec -n simple-app netshoot-diagnostic -- curl -s -D - web-server -o /dev/null
```

## Escenario 7: Monitoreo de Rendimiento en Tiempo Real

Estos comandos requieren una terminal interactiva:

### Monitorear ancho de banda

```bash
kubectl exec -it -n simple-app netshoot-diagnostic -- iftop
```

### Monitorear conexiones activas

```bash
kubectl exec -it -n simple-app netshoot-diagnostic -- watch -n 1 "netstat -tuln"
```

### Monitorear sockets en espera de conexión

```bash
kubectl exec -it -n simple-app netshoot-diagnostic -- watch -n 1 "ss -ltn"
```

## Limpieza

Cuando hayas terminado, puedes eliminar los recursos:

```bash
# Eliminar el pod de diagnóstico
kubectl delete pod netshoot-diagnostic -n simple-app

# Eliminar la aplicación completa (opcional)
kubectl delete namespace simple-app
```


