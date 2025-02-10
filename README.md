# V1 Coding Challenge Rules

## Overview

This coding challenge consists of creating an HTTP application server (bindend on port 3000) that exposes a GET route, which takes an integer as input and computes its Collatz conjecture, returning the result (computed as the number of steps to reach 1). The application must be deployed on a virtual machine, with the goal of maximizing the number of requests per second that the server can handle, minimizing the errors (non 2xx responses or wrong results).

**The choice of programming language or framework is entirely up to you.**

Once the server is deployed on the VM, you must register (using the HTTP requests provided below) with the machine that will perform the load tests.

When you request a test, your request will be placed in a queue and executed when the load testing machine is free, ensuring that only one test is run at a time and that the same bandwidth is allocated to each participant.

Once you have performed one or more tests, you will be able to see your detailed results and view the current leaderboard.

The score calculation will take into account the number of correct requests made within the time unit and the number of incorrect requests.

The score is calculated as follows (the higher the better):

```js
score = mean_number_of_request_per_second / (1 + non_2xx_responses + wrong_responses)
```

**You must also provide an endpoint that returns the number of request you received for every run, as exaplained below. If this number differ more than 1% of the total request made in the run, the score will be zero**


![Infra](assets/infra.png?raw=true "Infra")

### Locally develop your solution

In order to test the correctness of your application server, do this test (on your local machine):

curl -X GET http://0.0.0.0:3000/collatz/:run_id/:request_id/:number_to_compute

```sh
curl -X GET http://0.0.0.0:3000/collatz/938151bf-4c3b-4099-b6b7-582e2e972641/d2442ec6-0ae6-4b20-a9a7-4265ca8ec180/17
```

Must return (for instance):

Status code: 

```sh
HTTP/1.1 200 OK
```

Headers:

```sh
Date: Fri, 07 Feb 2025 07:47:55 GMT
Content-Length: 82
```

Body (request_id | steps):

```sh
d2442ec6-0ae6-4b20-a9a7-4265ca8ec180|13
```

**You need also a route to return the number of request received for a *run_id***:

curl -X GET http://0.0.0.0:3000/reqs/938151bf-4c3b-4099-b6b7-582e2e972641

```sh
curl -X GET http://0.0.0.0:3000/reqs/938151bf-4c3b-4099-b6b7-582e2e972641

# returns a plain number
```

### Load test info

The loader machine will target your VM with millions of request, using multiple connections, for a variable time interval.

## Setup the test enviroment

### 1. Create the VM on DO

The first step is to upload your public ssh key on DO (for who has already an ssh key on DO, this step is not mandatory):

```sh
export DIGITALOCEAN_TOKEN=XXX # I will provide the token
export SSH_PUB_KEY_NAME=XXX-coding-challenge # Replace with a meaningful name
export SSH_PUB_KEY=ssh-rsa XXXX # Replace with your id_rsa.pub, for instance: ssh-rsa ds+Zt6kvJ95dkWE2YteQhG5OkNVFgmGKeQNgZ0kURM9feeLyYtwNNG1Lynttb9pqo1u93CH+cRuxNo/cAHsgcMd6KAQ9d1k4+dssd+L/JK3e8xqK+oM3wnT+dsdsd+dsdsds/Kx1UK/09FJfhurBTrWQvcrvtF6WriVdy90rHcrVPw08wTxDhK/+qkSNW4W/dsds+una8pV8gMorNdDEnAXix5B6AbKZnVpZOGTiiglfQYRO8GCVO6LV3EnPXkrxWbFp1nwuCxw6mIB1nbGmp6j4pVi9llKzkOJ7ZyBzrujfP5Ewr6EjQWUoovRfEPMD2VLrl8hpqX4+A8= as@alice-MacBook-Air.local


curl -X POST -H "Content-Type: application/json" -H "Authorization: Bearer $DIGITALOCEAN_TOKEN" -d '{"name":"'$SSH_PUB_KEY_NAME'","public_key":"'$SSH_PUB_KEY'"}' "https://api.digitalocean.com/v2/account/keys"
```

Get your ssh public key fingerprint:

```sh
ssh-keygen -l -E md5 -f $HOME/.ssh/id_rsa.pub
```

Create the specified VM on the coding challenge vpc:

```sh
export VM_NAME=XXX # Choose your machine name, please insert your name/surname in some way
export SSH_FINGERPRINT=XXX # For instance: 7a:55:23:49:c4:11:77:a6:34:26:23:94:c1:75:67:11 - Remove the first "MD5:" part if ssh-keygen print it

curl -X POST -H "Content-Type: application/json" -H "Authorization: Bearer $DIGITALOCEAN_TOKEN" -d '{"name":"'$VM_NAME'","region":"fra1","size":"c-4-intel","image":"ubuntu-22-04-x64","ssh_keys":["'$SSH_FINGERPRINT'"],"backups":false,"ipv6":false,"monitoring":false,"tags":["coding-challenge"],"vpc_uuid":"1f542d99-824b-447e-ad22-c5afe2448833"}' "https://api.digitalocean.com/v2/droplets" 
```

