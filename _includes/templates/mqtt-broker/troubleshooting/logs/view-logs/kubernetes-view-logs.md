View all pods of the cluster:

```bash
kubectl get pods
```

View last logs for the desired pod:
 
```bash
kubectl logs -f POD_NAME
```

To view ThingsBoard MQTT Broker logs use command:

```bash
kubectl logs -f tb-broker-0
```

You can use <b>grep</b> command to show only the output with desired string in it. 
For example, you can use the following command in order to check if there are any errors on the backend side:

```bash
kubectl logs -f tb-broker-0 | grep ERROR
```

If you have multiple nodes you could redirect logs from all nodes to files on you machine and then analyze them: 

```bash
kubectl logs -f tb-broker-0 > tb-broker-0.log
kubectl logs -f tb-broker-1 > tb-broker-1.log
```

**Note:** you can always log into the ThingsBoard MQTT Broker container and view logs there:

```bash
kubectl exec -it tb-broker-0 -- bash
cat /var/log/thingsboard-mqtt-broker/thingsboard-mqtt-broker.log
```
