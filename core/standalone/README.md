<!--
#
# Licensed to the Apache Software Foundation (ASF) under one or more
# contributor license agreements.  See the NOTICE file distributed with
# this work for additional information regarding copyright ownership.
# The ASF licenses this file to You under the Apache License, Version 2.0
# (the "License"); you may not use this file except in compliance with
# the License.  You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#
-->

# OpenWhisk Standalone Server

OpenWhisk standalone server is meant to run a simple OpenWhisk server for local development and test purposes. It can be
executed as a normal java application from command line.

```bash
java -jar openwhisk-standalone.jar
```

This should start the OpenWhisk server on port 3233 by default. Once the server is started then [configure the cli][1]
and then try out the [samples][2].

This server by default uses a memory based store and does not depend on any other external service like Kafka and CouchDB.
It only needs Docker and Java to for running.

Few key points related to it

* Uses in memory store. Once the server is stopped all changes would be lost
* Bootstraps the `guest` and `whisk.system` with default keys
* Supports running on MacOS, Linux and Windows (experimental) setup
* Can be customized to use any other storage like CouchDB


### Build

To build this standalone server run

```bash
$ ./gradlew :core:standalone:build
```

This would create the runnable jar in `bin/` directory. If you directly want to run the
server then you can use following command

```bash
$ ./gradlew :core:standalone:bootRun
```

To pass argument to the run command use

```bash
$ ./gradlew :core:standalone:bootRun --args='-m runtimes.json'
```

###  Usage

OpenWhisk standalone server support various launch options

```
$ java -jar openwhisk-standalone.jar -h


       /\   \    / _ \ _ __   ___ _ __ | |  | | |__ (_)___| | __
  /\  /__\   \  | | | | '_ \ / _ \ '_ \| |  | | '_ \| / __| |/ /
 /  \____ \  /  | |_| | |_) |  __/ | | | |/\| | | | | \__ \   <
 \   \  /  \/    \___/| .__/ \___|_| |_|__/\__|_| |_|_|___/_|\_\
  \___\/ tm           |_|

      --api-gw                     Enable API Gateway support
      --api-gw-port  <arg>         Api Gateway Port
      --clean                      Clean any existing state like database
  -c, --config-file  <arg>         application.conf which overrides the default
                                   standalone.conf
      --couchdb                    Enable CouchDB support
  -d, --data-dir  <arg>            Directory used for storage
      --dev-mode                   Developer mode speeds up the startup by
                                   disabling preflight checks and avoiding
                                   explicit pulls.
      --disable-color-logging      Disables colored logging
      --kafka                      Enable embedded Kafka support
      --kafka-docker-port  <arg>   Kafka port for use by docker based services.
                                   If not specified then 9091 or some random
                                   free port (if 9091 is busy) would be used
      --kafka-port  <arg>          Kafka port. If not specified then 9092 or
                                   some random free port (if 9092 is busy) would
                                   be used
      --kafka-ui                   Enable Kafka UI
  -m, --manifest  <arg>            Manifest json defining the supported runtimes
  -p, --port  <arg>                Server port
      --user-events                Enable User Events along with Prometheus and
                                   Grafana
  -v, --verbose
      --zk-port  <arg>             Zookeeper port. If not specified then 2181 or
                                   some random free port (if 2181 is busy) would
                                   be used
  -h, --help                       Show help message
      --version                    Show version of this program

OpenWhisk standalone server


```

Sections below would illustrate some of the supported options

To change the default config you can provide a custom `application.conf` file via `-c` option. The application conf file
must always include the default `standalone.conf`

```hocon
include classpath("standalone.conf")

whisk {
  //Custom config
}
```

Then pass this config file

```bash
java -jar openwhisk-standalone.jar -c custom.conf
```

#### Adding custom namespaces

If you need to register custom namespaces (aka users) then you can pass them via config file like below

```hocon
include classpath("standalone.conf")

whisk {
  users {
    whisk-test = "cafebabe-cafe-babe-cafe-babecafebabe:007zO3xZCLrMN6v2BKK1dXYFpXlPkccOFqm12CdAsMgRU4VrNZ9lyGVCGuMDGIwP"
  }
}
```

Then pass this config file via `-c` option. You can check the users created from log

