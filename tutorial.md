# Try Otoroshi

in this tutorial, we will see how to get and install Otoroshi, and how to proxy an http api with load balancing

## Step 1 - Download Otoroshi and its CLI

```sh
wget --quiet https://dl.bintray.com/mathieuancelin/otoroshi/latest/otoroshi.jar
wget --quiet https://github.com/MAIF/otoroshi/edit/master/clients/cli/otoroshicli.toml
wget --quiet https://dl.bintray.com/mathieuancelin/otoroshi/linux-otoroshicli/latest/otoroshicli
chmod +x ./otoroshicli
```

## Step 2 - Run Otoroshi

```sh
java -jar otoroshi.jar &
```

## Step 3 - Check if admin api works

```sh
./otoroshicli services all
./otoroshicli apikeys all
./otoroshicli groups all
```

## Step 4 - Create a service 

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

## Step 5 - Try loadbalancing

Run 3 new microservices in 3 new terminal processes

```sh
./otoroshicli tryout serve 9901
./otoroshicli tryout serve 9902
./otoroshicli tryout serve 9903
```

Create a service that will loadbalance between these 3 microservices and serves them through http://api.hello.com:8080

```sh
./otoroshicli services create --group default --id hello-api --name hello-api \
  --env prod --domain hello.com --subdomain api --root / \
  --target "http://127.0.0.1:9901" \
  --target "http://127.0.0.1:9902" \
  --public-pattern '/.*' --no-force-https --client-retries 3
```

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

Then kill one of the microservices and test it multiple time to observe loadbalancing

```sh
./otoroshicli tryout call "http://127.0.0.1:8080/" -H 'Host: api.hello.com' -H 'Accept: application/json'
```

Then kill a second microservices and test it multiple time to observe loadbalancing

```sh
./otoroshicli tryout call "http://127.0.0.1:8080/" -H 'Host: api.hello.com' -H 'Accept: application/json'
```

Then kill the last microservices and test it to observe connection error

```sh
./otoroshicli tryout call "http://127.0.0.1:8080/" -H 'Host: api.hello.com' -H 'Accept: application/json'
```

then delete your service

```sh
./otoroshicli services delete hello-api
```