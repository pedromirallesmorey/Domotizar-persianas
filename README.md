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

Abrir Ventanas Post-Lluvia
```
alias: Abrir Ventanas Post-Lluvia
description: Abre las ventanas después de la lluvia
triggers:
  - event_type: mobile_app_notification_action
    event_data:
      action: ABRIR_VENTANAS
    trigger: event
actions:
  - action: input_boolean.turn_on
    target:
      entity_id: input_boolean.no_abrir_persiana_lluvia
  - action: cover.open_cover
    target:
      entity_id: cover.persiana_cocina
  - delay:
      seconds: 2
  - action: input_boolean.turn_off
    target:
      entity_id: input_boolean.no_abrir_persiana_lluvia
  - action: notify.mobile_app_sm_s936b_pedro
    data:
      title: ✅ Persiana abierta
      message: La persiana de cocina ha sido abierta
      data:
        tag: fin_lluvia
mode: restart
```

Cancelar Cierre Lluvia
```
alias: Cancelar Cierre Lluvia
description: Cancela el cierre automático de la persiana
triggers:
  - event_type: mobile_app_notification_action
    event_data:
      action: CANCELAR_CIERRE_LLUVIA
    trigger: event
actions:
  - action: input_boolean.turn_on
    target:
      entity_id: input_boolean.cancelar_cierre_lluvia
    data: {}
  - delay:
      seconds: 65
  - action: input_boolean.turn_off
    target:
      entity_id: input_boolean.cancelar_cierre_lluvia
    data: {}
  - action: notify.mobile_app_sm_s936b_pedro
    data:
      title: ⚠️ Cierre cancelado
      message: El cierre automático ha sido cancelado
      data:
        tag: cerrar_ventanas_lluvia
mode: restart
```

Cerrar Ahora Lluvia
```
alias: Cerrar Ahora Lluvia
description: Cierra la persiana inmediatamente sin esperar
triggers:
  - event_type: mobile_app_notification_action
    event_data:
      action: CERRAR_AHORA_LLUVIA
    trigger: event
actions:
  - action: input_boolean.turn_on
    target:
      entity_id: input_boolean.cancelar_cierre_lluvia
    data: {}
  - action: cover.close_cover
    target:
      entity_id: cover.persiana_cocina
    data: {}
  - delay:
      seconds: 2
  - action: input_boolean.turn_off
    target:
      entity_id: input_boolean.cancelar_cierre_lluvia
    data: {}
  - action: notify.mobile_app_sm_s936b_pedro
    data:
      title: ✅ Persiana cerrada
      message: Persiana cerrada inmediatamente
      data:
        tag: cerrar_ventanas_lluvia
mode: restart
```

Cerrar Persianas por Lluvia
```
alias: Cerrar Persianas por Lluvia
description: Pregunta antes de cerrar automáticamente cuando se confirma lluvia
triggers:
  - entity_id: binary_sensor.lluvia_confirmada_meteoclimatic
    to: "on"
    trigger: state
conditions:
  - condition: state
    entity_id: cover.persiana_cocina
    state: open
actions:
  - data:
      title: 🌧️ ¡Lluvia detectada!
      message: >
        Cerrando persiana de cocina en 1 minuto.  {% set precip =
        states('sensor.marratxi_son_ametler_mallorca_daily_precipitation') %} {%
        if precip not in ['unknown', 'unavailable', 'none', ''] %} Precipitación
        hoy: {{ precip }}mm {% endif %} Humedad: {{
        state_attr('weather.marratxi_son_ametler_mallorca', 'humidity') }}%
      data:
        tag: cerrar_ventanas_lluvia
        actions:
          - action: CANCELAR_CIERRE_LLUVIA
            title: Cancelar
          - action: CERRAR_AHORA_LLUVIA
            title: Cerrar ahora
    action: notify.mobile_app_sm_s936b_pedro
  - delay:
      hours: 0
      minutes: 1
      seconds: 0
  - condition: state
    entity_id: input_boolean.cancelar_cierre_lluvia
    state: "off"
  - action: cover.close_cover
    target:
      entity_id: cover.persiana_cocina
  - data:
      title: ✅ Persiana cerrada
      message: >
        Persiana de cocina cerrada por lluvia. {% set precip =
        states('sensor.marratxi_son_ametler_mallorca_daily_precipitation') %} {%
        if precip not in ['unknown', 'unavailable', 'none', ''] %} Precipitación
        hoy: {{ precip }}mm {% endif %} Humedad: {{
        state_attr('weather.marratxi_son_ametler_mallorca', 'humidity') }}%
      data:
        tag: ventanas_cerradas
    action: notify.mobile_app_sm_s936b_pedro
mode: single
```

