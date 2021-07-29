![Kogito](./docsimg/kogito.png)

# kogito-knative-loan-demo

## Ambition

Deploy a business application using Kogito resources.

We will :

- deploy 2 business/kogito services that represent the eligibility and scoring parts of a loan  workflow.

- invoke services 
  *  the first service  `eligibility` can be invoked by an http call using cloudevent specification or a push of a cloudevent on the `eligibilityapplication` kafka topic.then the eligibility service will produce another cloudevent on the `notationapplication` kafka topic, the cloudevent will trigger the notation service, a score and an offer are produced in `resultoffer` kafka topic    




## Deploy on openshift
- Please install : 
  - oc cli : https://docs.openshift.com/container-platform/4.7/cli_reference/openshift_cli/getting-started-cli.html
  - kn cli : https://docs.openshift.com/container-platform/4.7/serverless/installing-kn.html
  - kogito cli : https://docs.jboss.org/kogito/release/latest/html_single/#proc-kogito-operator-and-cli-installing_kogito-deploying-on-openshift
  - Infinispan, AMQ Streams (or Strimzi) and Kogito operators through OperatorHub


* connect to Openshift server

  ```sh
  oc login https://ocp-url:6443 -u login -p password
  ```



* Create a new namespace
  ```sh
    export PROJECT=loan
    oc new-project $PROJECT
  ```
* Install Openshift Serverless and knative-serving 

  * install openshift-serverless operator from OperatorHub

    https://docs.openshift.com/container-platform/4.7/serverless/admin_guide/installing-openshift-serverless.html#installing-openshift-serverless

  * create a knative-serving and knative-eventing instance
    ```sh
    ./manifest/scripts/knative-serving.sh
    ./manifest/scripts/knative-eventing.sh
    oc label namespace loan bindings.knative.dev/include=true 
    ```

  * create the knative bkoker
    ```sh
    oc apply -f manifest/services/keventing/infra/broker.yml
    ```
  * add the event-display service to follow the cloud native events 
    ```sh
    oc apply -f manifest/services/keventing/kogito-services/event-display-service.yml
    ```
  * deploy a trigger to run event-display service to log all events
   
    ```sh
    oc apply -f ../manifest/services/keventing/trigger/display-all-events-trigger
    ```
    
* Deploy Infinispan instance
  ```sh
  cat << EOF | oc apply -f -
  apiVersion: infinispan.org/v1
  kind: Infinispan
  metadata:
    name: kogito-infinispan
    namespace: loan
  spec:
    container:
      memory: 512Mi
    image: 'quay.io/infinispan/server:11.0.7.Final'
    logging: {}
    replicas: 1
  EOF
  ```
* Deploy Kafka instance
```sh
  cat <<EOF | oc apply -f -
    apiVersion: kafka.strimzi.io/v1beta2
    kind: Kafka
    metadata:
      name: kogito-kafka
      namespace: loan
    spec:
      entityOperator:
        topicOperator: {}
        userOperator: {}
      kafka:
        config:
          inter.broker.protocol.version: '2.7'
          log.message.format.version: '2.7'
          offsets.topic.replication.factor: 3
          transaction.state.log.min.isr: 2
          transaction.state.log.replication.factor: 3
        listeners:
          - name: plain
            port: 9092
            tls: false
            type: internal
          - name: tls
            port: 9093
            tls: true
            type: internal
        replicas: 3
        storage:
          type: ephemeral
        version: 2.7.0
      zookeeper:
        replicas: 3
        storage:
          type: ephemeral
    EOF
  ```