Once you have created the machine, you can use this curl request in order to get the machine IP addresses:

```sh
export VM_ID=XXX # Retrive this from the previous json response at *droplet.id*
curl -X GET -H "Content-Type: application/json" -H "Authorization: Bearer $DIGITALOCEAN_TOKEN" "https://api.digitalocean.com/v2/droplets/$VM_ID"   
```

*This machine has 4 vCPU and 8GB of RAM, and it cost 108$/month.*

You will be able to connect to the machine via ssh. All other ports (from the external internet) will be blocked by the firewall.


### 2. Register yourself to the challenge server

Once inside the VM, first find your IP address (note that these machines can have multiple network interfaces and you will find both the internal VPC address and the public one, use the internal).

There is a global registration token that you have to use only to register your machine to the loader machine.

*token*

```sh
export SP_EMAIL="XXX@smartpricing.it" # This is your email
export SP_REGISTRATION_TOKEN="XXX" # I Will provide it
export SP_TEST_MACHINE_IP=Z.Z.Z.Z # This is your VM IP
```

Register to the loader machine:

```sh
curl -X POST -H 'Content-Type: application/json' -d '{"user":"'$SP_EMAIL'", "token": "'$SP_REGISTRATION_TOKEN'", "ip":"'$SP_TEST_MACHINE_IP'"}'  http://10.114.16.2:2999/cc/v1/register
```

and save the returned personal token that you will have to use in all other requests.

### 3. Setup your application server and test it

Once your application server is up and running on the VM, you can try the connection between the loader machine and your VM:

```sh
export SP_TOKEN="XXX" # Printend by the register route above

curl -X POST -H "Authorization: Bearer $SP_TOKEN" http://10.114.16.2:2999/cc/v1/test
```

In case of succesful request the response will be:

```json
{"error":null,"connected":true,"response":"1|0"}
```

## Require a loading test

```sh
curl -X POST -H "Authorization: Bearer $SP_TOKEN" 10.114.16.2:2999/cc/v1/run
```

In case of succesful request the response will be:

```json
{"error":null}
```

In case of succesful request the response will be:

## Leaderboard

To retrive the global leaderboard:

```sh
curl -X GET -H "Authorization: Bearer $SP_TOKEN" 10.114.16.2:2999/cc/v1/leaderboard

alice.setti@smartpricing.it	1000	2025-02-06T09:07:54.048Z	1367min ago
aliceviola@smartpricing.it	900		2025-02-06T09:08:43.325Z	1367min ago
```

To retrive only your test statistic (ordered by score):

```sh
curl -X GET -H "Authorization: Bearer $SP_TOKEN" 10.114.16.2:2999/cc/v1/my
```

Response:

```json
{"user":"alice.setti@smartpricing.it","date":"2025-02-07T20:24:54.246Z","result":{"url":"http://0.0.0.0:3000","connections":1,"sampleInt":1,"pipelining":1,"workers":1,"duration":4.45,"samples":24,"start":"2025-02-07T20:24:49.400Z","finish":"2025-02-07T20:24:53.850Z","errors":0,"timeouts":0,"mismatches":1,"non2xx":0,"resets":0,"1xx":0,"2xx":1,"3xx":0,"4xx":0,"5xx":0,"statusCodeStats":{"200":{"count":1}},"latency":{"average":1,"mean":1,"stddev":1,"min":1,"max":1,"p0_001":0,"p0_01":0,"p0_1":0,"p1":0,"p2_5":0,"p10":0,"p25":0,"p50":1,"p75":1,"p90":3,"p97_5":1,"p99":1,"p99_9":1,"p99_99":1,"p99_999":1,"totalCount":1},"requests":{"average":1.34,"mean":1.1,"stddev":1.1,"min":1,"max":1,"total":1,"p0_001":1,"p0_01":1,"p0_1":1,"p1":1,"p2_5":1,"p10":1,"p25":1,"p50":1,"p75":1,"p90":1,"p97_5":1,"p99":1,"p99_9":1,"p99_99":1,"p99_999":1,"sent":1},"throughput":{"average":1,"mean":1,"stddev":1.04,"min":1,"max":1,"total":1,"p0_001":1,"p0_01":1,"p0_1":1,"p1":1,"p2_5":1,"p10":1,"p25":1,"p50":1,"p75":1,"p90":1,"p97_5":1,"p99":1,"p99_9":1,"p99_99":1,"p99_999":1}},"score":0.1}
```

## Rules and hints

All the loader machine endpoints are rate limited by IP:


```sh
/cc/v1/register [1 request/minute]
/cc/v1/test [10 request/minute]
/cc/v1/run [1 request/minute]
/cc/v1/leaderboard [1 request/minute]
/cc/v1/my [1 request/minute]
```

You should develop and test a lot on your local machine, try to find the best solution in order to achieve the goal. Try different approches/technologies.