Cerrar Ventanas Manual
```
alias: Cerrar Ventanas Manual
description: Cierra ventanas desde la notificación móvil
triggers:
  - event_type: mobile_app_notification_action
    event_data:
      action: CERRAR_VENTANAS_MANUAL
    trigger: event
actions:
  - target:
      entity_id:
        - cover.persiana_cocina
    action: cover.close_cover
    data: {}
  - data:
      title: ✅ Ventanas cerradas
      message: Todas las ventanas han sido cerradas
    action: notify.mobile_app_sm_s936b_pedro
```

No Abrir Persiana Post-Lluvia
```
alias: No Abrir Persiana Post-Lluvia
description: Cancela la apertura automática de la persiana
triggers:
  - event_type: mobile_app_notification_action
    event_data:
      action: NO_ABRIR_PERSIANA
    trigger: event
actions:
  - action: input_boolean.turn_on
    target:
      entity_id: input_boolean.no_abrir_persiana_lluvia
  - delay:
      seconds: 125
  - action: input_boolean.turn_off
    target:
      entity_id: input_boolean.no_abrir_persiana_lluvia
  - action: notify.mobile_app_sm_s936b_pedro
    data:
      title: ⚠️ Apertura cancelada
      message: La persiana permanecerá cerrada
      data:
        tag: fin_lluvia
mode: restart
```

Notificar Fin Lluvia
```
alias: Notificar Fin Lluvia
description: Avisa cuando deja de llover con opción de abrir, o abre automáticamente en 2 minutos
triggers:
  - entity_id: binary_sensor.lluvia_confirmada_meteoclimatic
    to: "off"
    for:
      hours: 0
      minutes: 10
      seconds: 0
    trigger: state
conditions:
  - condition: sun
    before: sunset
    after: sunrise
  - condition: state
    entity_id: cover.persiana_cocina
    state: closed
actions:
  - data:
      title: ☀️ Ha dejado de llover
      message: ¿Abrir persiana de cocina? Se abrirá automáticamente en 2 minutos si no respondes.
      data:
        tag: fin_lluvia
        actions:
          - action: ABRIR_VENTANAS
            title: Abrir ahora
          - action: NO_ABRIR_PERSIANA
            title: Dejar cerrada
    action: notify.mobile_app_sm_s936b_pedro
  - delay:
      hours: 0
      minutes: 2
      seconds: 0
  - condition: state
    entity_id: input_boolean.no_abrir_persiana_lluvia
    state: "off"
  - condition: state
    entity_id: cover.persiana_cocina
    state: closed
  - action: cover.open_cover
    target:
      entity_id: cover.persiana_cocina
  - action: notify.mobile_app_sm_s936b_pedro
    data:
      title: ✅ Persiana abierta
      message: Persiana de cocina abierta automáticamente tras dejar de llover
      data:
        tag: fin_lluvia
mode: single
```
🎯 Cómo funciona
* Deja de llover → Notificación: "¿Abrir persiana? Se abrirá en 2 min"

Tienes 3 opciones: 
* ✅ "Abrir ahora" → Abre inmediatamente
* ❌ "Dejar cerrada" → Cancela apertura automática
* ⏱️ No hacer nada → Espera 2 minutos y abre automáticamente

Protección: Si durante esos 2 minutos: 
Cierras la persiana manualmente → No la abrirá (comprueba que siga cerrada)
Pulsas cualquier botón → Cancela la apertura automática


Persiana cocina abierta amanecer
```
alias: Persiana cocina abierta amanecer
description: ""
triggers:
  - trigger: sun
    event: sunrise
    offset: 0
conditions:
  - condition: device
    device_id: 01b8c579342b7316b949b715d378f4f8
    domain: cover
    entity_id: c2a9f78df8bfb0c82c00f48577da3852
    type: is_closed
  - condition: device
    device_id: 01b8c579342b7316b949b715d378f4f8
    domain: cover
    entity_id: c2a9f78df8bfb0c82c00f48577da3852
    type: is_position
    below: 100
  - condition: state
    entity_id: binary_sensor.lluvia_confirmada_meteoclimatic
    state:
      - "off"
actions:
  - action: cover.open_cover
    metadata: {}
    data: {}
    target:
      entity_id: cover.persiana_cocina
  - action: notify.mobile_app_sm_s936b_pedro
    metadata: {}
    data:
      title: ✅ Persiana cocina abierta
      message: Persiana cocina abierta al amanecer
mode: single
```

Persiana cocina cerrada atardecer
```
alias: Persiana cocina cerrada atardecer
description: ""
triggers:
  - trigger: sun
    event: sunset
    offset: 0
conditions:
  - condition: or
    conditions:
      - condition: state
        entity_id: cover.persiana_cocina
        state:
          - open
actions:
  - action: cover.close_cover
    metadata: {}
    data: {}
    target:
      entity_id: cover.persiana_cocina
  - action: notify.mobile_app_sm_s936b_pedro
    metadata: {}
    data:
      message: Persiana cocina cerrada al atardecer
      title: ✅ Persiana cocina cerrada
mode: single
```