* Deploy kogito infra and notation service

  * deploy the kogito infra 
    ```sh
    cd .. 
    # install  kogito knative infra
    oc apply -f ./manifest/services/keventing/infra/kogito-knative-infra.yml -n $PROJECT
    # install  kogito kafka infra
    kogito install infra kogito-kafka-infra --kind Kafka --apiVersion kafka.strimzi.io/v1beta2 --resource-name kogito-kafka
    # install infinispan infra
    kogito install infra kogito-infinispan-infra --kind Infinispan --apiVersion infinispan.org/v1 --resource-name kogito-infinispan 
    ```

  * deploy eligibility  service
    ```sh
    #create the service through kogito operator 
    cd eligibility
    kogito deploy-service eligibility --infra kogito-infinispan-infra  --infra kogito-kafka-infra --infra kogito-knative-infra
    mvn clean package  -DskipTests=true
    oc start-build eligibility --from-dir=target 
    ```
    add the following properties to the eligibility-properties configmap

    ```sh
        # edit the configmap
        oc edit cm/eligibility-properties
        #add the following properties after the `application.properties:` line
        org.redhat.bbank.eligibility.rest.CompaniesRemoteService/mp-rest/url=http://companies-svc-companies-svc.apps.ocp4.ouachani.org
        org.redhat.bbank.eligibility.rest.CompaniesRemoteService/mp-rest/scope=javax.enterprise.context.ApplicationScoped

        mp.messaging.incoming.kogito_incoming_stream.connector=smallrye-kafka
        mp.messaging.incoming.kogito_incoming_stream.topic=eligibilityapplication
        mp.messaging.incoming.kogito_incoming_stream.value.deserializer=org.apache.kafka.common.serialization.StringDeserializer

        mp.messaging.outgoing.kogito_outgoing_stream.connector=smallrye-kafka
        mp.messaging.outgoing.kogito_outgoing_stream.topic=noteapplication
        mp.messaging.outgoing.kogito_outgoing_stream.value.serializer=org.apache.kafka.common.serialization.StringSerializer

        mp.messaging.outgoing.kogito-processinstances-events.connector=smallrye-kafka
        mp.messaging.outgoing.kogito-processinstances-events.topic=kogito-processinstances-events
        mp.messaging.outgoing.kogito-processinstances-events.value.serializer=org.apache.kafka.common.serialization.StringSerializer

        mp.messaging.outgoing.kogito-usertaskinstances-events.connector=smallrye-kafka
        mp.messaging.outgoing.kogito-usertaskinstances-events.topic=kogito-usertaskinstances-events
        mp.messaging.outgoing.kogito-usertaskinstances-events.value.serializer=org.apache.kafka.common.serialization.StringSerializer

        mp.messaging.outgoing.kogito-variables-events.connector=smallrye-kafka
        mp.messaging.outgoing.kogito-variables-events.topic=kogito-variables-events
        mp.messaging.outgoing.kogito-variables-events.value.serializer=org.apache.kafka.common.serialization.StringSerializer
    ```
  * deploy notation  service

    ```sh
    #create the service through kogito operator 
    cd ../notation
    kogito deploy-service notation --infra kogito-infinispan-infra  --infra kogito-kafka-infra --infra kogito-knative-infra
    mvn clean package  -DskipTests=true
    oc start-build notation --from-dir=target 
    ```

     add the following properties to the note-properties configmap

    ```sh
        # edit the configmap
        oc edit cm/notation-properties
        #add the following properties after the `application.properties:` line
       
        mp.messaging.incoming.kogito_incoming_stream.connector=smallrye-kafka
        mp.messaging.incoming.kogito_incoming_stream.topic=noteapplication
        mp.messaging.incoming.kogito_incoming_stream.value.deserializer=org.apache.kafka.common.serialization.StringDeserializer

        mp.messaging.outgoing.kogito_outgoing_stream.connector=smallrye-kafka
        mp.messaging.outgoing.kogito_outgoing_stream.topic=offerresult
        mp.messaging.outgoing.kogito_outgoing_stream.value.serializer=org.apache.kafka.common.serialization.StringSerializer

        mp.messaging.outgoing.kogito-processinstances-events.connector=smallrye-kafka
        mp.messaging.outgoing.kogito-processinstances-events.topic=kogito-processinstances-events
        mp.messaging.outgoing.kogito-processinstances-events.value.serializer=org.apache.kafka.common.serialization.StringSerializer

        mp.messaging.outgoing.kogito-usertaskinstances-events.connector=smallrye-kafka
        mp.messaging.outgoing.kogito-usertaskinstances-events.topic=kogito-usertaskinstances-events
        mp.messaging.outgoing.kogito-usertaskinstances-events.value.serializer=org.apache.kafka.common.serialization.StringSerializer

        mp.messaging.outgoing.kogito-variables-events.connector=smallrye-kafka
        mp.messaging.outgoing.kogito-variables-events.topic=kogito-variables-events
        mp.messaging.outgoing.kogito-variables-events.value.serializer=org.apache.kafka.common.serialization.StringSerializer
    ```

    * get the eligibility route
    ```sh
    projects/kogito-knative-loan main ❯ oc get route
        NAME          HOST/PORT                                                         PATH   SERVICES      PORT   TERMINATION   WILDCARD
        eligibility   eligibility-loan.apps.cluster-152a.152a.sandbox1775.opentlc.com          eligibility   http                 None
        notation      notation-loan.apps.cluster-152a.152a.sandbox1775.opentlc.com 
    ```
    * We are ready for tests, 😆

        * call the eligibility service
            ```sh
                curl -X POST \
                    -H "content-type: application/json"  \
                    -H "ce-specversion: 1.0"  \
                    -H "ce-source: /from/localhost"  \
                    -H "ce-type: eligibilityapplication" \
                    -H "ce-id: 12346"  \
                    -d "{\"age\":3,\"amount\":50000,\"bilan\":{\"gg\":5,\"ga\":2,\"hp\":1,\"hq\":2,\"dl\":50,\"ee\":2,\"siren\":\"423646512\",\"variables\":[]},\"ca\":200000,\"eligible\":false,\"msg\":\"string\",\"nbEmployees\":10,\"notation\":{\"decoupageSectoriel\":0,\"note\":\"string\",\"orientation\":\"string\",\"score\":0,\"typeAiguillage\":\"string\"},\"publicSupport\":true,\"siren\":\"423646512\",\"typeProjet\":\"IRD\"}" \
                    eligibility-loan.apps.cluster-152a.152a.sandbox1775.opentlc.com 
            ```
            The call  invoke the eligibility service that produce another event with type `noteapplication` that invoke the notation service.

  The eligibility process is triggered 
  ```sh
    oc logs eligibility-68949b7b9f-6b9b  
  ```
  ```log
        --/ __ \/ / / / _ | / _ \/ //_/ / / / __/ 
    -/ /_/ / /_/ / __ |/ , _/ ,< / /_/ /\ \   
    --\___\_\____/_/ |_/_/|_/_/|_|\____/___/   
    2021-06-08 16:48:49,911 WARN  [io.sma.rea.mes.provider] (Quarkus Main Thread) SRMSG00207: Some components are not connected to either downstream consumers or upstream producers:
      - SubscriberMethod{method:'org.kie.kogito.addon.cloudevents.quarkus.QuarkusCloudEventPublisher#onEvent', incoming:'kogito_incoming_stream'} has no upstream

    2021-06-08 16:48:49,922 INFO  [org.kie.kog.ser.eve.imp.AbstractMessageConsumer] (Quarkus Main Thread) Consumer for class org.redhat.bbank.model.Loan started.
    2021-06-08 16:48:49,923 INFO  [org.kie.kog.add.clo.qua.QuarkusKogitoExtensionInitializer] (Quarkus Main Thread) Registered Kogito CloudEvent extension
    2021-06-08 16:48:49,943 INFO  [io.quarkus] (Quarkus Main Thread) eligibility 2.0.0-SNAPSHOT on JVM (powered by Quarkus 1.13.3.Final) started in 1.334s. Listening on: http://localhost:8580
    2021-06-08 16:48:49,943 INFO  [io.quarkus] (Quarkus Main Thread) Profile dev activated. Live Coding activated.
    2021-06-08 16:48:49,943 INFO  [io.quarkus] (Quarkus Main Thread) Installed features: [cdi, hibernate-validator, kogito-decisions, kogito-predictions, kogito-processes, kogito-rules, mutiny, qute, rest-client, resteasy, resteasy-jackson, servlet, smallrye-context-propagation, smallrye-health, smallrye-openapi, smallrye-reactive-messaging, swagger-ui, vertx, vertx-web]
    2021-06-08 16:48:49,944 INFO  [io.qua.dep.dev.RuntimeUpdatesProcessor] (vert.x-worker-thread-0) Hot replace total time: 1.399s 
    Incoming request :  Loan [age=3, amount=50000.0, bilan=Bilan [dl=50.0, ee=2.0, fl=0.0, fm=0.0, ga=2.0, gg=5.0, hn=0.0, hp=1.0, hq=2.0, siren=423646512], ca=200000.0, elligible=false, msg=string, nbEmployees=10.0, notation=Notation [decoupageSectoriel=0.0, note=string, orientation=string, score=0.0, typeAiguillage=string], publicSupport=true, siren=423646512, typeProjet=IRD, rate=0.0, nbmonths=0]
    Company number : 423646512exist ? true
  ```

  followed by notation service 
  ```sh
    oc logs notation-75f5fbd586-k4mg7
  ```
  ```log
    __  ____  __  _____   ___  __ ____  ______ 
  --/ __ \/ / / / _ | / _ \/ //_/ / / / __/ 
  -/ /_/ / /_/ / __ |/ , _/ ,< / /_/ /\ \   
  --\___\_\____/_/ |_/_/|_/_/|_|\____/___/   
  2021-06-09 10:23:04,921 WARN  [io.sma.rea.mes.provider] (Quarkus Main Thread) SRMSG00207: Some components are not connected to either downstream consumers or upstream producers:
    - SubscriberMethod{method:'org.kie.kogito.addon.cloudevents.quarkus.QuarkusCloudEventPublisher#onEvent', incoming:'kogito_incoming_stream'} has no upstream

  2021-06-09 10:23:04,929 INFO  [org.kie.kog.ser.eve.imp.AbstractMessageConsumer] (Quarkus Main Thread) Consumer for class org.redhat.bbank.model.Loan started.
  2021-06-09 10:23:04,935 INFO  [org.kie.kog.add.clo.qua.QuarkusKogitoExtensionInitializer] (Quarkus Main Thread) Registered Kogito CloudEvent extension
  2021-06-09 10:23:04,942 INFO  [io.quarkus] (Quarkus Main Thread) notation 2.0.0-SNAPSHOT on JVM (powered by Quarkus 1.13.3.Final) started in 1.462s. Listening on: http://localhost:8680
  2021-06-09 10:23:04,942 INFO  [io.quarkus] (Quarkus Main Thread) Profile dev activated. Live Coding activated.
  2021-06-09 10:23:04,943 INFO  [io.quarkus] (Quarkus Main Thread) Installed features: [cdi, hibernate-validator, kogito-decisions, kogito-predictions, kogito-processes, kogito-rules, mutiny, qute, rest-client, resteasy, resteasy-jackson, servlet, smallrye-context-propagation, smallrye-health, smallrye-openapi, smallrye-reactive-messaging, swagger-ui, vertx, vertx-web]
  2021-06-09 10:23:04,943 INFO  [io.qua.dep.dev.RuntimeUpdatesProcessor] (vert.x-worker-thread-0) Hot replace total time: 1.500s 
  2021-06-09 10:23:05,026 DEBUG [org.kie.kog.add.clo.qua.htt.AbstractQuarkusCloudEventResource] (executor-thread-199) CloudEvent to publish: 
  ☁ ️cloudevents.Event
  Context Attributes,
    specversion: 1.0
    type: noteapplication
    source: /process/eligibility
    id: 427a6b8c-d8c0-4132-af09-5129454e8fdd
    datatype: null
  Extensions,	kogitousertaskist: 1
    kogitoprocinstanceid: 8fbb0f08-43c1-49c6-9057-f7441cf6b805
    kogitoprocid: eligibility
  Data,
    [B@68fb0ea1
  2021-06-09 10:23:05,027 WARN  [org.kie.kog.add.clo.qua.htt.AbstractQuarkusCloudEventResource] (executor-thread-199) Content-Type of the received CloudEvent 'noteapplication' is not supported. Content-type is null. Assuming application/json.
  2021-06-09 10:23:05,029 DEBUG [org.kie.kog.add.clo.qua.htt.AbstractQuarkusCloudEventResource] (executor-thread-199) Decoded event {"specversion":"1.0","id":"427a6b8c-d8c0-4132-af09-5129454e8fdd","source":"/process/eligibility","type":"noteapplication","datacontenttype":"application/json","time":"2021-06-09T10:23:03.363728+02:00","kogitousertaskist":"1","kogitoprocinstanceid":"8fbb0f08-43c1-49c6-9057-f7441cf6b805","kogitoprocid":"eligibility","data":{"siren":"423646512","ca":200000.0,"nbEmployees":10.0,"age":3,"publicSupport":true,"typeProjet":"IRD","amount":50000.0,"notation":{"score":0.0,"note":"string","orientation":"string","decoupageSectoriel":0.0,"typeAiguillage":"string"},"eligible":true,"msg":"Eligible","bilan":{"siren":"423646512","gg":5.0,"ga":2.0,"hp":1.0,"hq":2.0,"hn":0.0,"fl":0.0,"fm":0.0,"dl":50.0,"ee":2.0,"variables":[]},"rate":0.0,"nbmonths":0}}
  2021-06-09 10:23:05,029 DEBUG [org.kie.kog.add.clo.qua.QuarkusCloudEventPublisher] (executor-thread-199) Producing message to internal bus: {"specversion":"1.0","id":"427a6b8c-d8c0-4132-af09-5129454e8fdd","source":"/process/eligibility","type":"noteapplication","datacontenttype":"application/json","time":"2021-06-09T10:23:03.363728+02:00","kogitousertaskist":"1","kogitoprocinstanceid":"8fbb0f08-43c1-49c6-9057-f7441cf6b805","kogitoprocid":"eligibility","data":{"siren":"423646512","ca":200000.0,"nbEmployees":10.0,"age":3,"publicSupport":true,"typeProjet":"IRD","amount":50000.0,"notation":{"score":0.0,"note":"string","orientation":"string","decoupageSectoriel":0.0,"typeAiguillage":"string"},"eligible":true,"msg":"Eligible","bilan":{"siren":"423646512","gg":5.0,"ga":2.0,"hp":1.0,"hq":2.0,"hn":0.0,"fl":0.0,"fm":0.0,"dl":50.0,"ee":2.0,"variables":[]},"rate":0.0,"nbmonths":0}}
  2021-06-09 10:23:05,070 DEBUG [org.kie.kog.ser.eve.imp.AbstractMessageConsumer] (executor-thread-199) Received: org.redhat.bbank.NotationMessageDataEvent_3@5915684f on thread executor-thread-199
  2021-06-09 10:23:05,072 DEBUG [org.kie.kog.eve.imp.CloudEventConsumer] (executor-thread-199) Received message without reference id, starting new process instance with trigger 'noteapplication'
  bilan received Bilan [dl=50.0, ee=2.0, fl=0.0, fm=0.0, ga=2.0, gg=5.0, hn=0.0, hp=1.0, hq=2.0, siren=423646512]
  Variable [type=rentab_13, value=10.0]
  Variable [type=strfin_36, value=25.0]
  notation Notation [decoupageSectoriel=1.0, note=A, orientation=Approved, score=0.0, typeAiguillage=MODELE_1]
  Loan before offer Loan [age=3, amount=50000.0, bilan=Bilan [dl=50.0, ee=2.0, fl=0.0, fm=0.0, ga=2.0, gg=5.0, hn=0.0, hp=1.0, hq=2.0, siren=423646512], ca=200000.0, elligible=true, msg=Eligible, nbEmployees=10.0, notation=Notation [decoupageSectoriel=1.0, note=A, orientation=Approved, score=0.0, typeAiguillage=MODELE_1], publicSupport=true, siren=423646512, typeProjet=IRD, rate=0.0, nbmonths=0]
  notation Notation [decoupageSectoriel=1.0, note=A, orientation=Approved, score=0.0, typeAiguillage=MODELE_1]
  Offer calculated : 2.0 36
  ```

the calculated score is `Notation [decoupageSectoriel=1.0, note=A, orientation=Approved, score=0.0, typeAiguillage=MODELE_1]`
the proposed offer is `Rate : 2.0% - 36 months` 


