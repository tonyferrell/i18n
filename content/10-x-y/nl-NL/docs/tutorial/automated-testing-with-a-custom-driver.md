# Geautomatiseerd testen met een aangepast stuurprogramma

Om geautomatiseerde tests voor je Electron app te schrijven heb je een manier nodig om je applicatie te "schijven". [Spectron](https://electronjs.org/spectron) is een veelgebruikte oplossing waarmee u gebruikersacties kunt emuleren via [WebDriver](http://webdriver.io/). Het is echter ook mogelijk om uw eigen aangepaste stuurprogramma te schrijven met behulp van ingebouwde IPC-overSTDIO. Het voordeel van een custom driver is dat het minder overhead vereist dan Spectron, en laat je aangepaste methoden aan je testset blootstellen.

Om een aangepast stuurprogramma te maken, gebruiken we Node.js' [child_process](https://nodejs.org/api/child_process.html) API. De test suite zal het Electron proces starten en vervolgens een eenvoudig berichtenprotocol instellen:

```js
const childProcess = require('child_process')
const electronPath = require('electron')

// spawn the process
let env = { /* ... */ }
let stdio = ['inherit', 'inherit', 'inherit', 'ipc']
let appProcess = childProcess.spawn(electronPath, ['./app'], { stdio, env })

// listen for IPC messages from the app
appProcess.on('message', (msg) => {
  // ...
})

// stuur een IPC bericht naar de app
appProcess.send({ my: 'message' })
```

Van binnen de Electron app, kunt u naar berichten luisteren en antwoorden sturen met behulp van de Node.js [proces](https://nodejs.org/api/process.html) API:

```js
// luisteren voor IPC berichten van de test suite
process.on('message', (msg) => {
  // ...
})

// stuur een IPC-bericht naar het testsuite
process.send({ my: 'message' })
```

We kunnen nu vanuit de testsuite communiceren met de Electron app via het `appProcess` object.

Voor het gemak kunt u `appProcess` in een bestuurder object wikkelen dat meer hoogwaardige functies biedt. Hier is een voorbeeld van hoe je dit kunt doen:

```js
class TestDriver {
  constructor ({ path, args, env }) {
    this.rpcCalls = []

    // start child process
    env.APP_TEST_DRIVER = 1 // let the app know it should listen for messages
    this.process = childProcess.spawn(path, args, { stdio: ['inherit', 'inherit', 'inherit', 'ipc'], env })

    // handle rpc responses
    this.process.on('message', (message) => {
      // pop the handler
      let rpcCall = this.rpcCalls[message.msgId]
      if (!rpcCall) return
      this.rpcCalls[message.msgId] = null
      // reject/resolve
      if (message.reject) rpcCall.reject(message.reject)
      else rpcCall.resolve(message.resolve)
    })

    // wait for ready
    this.isReady = this.rpc('isReady').catch((err) => {
      console.error('Application failed to start', err)
      this.stop()
      process.exit(1)
    })
  }

  // simple RPC call
  // to use: driver.rpc('method', 1, 2, 3).then(...)
  async rpc (cmd, ...args) {
    // send rpc request
    let msgId = this.rpcCalls.length
    this.process.send({ msgId, cmd, args })
    return new Promise((resolve, reject) => this.rpcCalls.push({ resolve, reject }))
  }

  stop () {
    this.process.kill()
  }
}
```

In de app moet je een eenvoudige handler schrijven voor de RPC-oproepen:

```js
if (process.env.APP_TEST_DRIVER) {
  proces. n('bericht', onMessage)
}

async functie onMessage ({ msgId, cmd, args }) {
  let methode = METHODS[cmd]
  als (! ethod) methode = () => nieuwe fout ('Ongeldige methode: ' + cmd)
  probeer {
    let op te lossen = wacht method(. .args)
    proces. end({ msgId, resolve })
  } catch (err) {
    let reject = {
      message: err.message,
      stack: err.stack,
      name: err.name
    }
    proces. end({ msgId, reject })
  }
}

const METHODS = {
  isReady () {
    // doe elke setup nodig
    return true
  }
  // definieer je RPC-able methoden hier
}
```

Vervolgens kunt u in uw testset uw test-driver op de volgende manier gebruiken:

```js
const test = require('ava')
const electronPath = require('electron')

let app = new TestDriver({
  path: electronPath,
  args: ['. app'],
  env: {
    NODE_ENV: 'test'
  }
})
test.before(async t => {
  await app.isReady
})
test. fter.always('cleanup', async t => {
  wacht app.stop()
})
```