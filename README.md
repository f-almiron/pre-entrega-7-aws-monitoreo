# Pre-entrega 7: Plan de Monitoreo y Auditoría (AWS)

Curso: Cloud Computing (AWS) - Coderhouse
Alumno: Federico Almiron

Plan sencillo para vigilar la red que armé en la Pre-entrega 6: un dashboard, 5 alarmas que me avisan por mail y un registro de quién cambia cosas.

## De dónde parto

Uso la red de la Pre-entrega 6 (región us-east-1): un balanceador (ALB) que reparte el tráfico a 2 servidores EC2 en zonas distintas, una base de datos RDS con réplica, un NAT Gateway para que los servidores salgan a internet, y S3 con CloudFront para los archivos. Lo que más me importa vigilar es que los servidores no se saturen, que la base no se quede sin espacio y que no se me vaya el gasto.

## Dashboard de CloudWatch

Un solo panel con todo junto para ver el estado de un vistazo. Arriba pongo lo más importante (CPU y gasto) y abajo los gráficos de detalle.

| Qué miro | Métrica |
| --- | --- |
| Servidores EC2: carga y salud | `CPUUtilization` y `StatusCheckFailed` |
| Red | `NetworkIn` y `NetworkOut` de las EC2 |
| Base de datos | RDS: `CPUUtilization` y `FreeStorageSpace` |
| Almacenamiento | S3: `BucketSizeBytes` (cuánto ocupan los archivos) |
| Gasto del mes | `EstimatedCharges` |
| Errores de la app | Logs de CloudWatch, filtrando las líneas que dicen `ERROR` |

## Alarmas

Todas avisan por mail usando un tópico de SNS que llamo `alertas-mail`. Después de crearlo, confirmo la suscripción desde el mail que me manda AWS.

| Alarma | Cuándo salta | Durante cuánto |
| --- | --- | --- |
| CPU alta en EC2 | CPU mayor a 80% | 5 minutos, 2 veces seguidas |
| Servidor con problemas | `UnHealthyHostCount` del ALB mayor o igual a 1 | 5 minutos |
| Errores en la app | Errores 5xx del ALB mayores a 10 | 5 minutos |
| Base casi sin espacio | Espacio libre de RDS menor al 15% | 5 minutos |
| Gasto del mes | `EstimatedCharges` mayor a 10 USD | 6 horas |

La alarma de gasto es obligatoria en cualquier proyecto cloud. Para que funcione hay que activar "Recibir alertas de facturación" en la cuenta, y las métricas de facturación solo existen en us-east-1 (justo mi región). Uso períodos de al menos 5 minutos para que no salten por un pico de un segundo.

## Auditoría

**CloudTrail** registra quién hizo qué en la cuenta. Lo activo en todas las regiones y que guarde los registros en un bucket S3 privado. Con eso puedo ver quién cambió los Security Groups de la Pre-entrega 6, buscando eventos como `AuthorizeSecurityGroupIngress`.

**AWS Config** controla que todo siga bien configurado. Activo la regla `s3-bucket-public-read-prohibited` para asegurar que ningún bucket de S3 sea público, y de yapa `restricted-ssh` para que nadie abra el puerto 22 a todo internet.

## Conclusión

Con esto veo todo en una pantalla, me entero por mail antes de que algo falle o se dispare el gasto, y sé quién tocó la infraestructura.
