---
_build:
  publishResources: false
  render: never
  list: never
---

Consider the following two Workers, connected via a [Service Binding](/workers/runtime-apis/bindings/service-bindings/rpc). The counter service provides the RPC method `newCounter()`, which returns a function:

```toml
---
filename: wrangler.toml
---
name = "counter-service"
main = "./src/counterService.js"
```

```js
---
filename: src/counterService.js
---
import { WorkerEntrypoint } from "cloudflare:workers";

export default class extends WorkerEntrypoint {
  async fetch() { return new Response("Hello from counter-service"); }

  async newCounter() {
    let value = 0;
    return (increment = 0) => {
      value += increment;
      return value;
    }
  }
}
```

This function can then be called by the client Worker:

```toml
---
filename: wrangler.toml
---
name = "client_worker"
main = "./src/clientWorker.js"
services = [
  { binding = "COUNTER_SERVICE", service = "counter-service" }
]
```

```js
---
filename: clientWorker.js
---
export default {
  async fetch(request, env) {
    using f = await env.COUNTER_SERVICE.newCounter();
    await f(2);   // returns 2
    await f(1);   // returns 3
    const count = await f(-5);  // returns -2

    return new Response(count);
  }
}
```

{{<Aside type="note" description="The `using` declaration">}}
Refer to [Explicit Resource Management](/workers/runtime-apis/rpc/lifecycle) to learn more about the `using` declaration shown in the example above.
{{</Aside>}}

How is this possible? The system is not serializing the function itself. When the function returned by `CounterService` is called, it runs within `CounterService` — even if it is called by another Worker.

Under the hood, the caller is not really calling the function itself directly, but calling what is called a "stub". A "stub" is a [Proxy](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy) object that allows the client to call the remote service as if it were local, running in the same Worker. Behind the scenes, it calls back to the Worker that implements `CounterService` and asks it to execute the function closure that had been returned earlier.