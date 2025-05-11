---
title: Configurar un pin numérico para validar la ejecución de una acción en Home Assistant
date: 2025-05-11
categories:
  - raspberry
  - home-assistant
  - tutorial
---
# Configurar un PIN numérico para validar la ejecución de una acción en Home Assistant

En mi casa tengo una cerraja electrónica, que me permite abrir la puerta de entrada desde el móvil. Se trata de una *Nuki Lock Pro 4*. Está integrada con *Home Assistant* mediante *MQTT*. Y se exponen varias acciones ante el sistema que se pueden llevar a cabo, como bloquear/desbloquear la puerta, abrir la puerta... etc.

Una de esas acciones, la de abrir la puerta, es la que más utilizo. Dado que tengo mi instancia de *Home Assistant* expuesta al exterior, puedo abrir la puerta en cualquier momento a distancia. Sin embargo, la acción que provee el dispositivo de Nuki, es un simple botón que cuando lo pulsas se abre la puerta.

La acción no tiene ningún tipo de control de confirmación, es por eso que se me antoja un poco "peligrosa", ya que podría pulsar el botón por accidente desde la distancia y dejar la puerta abierta sin querer.

De ahí este post. El objetivo es documentar la implementación en *Home Assistant* de un desarrollo que permita que cuando se pulse dicha acción, aparezca un *keypad* numérico para introducir un pin preestablecido. Si se introduce de forma correcta, entonces se abrirá la puerta.

Como plus, si el pin se introduce de forma incorrecta, se enviará una notificación a la aplicación móvil. Además, en determinadas ocasiones, el dispositivo de Nuki se queda "desconectado" de la red. Me ha sucedido en alguna ocasión que si pulso sobre la acción, cuando "vuelve en sí", se ejecuta. Cosa que me parece muy peligrosa, ya que puedes pulsar para abrir la puerta pensando que no ha hecho nada y al rato se abre sola. Esto también lo controlaremos, evitando que se pueda ejecutar la acción si el dispositivo no está disponible.

Dicho todo esto, vamos a ello!!

## *Stack* tecnológico

En mi caso, tengo *Home Assistant* versión 2025.1.0, corriendo en un contenedor docker dentro de una Raspberry pi 4B. Además, necesitaremos crear y modificar archivos dentro de la instancia, así que os recomiendo configurar algún tipo de *plugin* que os permita este tipo de cosas. Yo en mi caso tengo otro contenedor docker denominado *hass-configurator*.

También necesitaremos tener instalado *HACS*, ya que utilizaremos algunos repositorios que nos permitan jugar con la interfaz gráfica.

## Instalación de HACS

Los repositorios que debemos instalar en *HACS* son:

- Browser_mod: para poder implementar una ventana modal de forma sencilla.
- Template-entity-row: para poder modificar el comportamiento de una fila, en una tarjeta existente (como la que tengo para el control de la cerraja electrónica).

## Construcción del *keypad* numérico

*Home Assistant* utiliza una tecnología denominada *lovelace* para el renderizado de su interfaz gráfica. El primer paso será crear una vista *lovelace* con un teclado numérico para poder introducir el pin que configuremos.

> En mi instalación de *Home Assistant* la configuración se encuentra dentro de *hass-config*, así que todo estará configurado en base a esta ruta.

Utilizando el editor, sobre la ruta */hass-config/www*, creamos un nuevo fichero denominado *my-pin-pad-lit.js*, con el siguiente contenido:

