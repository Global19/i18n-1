# API de Timing de Rendimiento

<!--introduced_in=v8.5.0-->

> Estabilidad: 1 - Experimental

The Performance Timing API provides an implementation of the [W3C Performance Timeline](https://w3c.github.io/performance-timeline/) specification. The purpose of the API is to support collection of high resolution performance metrics. Esta es la misma API de rendimiento implementada en los navegadores Web modernos.

```js
const { PerformanceObserver, performance } = require('perf_hooks');

const obs = new PerformanceObserver((items) => {
  console.log(items.getEntries()[0].duration);
  performance.clearMarks();
});
obs.observe({ entryTypes: ['measure'] });

performance.mark('A');
doSomeLongRunningProcess(() => {
  performance.mark('B');
  performance.measure('A to B', 'A', 'B');
});
```

## Clase: Performance

<!-- YAML
added: v8.5.0
-->

### performance.clearMarks([name])

<!-- YAML
added: v8.5.0
-->

* `name` {string}

If `name` is not provided, removes all `PerformanceMark` objects from the Performance Timeline. Si se proporciona `name`, esto elimina solo la marca nombrada.

### performance.mark([name])

<!-- YAML
added: v8.5.0
-->

* `name` {string}

Crea una nueva entrada `PerformanceMark` en la Línea de Tiempo de Rendimiento. A `PerformanceMark` is a subclass of `PerformanceEntry` whose `performanceEntry.entryType` is always `'mark'`, and whose `performanceEntry.duration` is always `0`. Performance marks are used to mark specific significant moments in the Performance Timeline.

### performance.measure(name, startMark, endMark)

<!-- YAML
added: v8.5.0
-->

* `name` {string}
* `startMark` {string}
* `endMark` {string}

Crea una nueva entrada `PerformanceMeasure` en la Línea de Tiempo de Rendimiento. A `PerformanceMeasure` is a subclass of `PerformanceEntry` whose `performanceEntry.entryType` is always `'measure'`, and whose `performanceEntry.duration` measures the number of milliseconds elapsed since `startMark` and `endMark`.

The `startMark` argument may identify any *existing* `PerformanceMark` in the Performance Timeline, or *may* identify any of the timestamp properties provided by the `PerformanceNodeTiming` class. If the named `startMark` does not exist, then `startMark` is set to [`timeOrigin`][] by default.

The `endMark` argument must identify any *existing* `PerformanceMark` in the Performance Timeline or any of the timestamp properties provided by the `PerformanceNodeTiming` class. If the named `endMark` does not exist, an error will be thrown.

### performance.nodeTiming

<!-- YAML
added: v8.5.0
-->

* {PerformanceNodeTiming}

An instance of the `PerformanceNodeTiming` class that provides performance metrics for specific Node.js operational milestones.

### performance.now()

<!-- YAML
added: v8.5.0
-->

* Devuelve: {number}

Returns the current high resolution millisecond timestamp, where 0 represents the start of the current `node` process.

### performance.timeOrigin

<!-- YAML
added: v8.5.0
-->

* {number}

The [`timeOrigin`][] specifies the high resolution millisecond timestamp at which the current `node` process began, measured in Unix time.

### performance.timerify(fn)

<!-- YAML
added: v8.5.0
-->

* `fn` {Function}

Wraps a function within a new function that measures the running time of the wrapped function. A `PerformanceObserver` must be subscribed to the `'function'` event type in order for the timing details to be accessed.

```js
const {
  performance,
  PerformanceObserver
} = require('perf_hooks');

function someFunction() {
  console.log('hello world');
}

const wrapped = performance.timerify(someFunction);

const obs = new PerformanceObserver((list) => {
  console.log(list.getEntries()[0].duration);
  obs.disconnect();
});
obs.observe({ entryTypes: ['function'] });

// Una entrada para el tiempo de rendimiento será creada envuelta();
```

## Clase: PerformanceEntry

<!-- YAML
added: v8.5.0
-->

### performanceEntry.duration

<!-- YAML
added: v8.5.0
-->

* {number}

El número total de milisegundos transcurridos para esta entrada. This value will not be meaningful for all Performance Entry types.

### performanceEntry.name

<!-- YAML
added: v8.5.0
-->

* {string}

El nombre de la entrada de rendimiento.

### performanceEntry.startTime

<!-- YAML
added: v8.5.0
-->

* {number}

The high resolution millisecond timestamp marking the starting time of the Performance Entry.

### performanceEntry.entryType

<!-- YAML
added: v8.5.0
-->

* {string}

El tipo de la entrada de rendimiento. Currently it may be one of: `'node'`, `'mark'`, `'measure'`, `'gc'`, `'function'`, or `'http2'`.

### performanceEntry.kind

<!-- YAML
added: v8.5.0
-->

* {number}

When `performanceEntry.entryType` is equal to `'gc'`, the `performance.kind` property identifies the type of garbage collection operation that occurred. El valor puede ser uno de los siguientes:

* `perf_hooks.constants.NODE_PERFORMANCE_GC_MAJOR`
* `perf_hooks.constants.NODE_PERFORMANCE_GC_MINOR`
* `perf_hooks.constants.NODE_PERFORMANCE_GC_INCREMENTAL`
* `perf_hooks.constants.NODE_PERFORMANCE_GC_WEAKCB`

## Clase: PerformanceNodeTiming expande PerformanceEntry

<!-- YAML
added: v8.5.0
-->

Proporciona detalles de timing para el propio Node.js.

### performanceNodeTiming.bootstrapComplete

<!-- YAML
added: v8.5.0
-->

* {number}

The high resolution millisecond timestamp at which the Node.js process completed bootstrapping. If bootstrapping has not yet finished, the property has the value of -1.

### performanceNodeTiming.loopExit

<!-- YAML
added: v8.5.0
-->

* {number}

The high resolution millisecond timestamp at which the Node.js event loop exited. Si el bucle del evento aún no ha cerrado, la propiedad tiene el valor de -1. Solo no puede tener un valor de -1 en un manejador del evento [`'exit'`][].

### performanceNodeTiming.loopStart

<!-- YAML
added: v8.5.0
-->

* {number}

The high resolution millisecond timestamp at which the Node.js event loop started. If the event loop has not yet started (e.g., in the first tick of the main script), the property has the value of -1.

### performanceNodeTiming.nodeStart

<!-- YAML
added: v8.5.0
-->

* {number}

The high resolution millisecond timestamp at which the Node.js process was initialized.

### performanceNodeTiming.v8Start

<!-- YAML
added: v8.5.0
-->

* {number}

The high resolution millisecond timestamp at which the V8 platform was initialized.

## Clase: PerformanceObserver

### nuevo PerformanceObserver(callback)

<!-- YAML
added: v8.5.0
-->

* `callback` {Function} 
  * `list` {PerformanceObserverEntryList}
  * `observer` {PerformanceObserver}

`PerformanceObserver` objects provide notifications when new `PerformanceEntry` instances have been added to the Performance Timeline.

```js
const {
  performance,
  PerformanceObserver
} = require('perf_hooks');

const obs = new PerformanceObserver((list, observer) => {
  console.log(list.getEntries());
  observer.disconnect();
});
obs.observe({ entryTypes: ['mark'], buffered: true });

performance.mark('test');
```

Because `PerformanceObserver` instances introduce their own additional performance overhead, instances should not be left subscribed to notifications indefinitely. Users should disconnect observers as soon as they are no longer needed.

The `callback` is invoked when a `PerformanceObserver` is notified about new `PerformanceEntry` instances. The callback receives a `PerformanceObserverEntryList` instance and a reference to the `PerformanceObserver`.

### performanceObserver.disconnect()

<!-- YAML
added: v8.5.0
--> Disconnects the 

`PerformanceObserver` instance from all notifications.

### performanceObserver.observe(options)

<!-- YAML
added: v8.5.0
-->

* `options` {Object} 
  * `entryTypes` {string[]} An array of strings identifying the types of `PerformanceEntry` instances the observer is interested in. If not provided an error will be thrown.
  * `buffered` {boolean} If true, the notification callback will be called using `setImmediate()` and multiple `PerformanceEntry` instance notifications will be buffered internally. If `false`, notifications will be immediate and synchronous. **Predeterminado:** `false`.

Subscribes the `PerformanceObserver` instance to notifications of new `PerformanceEntry` instances identified by `options.entryTypes`.

When `options.buffered` is `false`, the `callback` will be invoked once for every `PerformanceEntry` instance:

```js
const {
  performance,
  PerformanceObserver
} = require('perf_hooks');

const obs = new PerformanceObserver((list, observer) => {
  // llamado tres veces de forma sincrónica. la lista contiene un elemento
});
obs.observe({ entryTypes: ['mark'] });

for (let n = 0; n < 3; n++)
  performance.mark(`test${n}`);
```

```js
const {
  performance,
  PerformanceObserver
} = require('perf_hooks');

const obs = new PerformanceObserver((list, observer) => {
  // llamado una vez. la lista contiene tres elementos
});
obs.observe({ entryTypes: ['mark'], buffered: true });

for (let n = 0; n < 3; n++)
  performance.mark(`test${n}`);
```

## Clase: PerformanceObserverEntryList

<!-- YAML
added: v8.5.0
-->

The `PerformanceObserverEntryList` class is used to provide access to the `PerformanceEntry` instances passed to a `PerformanceObserver`.

### performanceObserverEntryList.getEntries()

<!-- YAML
added: v8.5.0
-->

* Devuelve: {PerformanceEntry[]}

Returns a list of `PerformanceEntry` objects in chronological order with respect to `performanceEntry.startTime`.

### performanceObserverEntryList.getEntriesByName(name[, type])

<!-- YAML
added: v8.5.0
-->

* `name` {string}
* `type` {string}
* Devuelve: {PerformanceEntry[]}

Returns a list of `PerformanceEntry` objects in chronological order with respect to `performanceEntry.startTime` whose `performanceEntry.name` is equal to `name`, and optionally, whose `performanceEntry.entryType` is equal to `type`.

### performanceObserverEntryList.getEntriesByType(type)

<!-- YAML
added: v8.5.0
-->

* `type` {string}
* Devuelve: {PerformanceEntry[]}

Returns a list of `PerformanceEntry` objects in chronological order with respect to `performanceEntry.startTime` whose `performanceEntry.entryType` is equal to `type`.

## Ejemplos

### Medir la duración de las operaciones asincrónicas

The following example uses the [Async Hooks](async_hooks.html) and Performance APIs to measure the actual duration of a Timeout operation (including the amount of time it to execute the callback).

```js
'use strict';
const async_hooks = require('async_hooks');
const {
  performance,
  PerformanceObserver
} = require('perf_hooks');

const set = new Set();
const hook = async_hooks.createHook({
  init(id, type) {
    if (type === 'Timeout') {
      performance.mark(`Timeout-${id}-Init`);
      set.add(id);
    }
  },
  destroy(id) {
    if (set.has(id)) {
      set.delete(id);
      performance.mark(`Timeout-${id}-Destroy`);
      performance.measure(`Timeout-${id}`,
                          `Timeout-${id}-Init`,
                          `Timeout-${id}-Destroy`);
    }
  }
});
hook.enable();

const obs = new PerformanceObserver((list, observer) => {
  console.log(list.getEntries()[0]);
  performance.clearMarks();
  observer.disconnect();
});
obs.observe({ entryTypes: ['measure'], buffered: true });

setTimeout(() => {}, 1000);
```

### Medir cuánto tiempo tarda la carga de dependencias

The following example measures the duration of `require()` operations to load dependencies:

<!-- eslint-disable no-global-assign -->

```js
'use strict';
const {
  performance,
  PerformanceObserver
} = require('perf_hooks');
const mod = require('module');

// Monkey patch que requiere función
mod.Module.prototype.require =
  performance.timerify(mod.Module.prototype.require);
require = performance.timerify(require);

// Activar el observer
const obs = new PerformanceObserver((list) => {
  const entries = list.getEntries();
  entries.forEach((entry) => {
    console.log(`require('${entry[0]}')`, entry.duration);
  });
  obs.disconnect();
});
obs.observe({ entryTypes: ['function'], buffered: true });

require('some-module');
```