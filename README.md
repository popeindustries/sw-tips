## How to avoid the zombie apocalypse

...or, things I wish I knew before I started

### async/await

understand how promises work, but use `async/await` instead. The ServiceWorker API takes promise use to new extremes, so using `async/await` can help make things way more legible:

```js
async function install(version, assets) {
  const cache = await caches.open(version);
  return cache.addAll(assets);
}
```

Although very few browsers support native `async/await`, it's just syntactic sugar over generators and promises, and every browser that supports ServiceWorker supports async code converted to use generators.

Using Babel with the [async-to-generator](https://babeljs.io/docs/plugins/transform-async-to-generator/) plugin will add minimal extra overhead, but be aware that this code cannot be minified with Uglify-js, which only supports ES5 input source. [Babili](https://github.com/babel/babili) is another plugin for Babel that you can use for minification instead.

### Don't register the ServiceWorker while the page is loading

Bandwidth and cpu time must be shared while the cache is being filled during the ServiceWorker's `installation` phase, so wait for the `window.onload` event (or some other signal) before registering:

```js
if ('serviceWorker' in navigator) {
  window.addEventListener('load', () => {
    navigator.serviceWorker.register('sw.js');
  });
}
```

All about [registration](https://developers.google.com/web/fundamentals/instant-and-offline/service-worker/registration) (Jeff Posnick).

### Know your dependencies

During the `installation` phase, passing a promise to `event.waitUntil` will delay ServiceWorker activation until resolved. However, if rejected, the ServiceWorker will be thrown away and marked `redundant`.

Since the `installation` phase is when you want to pre-cache assets, any asset that fails to load will cause a rejection.

In this sense, pre-cached assets should be considered hard dependencies, so beware!

```js
self.addEventListener('install', event => {
  event.waitUntil(install());
});

async function install() {
  const cache = await caches.open('v1');
  return cache.addAll(ASSETS);
}
```

All about [lifecycle](https://developers.google.com/web/fundamentals/instant-and-offline/service-worker/lifecycle) (Jake Archibald)

### Cache smarter

When upgrading a ServiceWorker, it's common to pre-cache assets in a new, uniquely named cache before deleting old ones during the `activation` phase:

```js
self.addEventListener('activate', event => {
  event.waitUntil(activate());
});

async function activate() {
  const keys = await caches.keys();
  return Promise.all(
    keys.map(key => {
      if (key !== 'v2') return caches.delete(key);
    })
  );
}
```

In most cases, this is a good approach, but if you release often, you can avoid wasting storage space and bandwidth by only fetching new assets and recycling the old ones.

Create more than one cache to separate versioned assets from those that won't change:

```js
self.addEventListener('install', event => {
  event.waitUntil(Promise.all([
    cacheStatic(ASSETS_STATIC),
    cacheVersioned('2', ASSETS_VERSIONED)
  ]));
});

async function cacheStatic(assets) {
  const exists = await caches.has('static');
  if (!exists) {
    const cache = await caches.open('static');
    return cache.addAll(assets);
  }
}
```

And copy existing versioned assets from the old cache, if they already exist:

```js
async function cacheVersioned(version, assets) {
  const exists = await caches.has(`version-${version}`);
  if (!exists) {
    const requests = assets.map(asset => new Request(asset));
    const preCachedResponses =
      await Promise.all(requests.map(req => caches.match(req)));
    const cache = await caches.open(`version-${version}`);
    return Promise.all(requests.map((request, idx) => {
      return preCachedResponses[idx]
        ? cache.put(request, preCachedResponses[idx].clone())
        : cache.add(request);
    }));
  }
}
```

### Avoid forcing activation for major changes

Forcing activation after an update can break already connected clients if the new ServiceWorker behaves very differently from the old one.

Avoid calling `self.skipWaiting()` after a major change, and consider prompting the user to trigger a refresh instead:

```js
// ServiceWorker
self.addEventListener('install', event => {
  event.waitUntil(install());
});

async function install() {
  // Install stuff, then...
  const allClients = await clients.matchAll();
  allClients.forEach(client => {
    client.postMessage({ type: 'update' });
  });
}

// Browser client
navigator.serviceWorker.addEventListener('message', (event) => {
  if (event.data.type == 'update') {
    // Show interactive prompt and trigger page refresh
  }
});
```

### Use a library for messaging

Sending messages between a ServiceWorker and it's clients can be a little unintuitive if you haven't worked with the `postMessage` API before. Using a messaging library like [Swivel](https://github.com/bevacqua/swivel) can help:

```js
const swivel = require('swivel');

// Handle messages from ServiceWorker
swivel.on('data', (context, ...data) => {
  // Handle data
});

// Send message to ServiceWorker
swivel.emit('data', ...data);

// Send message to clients
swivel.broadcast('data', ...data);
```

### Never rename the ServiceWorker script file

Once a ServiceWorker has been installed and activated, it will need to be updated. If the html file that registers the ServiceWorker is itself cached, it will be difficult to install a new ServiceWorker with a different name.

Avoid this chicken-and-egg problem by making sure the ServiceWorker script filename is never unique:

```js
// Don't
navigator.serviceWorker.register('sw-v1.js');
// Do
navigator.serviceWorker.register('sw.js');
```

### Set correct cache headers

If ServiceWorker script filenames are static, and the browser fetches the script from the browser cache before going to the network, you will need to correctly set `cache-control` headers to prevent the browser from caching outdated versions.

Use `no-cache` or `max-age=0` to always fetch from the network, or a `max-age` of a few minutes (at most) to benefit from client/edge caching offload (browser, CDN, etc):

```txt
# never cache
cache-control: max-age=0
# cache for 2 min
cache-control: max-age=120
```

As a precaution, to avoid accidentally installing a ServiceWorker for days/weeks/months, caches will be bypassed if the script is older than 24 hours, regardless of what you set.

Cache invalidation is always tricky, so in the future, browsers will use "cache busting" by default to ensure that ServiceWorker script files are always kept up-to-date.

More on [updating](http://stackoverflow.com/questions/38843970/service-worker-javascript-update-frequency-every-24-hours/38854905#38854905) (Jeff Posnick)

### Invalidate your ServiceWorker when updated

The ServiceWorker will be re-installed if it is byte different from the previous version. A simple setup is to treat the ServiceWorker file as a boot loader by using `importScripts` with versioned files:

```js
// sw.js
self.importScripts('vendor-sw-v1', 'index-sw-v1');
// ...that's all you need!
```

### Add a feature flag/kill switch

In the event of disaster, having an easy way to disable existing ServiceWorkers can be a lifesaver. Add a feature flag to control unregistration:

```js
if ('serviceWorker' in navigator) {
  window.addEventListener('load', () => {
    if (features.serviceWorker) {
      navigator.serviceWorker.register('sw.js');
    } else {
      navigator.serviceWorker.getRegistrations()
        .then((registrations) => {
          registrations.forEach((registration) => {
            registration.unregister();
          });
        });
    }
  });
}
```

...keep a no-op ServiceWorker handy for quick deploy:

```js
self.addEventListener('install', event => {
  self.skipWaiting();
});

self.addEventListener('activate', (event) => {
  // Delete caches...
  event.waitUntil(clients.claim());
});

// No 'fetch' handler
```

...or have the ServiceWorker phone home to check it's version, then force an update if outdated:

```js
  versionCheck();

  async function versionCheck() {
    const response = await fetch(SW_VERSION_URL);
    const version = await response.text();
    if (version !== VERSION) {
      self.registration.update();
    }
  }
```

More on [kill switches](http://stackoverflow.com/questions/33986976/how-can-i-remove-a-buggy-service-worker-or-implement-a-kill-switch/38980776#38980776) (Jeff Posnick)

### Don't cache bad responses

Always check the `ok` property of the response object returned from `fetch()` before you add it to your cache. HTTP error response codes (4xx, 5xx) won't cause the promise to reject:

```js
async function onFetch(event) {
  try {
    const response = await fetch(event.request);
    if (response.ok) {
      // Cache it
    } else {
      throw Error(`error fetching with ${response.status}`);
    }
  } catch (error) {
    // Handle response error
  }
}
```

### Don't store global state

Storing global state in a ServiceWorker is bad. Code outside of event handlers is run each time a ServiceWorker is started, but they're stopped and started many times over their lifetime in order to save battery and other resources, and that global state will be destroyed at unexpected times:

```js
// Don't do this!
let db;

self.addEventListener('install', event => {
  // Open database connection
  db = openDB();
});
self.addEventListener('fetch', event => {
  event.respondWith(
    db.readStuff().then(/*...*/)
  );
});
```

More about the risks [here](http://stackoverflow.com/questions/38835273/when-does-code-in-a-service-worker-outside-of-an-event-handler-run/38835274#38835274) (Jeff Posnick)

### Guard against missing APIs

A number of API methods were added in later browser versions, so it's wise to test whether they exist before calling them:

```js
if (self.skipWaiting) {
  self.skipWaiting();
}
```

The following methods were added in later versions of Chrome, after ServiceWorker was launched:

```js
// Chrome 42
self.skipWaiting()
clients.claim()

// Chrome 46
cache.add()
cache.addAll()

// Chrome 47
cache.matchAll()
```

### Test your ServiceWorker

Because of the installation lifecycle and the special environment they run in, ServiceWorkers are very difficult to test. As always, running tests in real browsers, with real code, will give the most realistic results.

Unfortunately, there aren't yet any good tools to help with browser tests, but the methodology is well laid out in this [article](https://medium.com/dev-channel/testing-service-workers-318d7b016b19) by Matt Gaunt of Google.

Automating browser tests comes with it's own set of challenges, so it's often desirable to test as much as possible with lightweight unit tests. Fortunately, there are some tools available to easily mock and test the ServiceWorker environment.

As part of their [service-workers](https://github.com/pinterest/service-workers) toolchain, Pinterest has developed helper functions and a mock you can use to make the Node.js global scope look like a ServiceWorker:

```js
const makeServiceWorkerEnv = require('service-worker-mock');

describe('ServiceWorker', () => {
  beforeEach(() => {
    Object.assign(global, makeServiceWorkerEnv());
    jest.resetModules();
  });
  it('should add listeners', () => {
    require('sw.js');
    expect(self.listeners['install']).toBeDefined();
    expect(self.listeners['activate']).toBeDefined();
    expect(self.listeners['fetch']).toBeDefined();
  });
});
```

I also released a project for testing ServiceWorkers in Node.js called  [sw-test-env](https://github.com/YR/sw-test-env). It's a little more thorough mock of the ServiceWorker specification, and allows you to run ServiceWorker code in an isolated, sandboxed context:

```js
const { connect, destroy } = require('sw-test-env');
let sw;

describe('ServiceWorker', () => {
  beforeEach(() => {
    sw = connect();
  });
  afterEach(destroy);
  it('should add listeners', async () => {
    await sw.register('sw.js');
    expect(sw.scope._listeners['install']).toBeDefined();
    expect(sw.scope._listeners['activate']).toBeDefined();
    expect(sw.scope._listeners['fetch']).toBeDefined();
  });
});
```

With it you can:

- inspect the properties of the ServiceWorker scope (`clients`, `caches`, `registration`, etc)
- manually trigger events (`install`, `activate`, `fetch`, etc)
- `postMessage` between clients and ServiceWorker instances
- use `importScripts()`
- `fetch()` real (or mocked) data
- use `indexedDB` storage
- `require()` modules without a build step

```js
it('should recycle assets on upgrade', async () => {
  await sw.register('./fixtures/cache-smarter.js');
  // Fill existing versioned cache
  const cache = await sw.scope.caches.open('version-1');
  await cache.put(new Request('bar.js'),
    new Response('bar'));
  await sw.trigger('install');
  const bar =
    await sw.scope.caches.match(new Request('bar.js'));
  const body = await bar.text();

  expect(bar.ok).to.equal(true);
  expect(bar.status).to.equal(200);
  expect(body).to.equal('bar');
});
```

### Use a ServiceWorker generator tool

If you don't want to get your hands dirty with the details, you can use one of several ServiceWorker generator tools and libraries:

- [sw-precache](https://github.com/GoogleChrome/sw-precache): build tool to generate a sw.js file with pre-cached assets
- [sw-toolbox](https://github.com/GoogleChrome/sw-toolbox): library for runtime caching
- [generate-service-worker](https://github.com/pinterest/service-workers/tree/master/packages/generate-service-worker): build tool to generate sw.js and handle runtime caching
- [offline-plugin](https://github.com/NekR/offline-plugin/): Webpack plugin to generate sw.js