```javascript
import { LitElement, html, css } from "https://unpkg.com/lit-element/lit-element.js?module";

class MyPinPadLit extends LitElement {
    
  // inicialización de propiedades
  static properties = {
    hass: { type: Object },
    config: { type: String },
    pin: { type: String },
    errorMessage: { type: String },
    pinStatus: { type: String },
  };

  constructor() {
    super();
    this.pin = "";
    this.errorMessage = "";
    this.pinStatus = "";
  }

  // estilos para la interfaz
  static styles = css`
    :host {
      display: flex;
      flex-direction: column;
      justify-content: center;
      align-items: center;
      height: 100vh;
      box-sizing: border-box;
      padding: 1rem;
    }

    .pad-container {
      display: flex;
      flex-direction: column;
      align-items: center;
      width: 100%;
    }

    .pin-input {
      margin-bottom: 1rem;
      font-size: 2.5rem;
      padding: 1rem;
      text-align: center;
      width: 80%;
      border: 3px solid #ccc;
      border-radius: 1rem;
      transition: border 0.3s;
    }

    .pin-input.success {
      border-color: green;
    }

    .pin-input.error {
      border-color: red;
    }

    .error-message {
      color: red;
      font-size: 1.2rem;
      margin-bottom: 1rem;
      min-height: 1.5rem;
      text-align: center;
    }

    .keypad {
      display: grid;
      grid-template-columns: repeat(3, 1fr);
      gap: 1rem;
      width: 100%;
      max-width: 500px;
    }

    .key {
      background: #03a9f4;
      color: white;
      border: none;
      font-size: 2rem;
      padding: 1.2rem;
      border-radius: 1rem;
      cursor: pointer;
      transition: background 0.2s;
      width: 100%;
    }

    .key:active {
      background: #0288d1;
    }

    .key-control {
      background: #f44336;
    }

    .key-control:active {
      background: #d32f2f;
    }
  `;

  // establecer la configuración
  setConfig(config) {
    this.config = config;
  }

  // renderización de la vista
  render() {
    // Acceder al estado de la entidad
    const estadoPuerta = this.hass.states['button.puerta_de_entrada_unlatch']?.state;
    
    // Verificar si el estado no es 'unavailable'
    const mostrarPad = estadoPuerta !== 'unavailable';
    
    return html`
      <div class="pad-container">
        <!-- Solo mostrar el pad si la puerta no está en 'unavailable' -->
        ${mostrarPad ? html`
          <input
            class="pin-input ${this.pinStatus}"
            type="password"
            .value=${this.pin}
            disabled
          />
          <div class="error-message">${this.errorMessage}</div>
          <div class="keypad">
            ${[1,2,3,4,5,6,7,8,9,"←",0,"OK"].map(key =>
              html`
                <button
                  class="key ${key === "OK" ? "key-control" : ""}"
                  @click=${() => this._handleKeyPress(key)}
                >${key}</button>
              `
            )}
          </div>
        ` : html`
          <div class="error-message">La puerta no está disponible</div>
        `}
      </div>
    `;
  }

  // manejar la pulsación de teclas
  _handleKeyPress(key) {
      
    // borrar el último carácter introducido
    if (key === "←") {
      this.pin = this.pin.slice(0, -1);
      
    // validar el pin introducido
    } else if (key === "OK") {
      this._validatePin();
      
    // añadir dígitos al pin
    } else if (this.pin.length < 4) {
      this.pin += key;
    }
  }

  // valida el pin introducido
  _validatePin() {
    const correctPin = this.hass.states["input_text.pin_seguro"].state;

    // si el pin es correcto...
    if (this.pin === correctPin) {
        console.log('Entro por aquí');
        this.pinStatus = "success";
        
        // espero un segundo
        setTimeout(() => {
            this.hass.callService("button", "press", {
                entity_id: this.config.script_action,
            });
            this._closePopup();
        }, 1000);
        
    } else {
        this.pinStatus    = "error";
        this.errorMessage = "PIN incorrecto";
    
        // incremento número de intentos
        this.hass.callService("input_number", "increment", {
            entity_id: "input_number.intentos_fallidos_pin",
        });

        // limpio pin
        this.pin = "";
        
        // espero 2 segundos
        setTimeout(() => {
            this.errorMessage = "";
            this.pinStatus = "";
        }, 2000);
    }
  }

  // cerrar la ventana modal
  _closePopup() {
    this.hass.callService("browser_mod", "close_popup");
  }

  // determina el tamaño de la tarjeta
  getCardSize() {
    return 3;
  }
}

customElements.define("my-pin-pad-lit", MyPinPadLit);
```

Como se puede observar, utilizamos *lit*. **Lit** es una biblioteca moderna y ligera para construir interfaces de usuario (UI) en la web, desarrollada por Google. Es especialmente popular en el ecosistema de *Home Assistant* porque permite crear componentes de Lovelace personalizados de manera eficiente, con un rendimiento óptimo y compatibilidad con las tecnologías web estándar.

> Lit nos permite que la interfaz se renderice con mayor compatibilidad entre distintos dispositivos. Por ejemplo desde la aplicación móvil de *Home Assistant*. Además, como esta funcionalidad se va a utilizar en la inmensa mayoría de casos desde la aplicación móvil, se ha implementado una interfaz con botones grandes, para una mejor UX.

Como puntos a destacar en este código, en el método *render*, comprobamos primero que la acción que vamos a ejecutar, en este caso *button.puerta_de_entrada_unlatch* esté disponible. En caso de no ser así, se mostrará un mensaje de error indicando que *"La puerta no está disponible"*, para evitar, como ya he explicado anteriormente, que se ejecute accidentalmente la acción en un momento que no tengamos controlado.

