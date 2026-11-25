# Pre-entrega 7: Plan de Monitoreo y Auditoría (AWS)

Curso: Cloud Computing (AWS) - Coderhouse
Alumno: Federico Almiron

Plan para vigilar la red que armé en la Pre-entrega 6: un dashboard, 10 alarmas que me avisan por mail y un registro de quién cambia cosas.

## De dónde parto

Uso la red de la Pre-entrega 6 (región us-east-1): un balanceador (ALB) que reparte el tráfico a 2 servidores EC2 en zonas distintas, una base de datos RDS con réplica, un NAT Gateway para que los servidores salgan a internet, y S3 con CloudFront para los archivos. Lo que más me importa vigilar es que los servidores no se saturen, que la base no se quede sin espacio y que no se me vaya el gasto.

## Dashboard de CloudWatch (mock-up)

Un solo panel con todo junto para ver el estado de un vistazo. Arriba pongo lo más importante (salud, CPU y gasto) y abajo los gráficos de detalle. Cada widget indica el tipo de gráfico y la ventana de tiempo.

| Fila | Widget | Métrica | Tipo de gráfico | Ventana de tiempo |
|---|---|---|---|---|
| 1 | Salud de servidores EC2 | StatusCheckFailed | Número (último valor) | Últimas 3 horas |
| 1 | Gasto proyectado del mes | EstimatedCharges | Número + línea | Últimos 30 días |
| 2 | CPU de servidores EC2 | CPUUtilization | Gráfico de líneas | Últimas 24 horas |
| 2 | Red de servidores EC2 | NetworkIn y NetworkOut | Líneas apiladas | Últimas 24 horas |
| 3 | CPU de la base RDS | CPUUtilization | Gráfico de líneas | Últimas 24 horas |
| 3 | Espacio libre de la base RDS | FreeStorageSpace | Gráfico de líneas | Últimos 7 días |
| 4 | Tamaño de S3 | BucketSizeBytes | Gráfico de barras | Últimos 30 días |
| 4 | Cantidad de archivos en S3 | NumberOfObjects | Número | Últimos 30 días |
| 5 | Errores de la app | Logs de CloudWatch filtrando ERROR (Logs Insights) | Tabla de logs | Últimas 24 horas |

## Matriz de alarmas

Todas avisan por mail usando un tópico de SNS que llamo alertas-mail. Después de crearlo, confirmo la suscripción desde el mail que me manda AWS.

| Alarma | Métrica | Cuándo salta | Durante cuánto | Acción de Alerta |
|---|---|---|---|---|
| CPU alta en EC2 | CPUUtilization | Mayor a 80% | 5 minutos, 2 veces seguidas | Enviar correo a admin@example.com vía SNS (alertas-mail) |
| Servidor con fallas | StatusCheckFailed | Mayor o igual a 1 | 5 minutos, 2 veces seguidas | Enviar correo vía SNS (alertas-mail) |
| Servidor no saludable | UnHealthyHostCount (ALB) | Mayor o igual a 1 | 5 minutos | Enviar correo vía SNS (alertas-mail) |
| Errores en la app | HTTPCode_Target_5XX_Count (ALB) | Mayor a 10 | 5 minutos | Enviar correo vía SNS (alertas-mail) |
| Salida de red anormal | NetworkOut (EC2) | Mayor a 1 GB | 5 minutos, 2 veces seguidas | Enviar correo vía SNS (alertas-mail) |
| Entrada de red anormal | NetworkIn (EC2) | Mayor a 1 GB | 5 minutos, 2 veces seguidas | Enviar correo vía SNS (alertas-mail) |
| CPU alta en RDS | CPUUtilization (RDS) | Mayor a 80% | 5 minutos, 3 veces seguidas | Enviar correo vía SNS (alertas-mail) |
| Base casi sin espacio | FreeStorageSpace (RDS) | Menor al 15% del disco | 5 minutos | Enviar correo vía SNS (alertas-mail) |
| Bucket S3 muy grande | BucketSizeBytes (S3) | Mayor a 50 GB | 1 día | Enviar correo vía SNS (alertas-mail) |
| Gasto del mes | EstimatedCharges | Mayor a 10 USD | 6 horas | Enviar correo vía SNS (alertas-mail) |

La alarma de gasto es obligatoria en cualquier proyecto cloud. Para que funcione hay que activar "Recibir alertas de facturación" en la cuenta, y las métricas de facturación solo existen en us-east-1 (justo mi región). Uso períodos de al menos 5 minutos para que no salten por un pico de un segundo.

## Auditoría

### CloudTrail

CloudTrail registra quién hizo qué en la cuenta. Lo activo en todas las regiones y que guarde los registros en un bucket S3 privado, con validación de integridad de los archivos. Con eso puedo ver quién cambió los Security Groups de la Pre-entrega 6, buscando eventos como AuthorizeSecurityGroupIngress.

Evento crítico: DeleteVpc. Envío los logs de CloudTrail a CloudWatch Logs y creo un filtro de métrica que cuenta los eventos DeleteVpc, con una alarma que salta con 1 solo evento y avisa por SNS (alertas-mail). Lo hago porque borrar la VPC tira abajo toda la red (ALB, EC2, RDS y NAT), así que solo debería pasar a propósito y quiero enterarme enseguida de quién lo hizo.

### AWS Config

AWS Config controla que todo siga bien configurado. Activo 3 reglas:

Regla 1 - s3-bucket-public-read-prohibited: asegura que ningún bucket de S3 sea público.

Regla 2 - restricted-ssh: asegura que nadie abra el puerto 22 a todo internet.

Regla 3 - encrypted-volumes: asegura que los discos EBS de los servidores estén cifrados.

## Conclusión

Con esto veo todo en una pantalla, me entero por mail antes de que algo falle o se dispare el gasto, y sé quién tocó la infraestructura.
