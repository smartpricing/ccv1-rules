# V1 Coding Challenge Rules

## Overview

This coding challenge consists of creating an HTTP application server (bindend on port 3000) that exposes a POST route, which takes an integer as input and computes its Collatz conjecture, returning the result. The application must be deployed on a virtual machine, with the goal of maximizing the number of requests per second that the server can handle.

**The choice of programming language or framework is entirely up to you.**

Once the server is deployed on the VM, you must register (using the HTTP requests provided below) with the machine that will perform the load tests.

When you request a test, your request will be placed in a queue and executed when the load testing machine is free, ensuring that only one test is run at a time and that the same bandwidth is allocated to each participant.

Once you have performed one or more tests, you will be able to see your detailed results and view the current leaderboard.

The score calculation will take into account the number of correct requests made within the time unit, the number of incorrect requests, and the number of attempts made by the user.

The score is calculated as follows (the higher the better):

```js
score = ((number_of_request_completed_in_interval / 10^6) + 10^6 / (test_count)) / (1 - number_of_wrong_results)
```

![Infra](assets/infra.png?raw=true "Infra")

### Locally develop your solution

In order to test the correctness of your application server, do this test (on your local machine):

```sh
curl -X POST -H 'Content-Type: application/json' -d '{"numbers":"1", "requestId": "XXX-XXX-XXX-XXX"}' 0.0.0.0:3000/collatz
```

Must return (for instance):

Status code: 

```sh
HTTP/1.1 200 OK
```

Headers:

```sh
Content-Type: application/json;charset=utf-8
Date: Fri, 07 Feb 2025 07:47:55 GMT
Content-Length: 52
```

Body:

```sh
{"output":[1,4,2,1],"requestId":"XXX-XXX-XXX-XXX"}
```

### Load test info

The loader machine will target your VM with millions of request, using multiple connections, for a time interval of 30 seconds, with an initial warmup time of 3 seconds.

## Setup the test enviroment

### 1. Create the VM on DO

Using the doctl, create the specified VM on the coding challenge vpc:

```sh
export SP_YOUR_MACHINE_NAME="cc-alice-v1"

doctl compute droplet create \
    --image ubuntu-22-04-x64 \
    --size c-4-intel \
    --region fra1 \
    --vpc-uuid 1f542d99-824b-447e-ad22-c5afe2448833 \
    --tag-names 'coding-challenge' \
    $SP_YOUR_MACHINE_NAME
```

*This machine has 4 vCPU and 8GB of RAM, and it cost 108$/month.*

You will be able to connect to the machine via ssh. All other ports (to and from the external internet) will be blocked by the firewall.


### 2. Register yourself to the challenge server

Once inside the VM, first find your IP address (note that these machines can have multiple network interfaces and you will find both the internal VPC address and the public one, use the internal).

There is a global registration token that you have to use only to register your machine to the loader machine.

*token*

```sh
export SP_EMAIL="XXX@smartpricing.it"
export SP_REGISTRATION_TOKEN="YYY"
export SP_TEST_MACHINE_IP=Z.Z.Z.Z
```

Register to the loader machine:

```sh
curl -X POST -H 'Content-Type: application/json' -d '{"user":"$SP_EMAIL", "token": "$SP_REGISTRATION_TOKEN", "ip":"$SP_TEST_MACHINE_IP"}'  10.114.16.2:2999/cc/v1/register
```

and save the returned personal token that you will have to use in all other requests.

### 3. Setup your application server and test it

Once your application server is up and running on the VM, you can try the connection between the loader machine and your VM (this operation is not counted in the loading count used in the score):

```sh
export SP_TOKEN="AAA"

curl -X POST -H "Authorization: Bearer $SP_TOKEN" 10.114.16.2:2999/cc/v1/test
```

Response

## Require a loading test

```sh
curl -X POST -H "Authorization: Bearer $SP_TOKEN" 10.114.16.2:2999/cc/v1/run
```

## Leaderboard

To retrive the global leaderboard:

```sh
curl -X GET -H "Authorization: Bearer $SP_TOKEN" 10.114.16.2:2999/cc/v1/leaderboard

alice.setti@smartpricing.it	1000	2025-02-06T09:07:54.048Z	1367min ago
aliceviola@smartpricing.it	900		2025-02-06T09:08:43.325Z	1367min ago
```

To retrive only your test statistic:

```sh
curl -X GET -H "Authorization: Bearer $SP_TOKEN" 10.114.16.2:2999/cc/v1/my
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

