# moleculer-web-error-handling-middleware

## Summary

When using [error handling middleware](https://moleculer.services/docs/0.13/moleculer-web.html#Error-handler-middleware) as described in the documentation a RequestTimeoutError is raised.
This repo simulates this behavior by applying two middlewares, one that raises an error and one that handles the error.

## How to reproduce

Clone [this repo](https://github.com/designtesbrot/moleculer-web-error-handling-middleware) and `cd` into it. Then
run `docker-compose up` and then `curl http://localhost:3000/api/greeter/hello`.

## Expected

The JSON response returned is:
```json
{
    "name": "Error",
    "message": "Something went wrong",
    "code": 500
}
```
The console log of the api service is:
```log
api_1      | [2018-07-15T10:03:16.897Z] INFO  0e908e290bf5-16/API: => GET /api/greeter/hello
api_1      | [2018-07-15T10:03:16.900Z] ERROR 0e908e290bf5-16/API: Error is occured in middlewares!
api_1      | [2018-07-15T10:03:16.908Z] INFO  0e908e290bf5-16/API: <= 500 GET /api/greeter/hello [+11.965 ms]
api_1      | [2018-07-15T10:03:16.908Z] INFO  0e908e290bf5-16/API: 
```

## Actual Result

The JSON response returned is:
```json
{
    "name": "Error",
    "message": "Something went wrong",
    "code": 500
}
```
The console log of the api service is (after 10 seconds):
```log
api_1      | [2018-07-15T10:03:16.897Z] INFO  0e908e290bf5-16/API: => GET /api/greeter/hello
api_1      | [2018-07-15T10:03:16.900Z] ERROR 0e908e290bf5-16/API: Error is occured in middlewares!
api_1      | [2018-07-15T10:03:16.908Z] INFO  0e908e290bf5-16/API: <= 500 GET /api/greeter/hello [+11.965 ms]
api_1      | [2018-07-15T10:03:16.908Z] INFO  0e908e290bf5-16/API: 
api_1      | [2018-07-15T10:03:26.927Z] WARN  0e908e290bf5-16/BROKER: Request 'api.rest' is timed out. { requestID: null, nodeID: '0e908e290bf5-16', timeout: 10000 }
api_1      | [2018-07-15T10:03:26.932Z] ERROR 0e908e290bf5-16/API:    Request error! RequestTimeoutError : Request is timed out when call 'api.rest' action on '0e908e290bf5-16' node. 
api_1      |  RequestTimeoutError: Request is timed out when call 'api.rest' action on '0e908e290bf5-16' node.
api_1      |     at p.timeout.catch.err (/app/node_modules/moleculer/src/middlewares/timeout.js:33:13)
api_1      |     at tryCatcher (/app/node_modules/bluebird/js/release/util.js:16:23)
api_1      |     at Promise._settlePromiseFromHandler (/app/node_modules/bluebird/js/release/promise.js:512:31)
api_1      |     at Promise._settlePromise (/app/node_modules/bluebird/js/release/promise.js:569:18)
api_1      |     at Promise._settlePromise0 (/app/node_modules/bluebird/js/release/promise.js:614:10)
api_1      |     at Promise._settlePromises (/app/node_modules/bluebird/js/release/promise.js:689:18)
api_1      |     at Async._drainQueue (/app/node_modules/bluebird/js/release/async.js:133:16)
api_1      |     at Async._drainQueues (/app/node_modules/bluebird/js/release/async.js:143:10)
api_1      |     at Immediate.Async.drainQueues (/app/node_modules/bluebird/js/release/async.js:17:14)
api_1      |     at runCallback (timers.js:810:20)
api_1      |     at tryOnImmediate (timers.js:768:5)
api_1      |     at processImmediate [as _immediateCallback] (timers.js:745:5) 
api_1      | Data: { action: 'api.rest', nodeID: '0e908e290bf5-16' }
api_1      | [2018-07-15T10:03:26.933Z] WARN  0e908e290bf5-16/API: Headers have already sent

```

## Deep Dive

The api service makes use of two middlewares [as documented](https://moleculer.services/docs/0.13/moleculer-web.html#Error-handler-middleware):

```javascript
'use strict';

const ApiGateway = require('moleculer-web');

module.exports = {
  name: 'api',
  mixins: [ApiGateway],

  // More info about settings:
  // https://moleculer.services/docs/0.13/moleculer-web.html
  settings: {
    
    // ...

    use: [
        
      // simulate a middleware error
      (req, res, next) => {
        next(new Error('Something went wrong'))
      },
      
      // simulate a error handling middleware
      function (err, req, res, next) {
        this.logger.error('Error is occured in middlewares!');
        this.sendError(req, res, err);
      },
      
    ],
  },
};

```

The first middleware simulates a middleware error (by simply calling `next` with an Error). The second middleware handles the error
by sending an error response. A real world use case for this is when for example
using swagger-tools middleware to perform request validation. This middleware calls `next(RequestValidationError)`. These errors should
be handeled error in the request-response flow (no necessity to call an action). When reading the sources of `moleculer-web` I could
pin the cause for this issue to these segments:

```javascript
// moleculer-web/src/index.js:216

routeHandler(ctx, route, req, res) {
  
    // ...
    
    return this.composeThen(req, res, ...route.middlewares).then(() => {
      
        // ...
        
        return this.aliasHandler(req, res, { action });
    })
},
```

Here the http request is handled by a route. First all middlewares are invoked in a serial promise chain. Upon resolve the
action is resolved and invoked. The issue here is that by definition connect-like middleware can terminate the request early
(by sending a response). This means that `composeThen` in this case is never being resolved. This causes the context to 
raise a TimeOutError.

## Possible solutions:

### Do not suggest to use service middleware for early request termination.

The potentially easiest solution could be to remove the documentation for error handling middleware, or explicitly state
that this is not working and instead the service should be invoked in middleware mode. In this case for example `express` 
can invoke the middlewares prior to a moluculer service context existing. Downside is that with such an implementation to
my understanding some moleculer features like service dependencies or hot reloading could not work.

### Racing against middleware

An alternative could be to do something like:
```javascript
return Promise.race([
    this.composeThen(),
    new Promise((resolve,reject) => {
      setTimeout(3000,() => {
        response.finished ? reject(MiddleWareEarlyTerminationError) : reject(MiddleWareTimeoutError)
      })
    })
]).then(/* ... */)
```
In this case the middleware would race against a timeout. If after the timeout there was no response send to the client,
the promise would be rejected with a `MiddleWareTimeoutError`. If a response has been send, the promise rejects with a 
`MiddleWareEarlyTerminationError`. The http handler could then, when catching on the promise compair the error and if it is a
`MiddleWareEarlyTerminationError` be silent about it. The problem here is that I do not know if at this stage the timeout
to race for can be savely determined. I think this would also be early enough to not interfere with rate limiting and such.
It's a bit hacky of course.

### Adapt architecture

I think the underlying problem is that the moleculer context (broker invoking action) is created to early here. Calling an
action does (and should) allways expect an answer and handle timeouts. However the way middlewares work, this behavior
does not necessarily occur here. I think the correct way would be to have a container running (at least for the global middlewares), 
which could be based on `connect` or sth, and only when the global middlewares passed, the `api.rest` action should actually
be invoked. However I am far to inexperienced with moleculer to judge the feasibility or even correctness of this approach.

