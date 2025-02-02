# MessageChannelMain

`MessageChannelMain` is the main-process-side equivalent of the DOM [`MessageChannel`][] object. Its singular function is to create a pair of connected [`MessagePortMain`](message-port-main.md) objects.

See the [Channel Messaging API][] documentation for more information on using channel messaging.

## Clase: MessageChannelMain

> Interfaz para la mensajería de canales del proceso principal.

Proceso: [Main](../glossary.md#main-process)

Ejemplo:

```js
// Proceso principal
const { MessageChannelMain } = require('electron')
const { port1, port2 } = new MessageChannelMain()
w.webContents.postMessage('port', null, [port2])
port1.postMessage({ some: 'message' })

// Proceso de renderizado
const { ipcRenderer } = require('electron')
ipcRenderer.on('port', (e) => {
  // e.ports es una lista de puertos enviados junto con este mensaje
  e.ports[0].on('message', (messageEvent) => {
    console.log(messageEvent.data)
  })
})
```

### Propiedades de Instancia

#### `channel.port1`

Una propiedad [`MessagePortMain`](message-port-main.md).

#### `channel.port2`

Una propiedad [`MessagePortMain`](message-port-main.md).

[`MessageChannel`]: https://developer.mozilla.org/en-US/docs/Web/API/MessageChannel
[Channel Messaging API]: https://developer.mozilla.org/en-US/docs/Web/API/Channel_Messaging_API
