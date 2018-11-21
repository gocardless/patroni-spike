# Patroni Spike

## What is this

A quick spike on running Patroni+Postgres in kubernetes, but only as a quick-and-dirty
substitute for spinning up VMs. As a result, we've avoided all of Patroni's "kube-native"
features.

## Deployment

`ls *.yaml | xargs -n1 kubectl apply -f`


## Manual failover

`patronictl` may fail you. So you can port-forward to any of the patroni
processes and run the following:

```
curl --silent -XPOST localhost:8008/failover -H "Content-Type: application/json" -d '{"leader": "postgres_1", "candidate": "postgres_2"}'
```