```
[2019-06-21T19:52:02.923Z] [INFO] [#tid_userBootstrap] [StandaloneOpenWhisk] Created user [guest]
[2019-06-21T19:52:03.008Z] [INFO] [#tid_userBootstrap] [StandaloneOpenWhisk] Created user [whisk.system]
[2019-06-21T19:52:03.094Z] [INFO] [#tid_userBootstrap] [StandaloneOpenWhisk] Created user [whisk.test]
```

#### Using custom runtimes

To use custom runtime pass the runtime manifest via `-m` option

```json
{
  "runtimes": {
    "ruby": [
      {
        "kind": "ruby:2.5",
        "default": true,
        "deprecated": false,
        "attached": {
          "attachmentName": "codefile",
          "attachmentType": "text/plain"
        },
        "image": {
          "prefix": "openwhisk",
          "name": "action-ruby-v2.5",
          "tag": "latest"
        }
      }
    ]
  }
}
```

The pass this file at launch time

```bash
java -jar openwhisk-standalone.jar -m custom-runtime.json
```

You can then see the runtime config reflect in `http://localhost:3233`

#### Using CouchDB

If you need to use CouchDB then you can launch the standalone server with `--couchdb` option. This would launch
a CouchDB server which would configured to store files in user home directory under `.openwhisk/standalone` folder.

If you need to connect to external CouchDB or any other supported artifact store then you can pass the required config

```hocon
include classpath("standalone.conf")

whisk {
  couchdb {
    protocol = "http"
    host = "172.17.0.1"
    port = "5984"
    username = "whisk_admin"
    password = "some_passw0rd"
    provider = "CouchDB"
    databases {
      WhiskAuth = "whisk_local_subjects"
      WhiskEntity = "whisk_local_whisks"
      WhiskActivation = "whisk_local_activations"
    }
  }
}
```

Then pass this config file via `-c` option.

#### Using Api Gateway

Api Gateway mode can be enabled via `--api-gw` flag. In this mode upon launch a separate container for [OpenWhisk Api gateway][3]
would be launched on port `3234` (can be changed with `--api-gw-port`). In this mode you can make use of the
[api gateway][4] support.

#### Using Kafka

Standalone OpenWhisk supports launching an [embedded kafka][5]. This mode is mostly useful for developers working on OpenWhisk
implementation itself.

```
java -jar openwhisk-standalone.jar --kafka
```

It also supports launching a Kafka UI based on [Kafdrop 3][6] which enables seeing the topics created and structure of messages
exchanged on those topics.

```
java -jar openwhisk-standalone.jar --kafka --kafka-ui
```

By default the ui server would be accessible at http://localhost:9000. In case 9000 port is busy then a random port would
be selected. To find out the port look for message in log like below (or grep for `whisk-kafka-drop-ui`)

```
[ 9092  ] localhost:9092 (kafka)
[ 9092  ] 192.168.65.2:9091 (kafka-docker)
[ 2181  ] Zookeeper (zookeeper)
[ 9000  ] http://localhost:9000 (whisk-kafka-drop-ui)
```

#### User Events

Standalone OpenWhisk supports emitting [user events][7] and displaying them via Grafana Dashboard. The metrics are
consumed by [User Event Service][8] which converts them into metrics consumed via Prometheus.

```
java -jar openwhisk-standalone.jar --user-events
```

This mode would launch an embedded Kafka, User Event service, Prometheus and Grafana with preconfigured dashboards.

```
Launched service details

[ 9092  ] localhost:9092 (kafka)
[ 9091  ] 192.168.65.2:9091 (kafka-docker)
[ 2181  ] Zookeeper (zookeeper)
[ 3235  ] http://localhost:3235 (whisk-user-events)
[ 9090  ] http://localhost:9090 (whisk-prometheus)
[ 3000  ] http://localhost:3000 (whisk-grafana)
```

[1]: https://github.com/apache/incubator-openwhisk/blob/master/docs/cli.md
[2]: https://github.com/apache/incubator-openwhisk/blob/master/docs/samples.md
[3]: https://github.com/apache/incubator-openwhisk-apigateway
[4]: https://github.com/apache/incubator-openwhisk/blob/master/docs/apigateway.md
[5]: https://github.com/embeddedkafka/embedded-kafka
[6]: https://github.com/obsidiandynamics/kafdrop
[7]: https://github.com/apache/openwhisk/blob/master/docs/metrics.md#user-specific-metrics
[8]: https://github.com/apache/openwhisk/blob/master/core/monitoring/user-events/README.md
