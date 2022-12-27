# One Docker image to rule them all

I just found out [nixery](https://nixery.dev) !

> *Nixery is a Docker-compatible container registry that is capable of transparently building and serving container images using Nix.*
> 
> *Images are* ***built on-demand*** *based on the image name. Every package that the user intends to include in the image is specified as a path component of the image name.*
> 
> *The path components refer to top-level keys in nixpkgs and are used to build a container image using a layering strategy that optimises for caching popular and/or large dependencies.*

In other words, you start with the base image, `nixery.dev/` and then list the packages and tools you want available. Usually, you start with the `shell` meta package, followed by any [NixOS package(s)](https://search.nixos.org/packages?channel=22.05&show=pingtcp&from=0&size=50&sort=relevance&type=packages).

This is very handy when working with Kubernetes.

## Examples

**Note**: the command format to run an ephemeral pod on Kubernetes is:

```bash
kubectl run -it --rm --restart=Never \
 --image=nixery.dv/<PACKAGES> \
 <NAME> -- <CMD>
```

Connect to a database using `psql`, assuming the service is called `my-db`:

```bash
kubectl run -it --rm --restart=Never \
  --image=nixery.dev/postgresql \
  --env PGPASSWORD=some-password \
   psql -- psql -h my-db -U some-username
```

Test the connectivity to a pod:

```bash
kubectl run -it --rm --restart=Never \
  --image=nixery.dev/shell/unixtools.ping \
  ping -- ping keycloak.cluster.local
```

Get a shell with `curl`, `grep` and `nc` commands: `bash kubectl run -it --rm --restart=Never \ --image=nixery.dev/shell/curl/gnugrep/ping/netcat \ shell -- bash`

## Limitations

For those not familiar with NixOs, it may be troublesome to find the package name that will bring you the executable you need. Here are some:

*   `psql` → package `postgresql`
    
*   `ping` → package `unixtools.ping`
    
*   `grep` → package `gnugrep`
    
*   `nc` → package `netcat`
    

Also, I wasn't able to run with `root` permissions, meaning I could not run `iptables -L` (with the package `iptables`). Maybe I missed something? Let me know in the comments!