# notes

## TODO

- look up create-react-app and react repos for "service workers"

Cache is King 💵: A Practical Introduction to Service Workers in React
Veekas Shrivastava (https://twitter.com/veekas)

Cache in on this opportunity to learn how to create a rich offline experience for your users with the use of service workers. We'll go over some best practices for offline caching and learn how to implement them with Workbox.

## resources

- example apps: https://hnpwa.com
- workbox tool: https://developers.google.com/web/tools/workbox
- great starbucks example, talk about desktop apps: https://www.youtube.com/watch?v=NITk4kXMQDw

## questions

- Pros and Cons of using offline storage rather than cached results via service workers... One big question I have about PWAs can I get away with just a manifest w/o any service workers and then write my own Proxies for handling offline calls, i.e just manually redirect them to my local storage???

## SBO - Architecting Web Apps to 'Just Work' Offline, Islam Sharabash, @ibashes

- offline is a big deal, don't lose data!!!
- simple, robust (errors should be rare and recoverable), fast
- modifier, queue, worker architecture
  - modifier: uptimistic UI update, rollback
    - rollback: if a modifier doesn't work, then roll back to source of truth in cache
  - queue: keep modifiers ordered, add durability
  - worker: sync, error handling
    - persist must be idempotent (safe to retry), meaning backend must give idempotent API
- conflict resolution
  - last write wins OR
  - resolve conflict on the client
- they use webSQL
- localStorage events for multi tab -> broadcast channel

## Youtube - Workbox: Flexible PWA Libraries (Chrome Dev Summit 2017) @jeffposnick

- SW sits between app and network
- the way a SW handles request is *strategy*
- add a cache storage api
- powerful! but complex =(
  - makes it fast and reliable
- Workbox
  - new iteration of `sw-precache` and `sw-toolbox`
  - focus on config, not boilerplate
  - used by Pinterest which has a heavy focus on PWA
  - WIRED uses workboxcli to generate new SW
    - three types of images: cache-first, runtime cache and expire
  - `workbox-webpack-plugin`
  - background caching is great for large pictures or video
- people to reach out to
  - addy osmani
  - jeffposnick
  - will farley (non google but doing the webpack plugin)
  - kayce basques
  - matt gaunt
  - philip walton
  - prateek bhatnagar

## SBO - Embracing Offline First

- "It is time to stop treating a loss of connectivity as an error state in our web apps. Offline and poor connectivity are inevitable states in our apps—ones we must plan for."
- "Offline-first is about accepting a simple truth: offline and low connectivity conditions are inevitable. These conditions should be treated not as a catastrophic failure, but as just another possible state in the life of your web app. A state you should plan for and handle gracefully."
- example usage:
  1. Return index.html using the cache, falling back to network with frequent updates pattern.
  2. Return all static files required to display the home page using the cache, falling back to network pattern.
  3. Return the Google Maps JavaScript file from the network. If the request fails, return an alternate script that works offline.
  4. Return the events.json file using the network, falling back to cache with frequent updates pattern.
  5. Return event image files using the cache on demand pattern, falling back to a default generic image if the network isn’t available and the image isn’t cached.
  6. Let analytics pings go through untouched.
- app shell: minimal layout HTML/CSS/JS to render meaningful view
- caching patterns
  - **Cache only**
    - resources typically downloaded as SW dependency
    - do not change within lifetime of single version of the app

    ```js
    self.addEventListener("fetch", function(event) {
      event.respondWith(
        caches.match(event.request)
      );
    });
    ```

  - **Cache, falling back to network**
    - check cache and only if not there, load from network (e.g. weather icons)
    - upside: fast, bandwidth efficient
    - downside: update SW with new HTML file, won't be shown until next visit

    ```js
    self.addEventListener("fetch", function(event) {
      event.respondWith(
        caches.match(event.request)
          .then(function(response) {
            return response || fetch(event.request);
          });
      );
    });
    ```

    - **then... caching on demand**

      ```js
      self.addEventListener("fetch", function(event) {
        event.respondWith(
          caches.open("cache-name").then(function(cache) {
            return cache.match(event.request)
              .then(function(cachedResponse) {
                return cachedResponse || fetch(event.request).then(function(networkResponse) {
                  cache.put(event.request, networkResponse.clone()); // clone() is important here
                  return networkResponse;
                });
            });
          })
        );
      });
      ```

    - **then... with frequent updates**
      - use case: resources changes regularly but it's more important to load fast than to be most up-to-date
      - upside: fast, sees latest file
      - downside: uses same bandwith as "network falling back to cache"

      ```js
      self.addEventListener("fetch", function(event) {
        event.respondWith(
          caches.open("cache-name").then(function(cache) {
            return cache.match(event.request).then(function(cachedResponse) {
              var fetchPromise =
                fetch(event.request).then(function(networkResponse) {
                  cache.put(event.request, networkResponse.clone());
                  return networkResponse;
                });
              return cachedResponse || fetchPromise;
            })
          })
        );
      });
      ```

  - **Cache, *then* network**
    - display from cache immediately while checking the network, then compare and update if newer
    - "best of both worlds"? maybe, but introduces challenges:
      1. write two requests, display cache, and update
      2. UX challenges: how to manage data the user is editing before network request comes back (e.g. field editing like Google Docs)
  - **Network, falling back to cache**
    - load from network, but if offline, load from cache (e.g. latest weather), if that fails, request fails
    - upside: always latest version
    - downside: missing out on chance to improve load time

    ```js
    self.addEventListener("fetch", function(event) {
      event.respondWith(
        fetch(event.request).catch(function() {
          return caches.match(event.request);
        })
      );
    });
    ```

  - **then... with frequent updates**
    - f

    ```js
    self.addEventListener("fetch", function(event) {
      event.respondWith(
        caches.open("cache-name").then(function(cache) {
          return fetch(event.request).then(function(networkResponse) {
            cache.put(event.request, networkResponse.clone());
            return networkResponse;
          }).catch(function() {
            return caches.match(event.request);
          });
        })
      );
    });
    ```

  - **Network only**
    - use for things you never want to cache (e.g. network pings)
    - rarely will need to do this within a SW, but this is an example:

    ```js
    self.addEventListener("fetch", function(event) {
      event.respondWith(
        fetch(event.request)
      );
    });
    ```

  - **Generic fallbacks**
    - gracefully handle failure
    - example - user avatar: use default image instead of broken image link
    - e.g. network, falling back to cache, falling back to generic pattern:

    ```js
      self.addEventListener("fetch", function(event) {
        event.respondWith(
          fetch(event.request).catch(function() {
            return caches.match(event.request).then(function(response) {
              return response || caches.match("/generic.png");
            });
          })
        );
      });
    ```

### Twitter PWA example

- 2.5% of Android app size
- uses less battery
- loads faster

## SBO - Progressive Web Application Development by Majid Hajian

### what is a PWA

  - progressively enhance to get them to behave like native apps
  - why? web is fast enough, less bandwidth needed, cross-platform, works for every user ("progressive"), easy to share
  - service workers are the most important part of PWA

#### core concepts

- progressive: works for everyone
- responsive
- secure: HTTPS
- connectivity independent: service worker
- installable
- app-like
- re-engageable
- linkable and discoverable

#### PRPL pattern

- PUSH critical resources for initial url route
- RENDER initial route
- PRE-CACHE remaining routes
- LAZY-LOAD and create remaining reroutes on demand

## Learn ECMAScript - Second Edition, chapter on "The service worker lifecycle"

- how the service worker lives
  - ![sw-diagram](/OReilly/Learn-ECMAScript/sw-lifecycle-diagram.png)
- step zero: check if browser supports SW

  ```js
    if('serviceWorker' in navigator) {
        // service worker available
        // lets code
    }
  ```

- step one: install (register)
  - `reg` object is the info associated with the sw
  - can be safely run multiple times (browser will not re-register)

  ```js
    navigator.serviceWorker.register('/sw.js')
      .then(reg => console.log(reg))
      .catch(err => console.log(err));
  ```

- step two: trigger events to handle different things
  - install event
    - optional caching

  ```js
    self.addEventListener('install', e => {
        e.waitUntil(async function() {
            const cache = await caches.open('cacheArea');
            await cache.addAll(['/', '/styles/main.css', '/styles/about.css']);
        }());
    });
  ```

  - fetch data

  ```js
    self.addEventListener('fetch', e => {
      e.respondWith(async function() {
        const response = await caches.match(e.request);
        if(response) {
            return response;
        }
        return fetch(e.request);
      }());
    });
  ```

- debug with Chrome DevTools (inspect -> service workers)
- sw scope matters

  ```js

  ```
