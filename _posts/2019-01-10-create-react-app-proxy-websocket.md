---
layout: post
title: proxy websocket requests in development with create-react-app
published: true
---

this is inspired from [https://facebook.github.io/create-react-app/docs/proxying-api-requests-in-development#configuring-the-proxy-manually] 


if you need to proxy your api call in development, it is very easy: just add the following in your package.json

```raw
proxy: "http://myserver"
```

so that calls to /api will be proxied to http://myserver in development. For example, in your react code you will have



```javascript
export const getUsers = () => {
    return fetch('/api/getUsers', { method: 'GET' })
    .then(response => { ...
```

in this example, the call localhost:3000/api/getUsers will be proxied to http://myserver/api/getUsers

if you need to proxy websockets as well, then there is also an easy solution:
* remove the proxy setting from the package.json
* add a file src/setupProxy.js with the following

```javascript
const proxy = require('http-proxy-middleware');

module.exports = function(app) {
  app.use(proxy('/api', { target: 'http://myserver' }));
  app.use(proxy('/ws', { target: 'ws://myserver', ws: true }));
};

```
in your React code nothing change for the /api calls, just use the URL /api. For the websockets request, it is a bit more involved, 
here is how I do it

```javascript
  componentDidMount() {
     const protocolPrefix = window.location.protocol === 'https:' ? 'wss:' : 'ws:';
     let { host } = window.location; // nb: window location contains the port, so host will be localhost:3000 in dev
     this.ws = new WebSocket(`${protocolPrefix}//${host}/ws/dbstates`); // dbstates is my websocket route
     this.ws.onopen = () => {
       ...
  }
```
so that when the url is localhost:3000/ws/dbstates, it will be proxied based on the setup in src/setupProxy.js

Another solution is to use an environment variable to change the websocket url in development. This can be used in combination with the previous solution, my code would then be

```javascript
const protocolPrefix = window.location.protocol === 'https:' ? 'wss:' : 'ws:';
let { host } = window.location; // nb: window location contains the port, so host will be localhost:3000 in dev
if (process.env.NODE_ENV === 'development' && process.env.REACT_APP_WSPROXYIP) {
  host = process.env.REACT_APP_WSPROXYIP;
  if (process.env.REACT_APP_WSPROXYPORT) {
    host += `:${process.env.REACT_APP_WSPROXYPORT}`;
  } else {
    const { port } = window.location;
    host += `:${port}`;
  }
}
this.ws = new WebSocket(`${protocolPrefix}//${host}/ws/dbstates`); // dbstates is my websocket route
```


