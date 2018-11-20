# Patroni Spike

## How to failover

`patronictl` may fail you. So you can port-forward to any of the patroni
processes and run the following:

```
curl --silent -XPOST localhost:8008/failover -H "Content-Type: application/json" -d '{"leader": "postgres_1", "candidate": "postgres_2"}'
```
