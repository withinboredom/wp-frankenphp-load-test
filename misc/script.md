: prep :

```shell
export DK_VERSION=19
docker build --pull -t registry.bottled.codes/withinboredom/frankenphp-wordpress:latest-worker-$DK_VERSION .
docker push registry.bottled.codes/withinboredom/frankenphp-wordpress:latest-worker-$DK_VERSION
sed -i "s/latest-worker-.*$/latest-worker-$DK_VERSION/" wordpress-frankenphp.yaml
kubectl apply -f wordpress-frankenphp.yaml
```



```shell
# let's first create a namespace on the cluster

loft use cluster
loft create space example

# and deploy a database and apache deployment using official images
kubectl apply -f mysql.yaml
kubectl apply -f wordpress-apache.yaml

# and make sure they are running
kubectl get pods -o wide
kubectl get ing -o wide

# I'm not waiting around for dns propagation, so I'm going to cheat
# and add an entry to my hosts file
echo "$(kubectl get ing -o jsonpath='{.items[0].status.loadBalancer.ingress[0].ip}') wordpress-apache.bottled.codes" | sudo tee -a /etc/hosts
# and now I can ping and curl the site
ping wordpress-apache.bottled.codes
curl wordpress-apache.bottled.codes

# now lets configure it
lynx https://wordpress-apache.bottled.codes

# and deploy fpm
kubectl apply -f wordpress-fpm.yaml

# and add it to my hosts file
echo "$(kubectl get ing -o jsonpath='{.items[1].status.loadBalancer.ingress[0].ip}') wordpress-fpm.bottled.codes" | sudo tee -a /etc/hosts

# and now lets deploy frankenphp
kubectl apply -f wordpress-frankenphp.yaml
kubectl get pods -o wide

# frankenphp is directly connected to a loadbalancer
# so we need to check on the service
kubectl get svc -o wide

# and again, we're not waiting for dns propagation
echo "$(kubectl get svc -o jsonpath='{.items[2].status.loadBalancer.ingress[0].ip}') wordpress-franken.bottled.codes" | sudo tee -a /etc/hosts

# ok, lets load test!!!!

clear


# scaling
kubectl scale deployment wordpress-apache --replicas=
kubectl scale deployment wordpress-fpm --replicas=
kubectl scale deployment wordpress-frankenphp --replicas=

# set all to 0 except what we are testing

# I have 16 cores on this computer, so I'm going to use 16 threads, 1000 connections, over 30 seconds
wrk -t 12 -c 1000 -d 30s --timeout 60s https://wordpress-franken.bottled.codes
 
# and now apache
wrk -t 12 -c 1000 -d 30s --timeout 60s https://wordpress-apache.bottled.codes

# and fpm
wrk -t 12 -c 1000 -d 30s --timeout 60s https://wordpress-fpm.bottled.codes


ali https://wordpress-apache.bottled.codes -w 16 -d 1m -r 200 --query-range 10m -t 1m
ali https://wordpress-fpm.bottled.codes -w 16 -d 1m -r 200 --query-range 10m -t 1m
ali https://wordpress-frankenphp.bottled.codes -w 16 -d 1m -r 200 --query-range 10m -t 1m
# and retest:

```
