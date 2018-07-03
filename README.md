# BCF Deployer

Blue CLoud Foundry(BCF) Deployer is an end-user client to be used to deploy CF on IBM Armada. It includes a set of **helm** chart configuration files and a `make` command set to help you deploy CF on Armada easily and quickly.

## Requirements
1. Clone this repository: `git clone git@github.com:bluebosh/bcf-deployer.git`
1. This BCF-Deployer is supported for Linux and Mac now, but **Linux** is prefer
1. The Kubernetes node size must be larger than **3**. The minimal BCF size is 2 Control Plane node, 1 Cell node
1. You need provide a free pair of ip and domain name resolution to access the CF. If you don't have a free domain, you can use the Armada cluster default domain as `domain=default`
1. You need to **allow** Kubernetes Helm to access internet during execution

## All-in-one deploy process
```bash
Usage: ./bcf-deployer.sh [options]
Options:
   -d|--domain <Ingress subdomain|Public domain>
                   Specify custom domain for BCF environment (set default to use Armada cluster default domain)

   -s|--size [Diego-cell size]
                   Specify Diego-cell size (default: 1)

   -e|--execute [install|upgrade|scale|clean]
                   Specify execute type

   -v|--imageversion [default|old|xxx]
                   Optional: Specify BCF image version (default: default is the version in Helm Charts)

   -bt|--blobstoretype [webdav|fog]
                   Optional: Specify blobstore type (default: webdav)

   -bn|--blobstorename [new|xxx]
                   Optional: Specify BCF blobstore name (default: 'new' = new name)

   -bu|--blobstoreurl <value>
                   Required: Specify BCF blobstore url

   -bi|--blobstoreid <value>
                   Optional: Specify BCF blobstore id

   -bk|--blobstorekey <value>
                   Optional: Specify BCF blobstore key

   -ph|--postgreshost [local|xxx]
                   Optional: Specify Postgres hostname if using external Postgres (default: 'local' = local Postgres)

   -pp|--postgresport
                   Optional: port of external Postgres, only required if using external Postgres

   -pu|--postgresuser
                   Optional: user of external Postgres, only required if using external Postgres

   -ppw|--postgrespwd
                   Optional: password for user of external Postgres, only required if using external Postgres

   -pdd|--postgresdd
                   Optional: name of external Postgres default database (default: 'compose')

   -dh|--dockerhostname [default|xxx]
                   The hostname of Docker Registry (default: the hostname which is defined in Armada)

   -du|--dockerusername [default|xxx]
                   The username of Docker Registry (default: the username which is defined in Armada)

   -dp|--dockerpassword [default|xxx]
                   The password of Docker Registry (default: the password which is defined in Armada)

   -lt|--loginservertype [uaa|local|xxx]
                   Optional: Specify LoginServer type (default: 'uaa' = UAA only)

   -li|--loginserveriamurl [ys1|yp|xxx]
                   Optional: Specify LoginServer iam url (default: 'yp' = yp iam url)

   -lc|--loginserverclientid <value>
                   Required: Specify LoginServer BlueID client id

   -ls|--loginserverclientsecret <value>
                   Required: Specify LoginServer BlueID client secret

   -lo|--loginserverssosecret <value>
                   Required: Specify LoginServer SSO secret

   -st|--storagetype [armada|local]
                   Optional: Specify Storage class type (default: armada)

   -is|--ingresssecret [true|false]
                   Optional: Specify if using armada ingress secret (default: false)

   -isn|--ingresssecretname
                   Optional: when armada ingress secret is true, this value indicate the ingress secret name. default as prefix of domain

   -cr|--certrecreate [true|false]
                   Optional: Specify if recreate internal certs (default: false)

   -dp|--diegocellpartition [0|xxx]
                   Specify how to upgrade diego-cell, this is required for BCF upgrade

   --generate
                   Render BCF helm charts

   --verify
                   Verify BCF functionality
```

**NOTE**
1. If you choose `domain=default`, your CF domain will be the ibm-provided cluster domain which have been been binded with Ingress Controller IP.
1. If you want to use custom domain, you have to use a public DNS like DYN/AWS to resolve _your domain name_ to _your cluster Ingress Controller External IP_.  For example,  in following case, **169.47.96.70** is the IP you need to bind with your custom domain **bcf-dev.cc**: <br />
```
$ k get svc -n kube-system
NAME                                             CLUSTER-IP       EXTERNAL-IP    PORT(S)                      AGE
public-cr9a7293d3e6de4f1a832a058b5e02f674-alb1   172.21.188.143   169.47.96.70   80:30754/TCP,443:31060/TCP   2d
...
```
1. If you don't use parameter `execute=true`, client just generates helm configuration files **but no deploy**
1. Now you have to modify Chart Version and HA Mode in values.yml manually, this will be improved in next version

## Support of Both local and External Postgres
Bcf-deployer support bosh local and external Postgres.
1. If there's **no --postgreshost** specified or **--postgreshost local**, bcf-deployer will use local Postgres, in this case a Postgres container will be deployed in **cf** and **uaa** namespace.
1. If you want to use external Postgres, please specify **--postgreshost {your-postgres-host}**, **--postgresport {your-postgres-port}**, **--postgresuser {your-postgres-user}** and **--postgrespwd {your-postgres-user-password}**in options.
   (Currently support Compose of PostgreSQL 9.6.3)
1. bcf-deployer will check the availability of the Postgres, config the Postgres and then run the deployment.
1. bcf-deployer will create a K8s service of ExternalName type as an Alias of the External Postgres Host, you can find it in uaa namespace by
```
$ kubectl get svc external-db-service -n uaa
```
1. bcf-deployer will create a K8s secret to store the password for CCDB, UAADB and BBSDB, you can find it in uaa namespace by
```
$ kubectl get secret external-db-password-secret -n uaa
```

## CF Login
After about 20 mins, the deployment is ready, start order is UAA -> CF, then you can see all pods are running.
If there is any pod is not running correctly, please check:
1. Pod Description: `kubectl describe pod <pod name> -n <namespace name>`
1. Pod log: `kubectl log <pod name> -n <namespace name>`
1. Login the pod to check `monit summary` directly
1. Please check if Persistent Disk is created and mounted correctly and your hostname can be resolved and accessed correctly. Or you will find the related error message in logs.

Set CF API:
```bash
cf api --skip-ssl-validation https://api.<your domain name>
```
Then login CF:
```bash
cf login -u admin -p changeme
```

## Clean
Clean all deployments of cf and uaa namespaces on Armada
```bash
./bcf-deployer.sh -e clean
```

## Upgrade
Upgrade new helm release
```bash
./bcf-deployer.sh -d default -e upgrade
```

## Scale
Scale up diego cell size from 1 to 2
```bash
./bcf-deployer.sh -e scale -s 2
```

Scale down diego cell size from 2 to 1
```bash
./bcf-deployer.sh -e scale -s 1
```

## Other functions
Get CF and image version
```bash
make version
```
Collect all logs, monit and jobs of pods
```bash
scripts/collect_logs.sh
```


***Note:***
1. You can use `kubectl get pods --all-namespaces=true` to monitor all pod status
1. You can use `scripts/clean_helm_release.sh true` to remove all helm releases
1. It is better to use **root** user, or you may get some Permission deny when access /tmp folder