También decir, que se irán acumulando el número de intentos fallidos, en una nueva entidad. Con la que no haremos nada, pero nos permitirá implementar otras funcionalidades a futuro, como por ejemplo bloquear la acción ante x intentos fallidos.

## Creación de nuevas entidades

El siguiente paso será crear las nuevas entidades que van a almacenar ciertos valores, como el número de intentos fallidos y otra para almacenar el valor del pin que tenemos configurado.

Creamos un nuevo fichero denominado *input_text.yaml* en */hass-config* con el siguiente texto:

```yaml
pin_seguro:
  name: PIN Puerta
  max: 4
```

Creamos también otro fichero en la misma ruta denominado *input_number.yaml* con el siguiente contenido:

```yaml
intentos_fallidos_pin:
  name: Intentos fallidos PIN
  initial: 0
  min: 0
  max: 10
  step: 1
  mode: box
```

Ahora necesitamos que *Home Assintant* cargue estas nuevas entidades y nuestra nueva interfaz, así que editamos el archivo *configuration.yaml* e incluimos las siguientes entradas:

```yaml
...
# Load frontend themes from the themes folder
frontend:
  themes: !include_dir_merge_named themes
  extra_module_url:
    - /local/my-pin-pad/my-pin-pad.js
...
# Acceso al pin desde secrets.yaml
sensor:
  - platform: template
    sensors:
      pin_seguro_template:
        friendly_name: "PIN Seguro"
        value_template: "{{ states('input_text.pin_seguro') }}"
...
input_text: !include input_text.yaml
input_number: !include input_number.yaml
...
```

> Puede que la entrada *frontend* con el *include_dir_merge_name themes* ya exista, de no ser así, habrá que crearla.

## Crear las automatizaciones oportunas

Para poder mostrar nuestra nueva vista y ejecutar la acción correspondiente si el pin introducido es correcto, necesitaremos automatizaciones, que incluiremos en el archivo *automations.yaml*

```yaml
...
- id: establecer_pin_serguro
  alias: Establecer PIN seguro al arrancar
  trigger:
  - platform: homeassistant
    event: start
  action:
  - service: input_text.set_value
    data:
      entity_id: input_text.pin_seguro
      value: '{{ states(''input_text.pin_seguro'') }}'
  mode: single
- id: notificar_intentos_fallidos
  alias: Notificar los intentos fallidos
  trigger:
  - platform: state
    entity_id: input_number.intentos_fallidos_pin
  condition: []
  action:
  - service: notify.mobile_app_iphone_de_alberto
    data:
      message: "\U0001F6A8 Se ha introducido un PIN incorrecto tratando de abrir la
        puerta"
  mode: single
...
```

*Establecer_pin_seguro* es la automatización encargada de establecer el valor configurado de pin sobre la entidad *input_text.pin_seguro* para que posteriormente podamos utilizarla en nuestro fichero javascript.

*Notificar_intentos_fallidos* es otra automatización para enviar una notificación a un dispositivo móvil, a través de la aplicación de *Home Assistant* cada vez que se intente establecer un pin y se falle.

## Modificar el panel existente para mostrar nuestro *keypad*

Para terminar, deberemos modificar la interfaz de usuario existente donde se encuentra la acción que queremos ejecutar protegida tras el pin. En mi caso, se trata de una tarjeta *lovelace* con las acciones dedicadas a la gestión de la cerraja Nuki, situada en la puerta de entrada.

Para ello, modifico la fila (gracias al *plugin* de *HACS* instalado anteriormente llamado *template_entity_row*) con el siguiente código yaml:

```yaml
type: custom:template-entity-row
entity: button.puerta_de_entrada_unlatch
name: Abrir con PIN
icon: mdi:door-open
tap_action:
  action: fire-dom-event
  browser_mod:
    service: browser_mod.popup
    data:
      title: Introduce el PIN
      size: normal
      content:
        type: custom:my-pin-pad-lit
        correct_pin: "<nuestro_pin_configurado>"
        script_action: button.puerta_de_entrada_unlatch
```

## Recargar la configuración de *Home Assistant*

Por último, sólo nos quedaría reiniciar *Home Assistant* para aplicar los cambios. Si todo ha ido bien, al pulsar sobre la acción *Abrir con PIN*, debería aparecer nuestro *keypad* numérico. Si introducimos un pin incorrecto, debería llegarnos una notificación a nuestro dispositivo móvil. Si en cambio lo introducimos correctamente, se debería abrir la puerta.

Y esto es todo, espero que os haya gustado y que os haya sido útil. Hasta la próxima aventura!!
