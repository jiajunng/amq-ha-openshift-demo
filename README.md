# amq-ha-openshift-demo
With AMQ Broker on OpenShift, highly available allows the maintaining the availability of brokers and the integrity of the messaging data if a broker fails. This demo includes the deployment the of multiple instance AMQ Broker through an operator on OpenShift with each individual broker pod writing its message data to a persistent storage; to one broker pod goes offline and message migration happens.

## Prerequisites/Requirements
- Up and running OpenShift (Tested on v4.3)
- AMQ Broker v7.5
- At least one broker Pod running in your deployment for message migration to occur.

## Basic Instructions
- Create AMQ operator via Operators -> OperatorHub

- Create a ActiveMQArtemises broker instance:

```
apiVersion: broker.amq.io/v2alpha1
kind: ActiveMQArtemis
metadata:
  name: ex-aao
  namespace: amq-demo
spec:
  acceptors:
    - name: amqp
      needClientAuth: false
      port: 5671
      protocols: amqp
      sslEnabled: true
      sslSecret: ex-aao-amqp-secret
      verifyHost: false
  deploymentPlan:
    image: 'registry.redhat.io/amq7/amq-broker:7.5'
    messageMigration: true
    persistenceEnabled: true
    size: 2
```

In the ActiveMQArtemises broker custom resource (CR), messageMigration and persistenceEnabled need to be set to true. For more information on the configuration, visit [here](https://github.com/rh-messaging/activemq-artemis-operator/blob/0.9.1/deploy/crds/broker_v2alpha1_activemqartemis_crd.yaml#L80-L82)

- There should be two broker pods:

```
# oc get pod -o wide
NAME                                   READY   STATUS    RESTARTS   AGE    IP           NODE                                              NOMINATED NODE   READINESS GATES
amq-broker-operator-78d5c5b79c-kdshq   1/1     Running   1          136m   10.10.2.9   ip-10-5-100-168.ap-southeast-1.compute.internal   <none>           <none>
ex-aao-ss-0                            1/1     Running   0          65s    10.10.0.1   ip-10-5-110-221.ap-southeast-1.compute.internal   <none>           <none>
ex-aao-ss-1                            1/1     Running   0          2m8s   10.10.0.2   ip-10-5-123-223.ap-southeast-1.compute.internal   <none>           <none>
```

- Go into each Pod and send some messages to each broker:

```
# oc rsh ex-aao-ss-0 
/opt/amq/bin/artemis producer --url tcp://10.10.0.1:61616 --user admin --password admin

# oc rsh ex-aao-ss-1 
/opt/amq/bin/artemis producer --url tcp://10.10.0.2:61616 --user admin --password admin
```

The preceding commands create a queue called TEST on each broker and add 1000 messages to each queue.

- On the AMQ Broker console, you should be able to see 1000 messages in the queue, TEST:

- Scale down the cluster from 2 to 1 by editing the ActiveMQArtemises broker custom resource (CR)

- You see that the Pod ex-aao-ss-1 starts to shut down, then go to the AMQ Broker console, you will now see 2000 messages in the queue, TEST:

![Image of message migration](<img width="1440" alt="Screenshot 2020-03-02 at 7 04 27 PM" src="https://user-images.githubusercontent.com/25560159/75670949-adedc480-5cb8-11ea-9fa0-8e9a78513e55.png">)
