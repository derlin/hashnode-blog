# kubectl run: spawn temporary docker containers on Kubernetes

When working with Kubernetes, there are times when you wished you had a specific tool to help debug a problem, visualize some data, or take some actions. Well, there is actually an easy way: deploy a docker container with the necessary tool directly to your cluster with one command, use it, and let it be destroyed as soon as you do not use it anymore!

How to spawn ephemeral containers on k8s.

## `kubectl` run command

The `run` command creates and runs a particular image in a pod.

> `run` *will start running 1 or more instances of a container image on your cluster. Usage:*
> 
> ```bash
> kubectl run NAME --image=image [--env="key=value"] [--port=port]
> [--dry-run=server|client] [--overrides=inline-json]
> [--command] -- [COMMAND] [args...]
> ```

The image can be anything, as long as it can be pulled from the cluster. With the two options `--attach`/`-it` (wait for the Pod to start running, and then attach to the Pod / open a shell) and `--rm` (delete the pod after it exits), it is the perfect way to get the right tools into the cluster for a short while.

## Example: interact with a database using psql

Let's say your project runs in the namespace `myproject` and the micro-services use PostgreSQL on RDS (managed relational database on AWS). RDS doesn't come with a nice postgres admin tool, and your infra colleagues only gave you [adminer](https://www.adminer.org/) to interact with it. You want `psql`.

From your terminal, ensure you are connected to the right kubernetes (`kubectl config current-context`) context and run:

```bash
# run bash in a container with psql installed 
# on your namespace
kubectl --namespace myproject \
  run -it --rm psql --image=postgres:13 -- bash;
```

Running this command, you suddenly have a shell in your cluster with `psql` installed. You can now run the following to have the `psql` prompt:

```bash
PGPASSWORD='my-user-pwd' psql postgres -U my-user \
  -h hostname-of-the-db-cluster.rds.amazonaws.com
```

In case you have network policies in place, you can easily add the needed labels to your `psql` pod using the `-l` option:

```bash
kubectl run ... -l "db-access: true" -l "role: alice" ...
```

Once you exit the shell, the pod should disappear.

## Example: Kafka UI

Let's say you have a Kafka cluster running, and you need to debug the messages going through. There is no Kafka visual interface on your cluster.

Why not just set a port forwarding (`kubectl port-forward my-kafka 9092`) and run some tool locally you ask? Well, the tool would be able to connect, but the advertised hostname will point to a host only available from within kubernetes. Getting messages will thus fail (unless manually editing `/etc/hosts`, which is not possible if the tool runs on a Docker container).

Kafka visualization tools are a plethora: [Kafka UI](https://github.com/provectus/kafka-ui), [Kafka Magic](https://www.kafkamagic.com/), [kafdrop](https://github.com/obsidiandynamics/kafdrop) to cite a few.

Let's take kafka-ui as an example:

```bash
kubectl --n myproject run -it --rm kui \
  --port 8080 \
  --image=provectuslabs/kafka-ui:latest \
  --env "KAFKA_CLUSTERS_0_NAME=main" \
  --env "KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS=kafka:9092" \
```

The `KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS` must match the name of the pod/service of the kafka broker in the cluster. If you use [strimzi](https://strimzi.io/), it would look something like `<cluster-name>-kafka-bootstrap:9092`.

Now, you just need to port forward the `kui` port to access it from your local machine:

```bash
# on another terminal !
kubectl -n myproject port-forward kui 8080
```

You have kafka-ui available on [http://localhost:8080](http://localhost:8080). Once you finished, close the terminal running `kubectl run` and the `kui` pod will be deleted.

The same logic applies to other tools: run the docker image in the cluster, then port forward.

## Wrap-up

`kubectl run` is one heck of a magic tool to add to your debugging toolbox. As the kafka example shows, it may also be a way to avoid crowding your kubernetes cluster with management tools that are only used once in a while. Don't maintain, just run when needed!

* * *

Written with ❤ by [derlin](https://derlin.ch)