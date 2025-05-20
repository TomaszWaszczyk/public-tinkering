# 💡 Why Public Tinkering

I share my technical learning publicly to deepen my understanding, help others, and connect with the community. Public learning invites feedback, collaboration, and faster growth for everyone involved.

## Hydration

ReactJS hydrates HTML, but what does it mean?

When using server side rendering, the server generates and sends fully rendered HTML to the browser. This allows users to see the content quickly, but initially, this HTML is "static" — it does not respond to user interactions because the JavaScript logic and event handlers are not yet active.

Once the browser loads the necessary JavaScript, React "hydrates" the existing HTML. This means React attaches its event listeners and internal state to the already-rendered DOM nodes, transforming the static page into a fully interactive React application.

Hydration is different from a full client-side render, where React would create the DOM from scratch. Instead, hydration reuses the server-rendered HTML and simply "activates" it with interactivity



## Garbage collector in the browser

Using a `WeakMap` in JavaScript offers several key advantages:

- Automatic Memory Management

- No Manual Cleanup Required: Because entries are automatically removed when the key object becomes unreachable, you don't need to manually delete entries to free up memory.
- Prevents Memory Leaks: Unlike Map, which holds strong references to keys and can keep objects in memory indefinitely, WeakMap allows objects to be garbage collected as soon as they are no longer referenced elsewhere, reducing the risk of unintentional memory retention

![weakmap](./resources/weakmap.png)

Just garbage collector in a web browser ;-)

## IndexedDB vs LocalStorage

IndexedDB - everyone knows LocalStorage but there is better alternative

IndexedDB is ideal for offline mode because it allows web applications to store large amounts of structured data directly in the user's browser, enabling full CRUD operations without an internet connection. This means users can interact with the app and access or modify their data while offline, and any changes can be synchronized with the server once connectivity is restored.


```
let openRequest = indexedDB.open("store", 1);

openRequest.onupgradeneeded = function() {
  // triggers if the client had no database
  // ...perform initialization...
};

openRequest.onerror = function() {
  console.error("Error", openRequest.error);
};

openRequest.onsuccess = function() {
  let db = openRequest.result;
  // continue working with database using db object
};
```
### References

https://javascript.info/indexeddb

https://dexie.org/
