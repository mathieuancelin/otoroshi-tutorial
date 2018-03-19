# Try Otoroshi

in this tutorial, we will see how to get and install Otoroshi, and how to proxy an http api with load balancing

## Step 0 - install and run demo env.

Otoroshi demo environment is provided as a docker image, you just have to run the following command

```sh
docker build --no-cache -t otoroshi-demo .
docker run -p "8080:8080" -it otoroshi-demo bash
```

## Step 1 - Run Otoroshi

```sh
java -jar otoroshi.jar &
```

## Step 2 - Check if admin api works

```sh
./otoroshicli services all
./otoroshicli apikeys all
./otoroshicli groups all
```

## Step 3 - Create a service 

the service will proxy call to https://freegeoip.net through http://ip.geo.com:8080

```sh
./otoroshicli services create --group default --name geo-ip-api --env prod \
  --domain geo.com --subdomain ip --root /json/ --target https://freegeoip.net \
  --public-pattern '/.*' --no-force-https
```

Then test it

```sh
./otoroshicli tryout call "http://127.0.0.1:8080/" -X GET -H 'Host: ip.geo.com'
```

## Step 4 -  Create a new service with multiple targets

Run 3 web server on different ports. Each server will expose an API responding with different names.

```sh
./otoroshicli tryout serve 9901 & 
./otoroshicli tryout serve 9902 &
./otoroshicli tryout serve 9903 &
```

Create a service that will loadbalance between these 3 services and serves them through http://api.hello.com:8080

```sh
./otoroshicli services create --group default --id hello-api --name hello-api \
  --env prod --domain hello.com --subdomain api --root / \
  --target "http://127.0.0.1:9901" \
  --target "http://127.0.0.1:9902" \
  --public-pattern '/.*' --no-force-https --client-retries 3
```

## Step 5 - Try loadbalancing

Then test it multiple time to observe loadbalancing

```sh
./otoroshicli tryout call "http://127.0.0.1:8080/" -H 'Host: api.hello.com' -H 'Accept: application/json'
```

Then add a new target

```sh
./otoroshicli services add-target hello-api --target="http://127.0.0.1:9903"
```

Then test it multiple time to observe loadbalancing

```sh
./otoroshicli tryout call "http://127.0.0.1:8080/" -H 'Host: api.hello.com' -H 'Accept: application/json'
```

## Step 6 - kill some processes

Kill the first server 

```sh
ps aux  |  grep -i 9901 |  grep -v grep   | awk '{print $2}' | xargs kill
```

test it multiple time to observe loadbalancing

```sh
./otoroshicli tryout call "http://127.0.0.1:8080/" -H 'Host: api.hello.com' -H 'Accept: application/json'
```

Then kill the second server 

```sh
ps aux  |  grep -i 9902 |  grep -v grep   | awk '{print $2}' | xargs kill
```

and test it multiple time to observe loadbalancing

```sh
./otoroshicli tryout call "http://127.0.0.1:8080/" -H 'Host: api.hello.com' -H 'Accept: application/json'
```

Then kill the last server 

```sh
ps aux  |  grep -i 9903 |  grep -v grep   | awk '{print $2}' | xargs kill
```

and test it to observe connection error

```sh
./otoroshicli tryout call "http://127.0.0.1:8080/" -H 'Host: api.hello.com' -H 'Accept: application/json'
```

## Step 7 - stop everything

now you can delete your service

```sh
./otoroshicli services delete hello-api
```

stop otoroshi

```sh
ps aux  |  grep -i java |  grep -v grep   | awk '{print $2}' | xargs kill
```

and stop the container

```sh
exit
```
