# Domotizar-persianas
## Como domotizar persianas

## Material utilizado:
* Schellenberg 20407 
Motor Tubular Eléctrico para Persianas, con Eje de 40 mm, Juego Completo Incl. Soporte Mural, 15 kg, 6 nm
<img width="243" height="229" alt="Image" src="https://github.com/user-attachments/assets/f751e76a-9fcf-4a55-93cf-55d288034c9e" />

* Shelly 2PM Gen3
Relé Interruptor de Persianas Inalàmbrico, 2 Canales - 16 A, Wi-Fi & Bluetooth, con Medidor de Consumo
<img width="143" height="129" alt="Image" src="https://github.com/user-attachments/assets/3031d615-13d1-4156-bcf0-c28e0fded1b3" />

* Schellenberg 
Pulsador de Persianas, Interruptor Empotrable Pared
<img width="143" height="129" alt="Image" src="https://github.com/user-attachments/assets/0e7821bd-54f4-499b-8381-a9417c84e2dc" />

* Sensor de lluvia y luz Tiardey Smart Zigbee
<img width="143" height="247" alt="Image" src="https://github.com/user-attachments/assets/2ddb1e8c-465c-49f4-b17e-4a6504ef66b0" />

## Esquema de conexión:

<img width="747" height="486" alt="Image" src="https://github.com/user-attachments/assets/ba0d1eec-347c-4e2d-9dfb-d5d929d7db52" />


### 🔑 Entidades principales
* Sensor físico de lluvia Tiardey / binary_sensor.sensor_lluvia
* Estación Meteoclimatic Marratxí / weather.marratxi_son_ametler_mallorca
* Persiana / cover.persiana_cocina
* Servicio de notificaciones / notify.mobile_app_sm_s936b_pedro

### 📊 Variables usadas en validación
* precipitation → Precipitación (si está disponible en Meteoclimatic)
* humidity → Humedad relativa (%)
* temperature → Temperatura (°C)
* condition → Estado del tiempo (rainy, pouring, etc.)
* sensor_local → Estado del sensor Tiardey (on = mojado)

### 🧩 SENSORES/AYUDANTES
### Sensor binario
Lluvia Confirmada Meteoclimatic (Sensor binario)
Creamos un sensor que detecta si es lluvia real o rocío.

```
{# Datos Meteoclimatic #}
{% set precipitacion_diaria = states('sensor.marratxi_son_ametler_mallorca_daily_precipitation') | float(0) %}
{% set humedad = state_attr('weather.marratxi_son_ametler_mallorca', 'humidity') | float(0) %}
{% set condicion = states('weather.marratxi_son_ametler_mallorca') %}
      
{# Filtro horario inteligente #}
{% set hora = now().hour %}
{% set mes = now().month %}
{% set temperatura = states('sensor.marratxi_son_ametler_mallorca_temperature') | float(15) %}
{% if temperatura < 5 %}
  {% set inicio_rocio = 7 %}
  {% set fin_rocio = 11 %}
{% elif temperatura > 25 %}
  {% set inicio_rocio = 4 %}
  {% set fin_rocio = 7 %}
{% elif mes in [6, 7, 8] %}
  {% set inicio_rocio = 5 %}
  {% set fin_rocio = 8 %}
{% elif mes in [12, 1, 2] %}
  {% set inicio_rocio = 6 %}
  {% set fin_rocio = 10 %}
{% else %}
  {% set inicio_rocio = 5 %}
  {% set fin_rocio = 9 %}
{% endif %}
      
{# Validaciones #}
{% set meteoclimatic_llueve = precipitacion_diaria > 0 or condicion in ['rainy', 'pouring', 'lightning-rainy'] %}
{% set humedad_lluvia = humedad > 75 %}
{% set fuera_hora_rocio = not (inicio_rocio <= hora < fin_rocio) %}
      
{# Lluvia confirmada si: sensor mojado + tiempo + (meteoclimatic O humedad alta O fuera de hora rocío) #}
{{ sensor_local and tiempo_min and (meteoclimatic_llueve or humedad_lluvia or fuera_hora_rocio) }}
```

### Posible Rocío
```
{% set sensor_mojado = is_state('binary_sensor.sensor_lluvia', 'on') %}
{% set precipitacion_diaria = states('sensor.marratxi_son_ametler_mallorca_daily_precipitation') | float(0) %}
{% set humedad = state_attr('weather.marratxi_son_ametler_mallorca', 'humidity') | float(0) %}
{% set condicion = states('weather.marratxi_son_ametler_mallorca') %}
          
{{ sensor_mojado and precipitacion_diaria == 0 and humedad < 75 and condicion not in ['rainy', 'pouring', 'lightning-rainy'] }}
```

### Estado Lluvia Detallado
```
{% set sensor = is_state('binary_sensor.sensor_lluvia', 'on') %}
{% set precipitacion_diaria = states('sensor.marratxi_son_ametler_mallorca_daily_precipitation') | float(0) %}
{% set humedad = state_attr('weather.marratxi_son_ametler_mallorca', 'humidity') | float(0) %}
{% set condicion = states('weather.marratxi_son_ametler_mallorca') %}
          
{% if sensor and (precipitacion_diaria > 0 or condicion in ['rainy', 'pouring']) %}
   Lluvia confirmada
{% elif sensor and humedad > 75 %}
   Probable lluvia
{% elif sensor and humedad < 75 %}
   Probable rocío
{% elif not sensor and precipitacion_diaria > 0 %}
   Lluvia no local
{% else %}
   Despejado
{% endif %}
```

### Persianas abiertas, cerradas y entreabiertas
Sensor abiertas
```
{{ states.cover | selectattr('attributes.current_position', 'defined') | selectattr('attributes.current_position', '==', 100) | list | count }}
```

Sensor cerradas
```
{{ states.cover | selectattr('attributes.current_position', 'defined') | selectattr('attributes.current_position', '==', 0) | list | count }}
```

Sensor entreabiertas
```
{{ states.cover | selectattr('attributes.current_position', 'defined') | selectattr('attributes.current_position', '>', 0) | selectattr('attributes.current_position', '<', 100) | list | count }}
```

### Input Boolean
Cancelar cierre lluvia  (input_boolean)
```
cancelar_cierre_lluvia:     
  name: Cancelar cierre automático por lluvia     
  initial: false
```

No abrir persiana lluvia  (input_boolean)
```
no_abrir_persiana_lluvia:
  name: No abrir persiana después de lluvia
  initial: false
```

## ⚙️ AUTOMATIZACIONES

Alerta Posible Rocío
```
alias: Alerta Posible Rocío
description: Informa de posibles falsos positivos por rocío
triggers:
  - entity_id: binary_sensor.posible_rocio
    to: "on"
    for:
      minutes: 5
    trigger: state
actions:
  - data:
      title: 💧 Posible rocío detectado
      message: >-
        Sensor local mojado pero sin lluvia en Meteoclimatic. Probablemente es
        rocío o condensación.
      data:
        tag: alerta_rocio
    action: notify.mobile_app_sm_s936b_pedro
```









