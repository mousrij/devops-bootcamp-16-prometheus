## Demo Project - Monitoring Own Applications

### Topics of the Demo Project
Configure Monitoring for Own Application

### Technologies Used
- Prometheus
- Kubernetes
- Node.js
- Grafana
- Docker
- Docker Hub

### Project Description
- Configure our NodeJS application to collect & expose Metrics with Prometheus Client Library 
- Deploy the NodeJS application, which has a metrics endpoint configured, into Kubernetes cluster 
- Configure Prometheus to scrape this exposed metrics and visualize it in Grafana Dashboard


#### Steps to configure our NodeJS application to collect & expose metrics
In order to monitor our own application we need to use the Prometheus client library within our application. There are client libraries for different programming languages like Java, Python, JavaScript (Node.js), etc. The provide an abstract interface to expose your metrics and implement the various Prometheus metric types like 'Counter', 'Gauge' 'Histogram' or 'Summary'.

The sample application we want to monitor is the Node.js application in the folder `node-project/app`.

We want to provide the following two metrics:
- the number of requests
- the duration of requests

A Prometheus client library for NodeJS applications can be found here: [https://www.npmjs.com/package/prom-client](https://www.npmjs.com/package/prom-client). So add the following dependency to the `package.json` file:
```json
"prom-client": "14.2.0"
```

In the `server.js` file replace the code
```js
...
let app = express();

app.get('/', function (req, res) {
  res.sendFile(path.join(__dirname, "index.html"));
});
...
```

with the following:
```js
...
let app = express();

const client = require('prom-client');
const collectDefaultMetrics = client.collectDefaultMetrics;
// probe every 5th second
collectDefaultMetrics({ timeout: 5000 });

const httpRequestsTotal = new client.Counter({
  name: 'http_request_operations_total',
  help: 'Total number of Http requests'
});

const httpRequestDurationSeconds = new client.Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of Http requests in seconds',
  buckets: [0.1, 0.5, 2, 5, 10]
});

app.get('/metrics', async (req, res) => {
  res.set('Content-Type', client.register.contentType);
  res.end(await client.register.metrics());
});

app.get('/', function (req, res) {
  // simulate sleep for a random number of milliseconds
  var start = new Date();
  var simulateTime = Math.floor(Math.random() * (10000 - 500 + 1) + 500);

  setTimeout(function(argument) {
    // simulate execution time
    var end = new Date() - start;
    httpRequestDurationSeconds.observe(end / 1000); // convert to seconds
  }, simulateTime);

  httpRequestsTotal.inc();

  // do the real work
  res.sendFile(path.join(__dirname, "index.html"));
});
...
```

`Counter` is a cumulative metric, whose value can only increase. Resets to zero when the process restarts.

`Histogram` samples observations and counts them in configurable buckets. Tracks sizes and frequences of events.

So, as you see, as a developer you have to define the metrics and track the values in your application logic.

To test the metrics, execute the following commands:
```sh
cd node-project/app
npm install
npm start
# > bootcamp-node-project@1.0 start
# > node server.js
# 
# app listening on port 3000!
```

 Open the browser and navigate to 'http://localhost:3000'. Press the refresh button several times. Then navigate to 'http://localhost:3000/metrics'. You should see a bunch of default metrics and at the bottom our two custom metrics.

#### Steps to deploy the NodeJS application into Kubernetes cluster 


#### Steps to configure Prometheus to scrape the metrics and visualize them in Grafana

