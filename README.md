# Livepeer Pool

This page describe the proposal for pool implementation in Livepeer protocol. 

## Big picture

![Services diagram](https://github.com/anthonyolazabal/livepeer-pool/blob/main/Pool%20Flow.png)

## API

All writable endpoints require a HMAC authentication.
All readable endpoints will be accessible without authentication and return JSON data.
Self-signe certificated, with the DNS Name of the API. Generated a new each time the API start.

### Available endpoints:

`/recordTranscoderJob` save a transcoder job in DB (sent by an orchestrator). Parameter should be named `jobDetails`.

`/recordWinningTicket` save information regarding a winning ticket in DB (sent by an orchestrator). Parameter should be named `winningTicket`.

`/getTranscodersJobs` returns the all transcoders jobs for the last 24h.

`/getTranscoderJobs` returns jobs for a transcoder for the last 24h. Parameter should be named `transcoderAddress` (ETH address).

`/getRewards` returns all rewards for the last 30d.

`/getTranscoderRewards` returns all rewards for a transcoder. Parameter should be named `transcoderAddress` (ETH address).

`/getTranscoderRewards` returns rewards for a transcoder. Parameter should be named `transcoderAddress`.

## Tables
* [transcoderJobs](#transcoderJobs)
* [rewardsPayments](#rewardsPayments)
* [winningTickets](#winningTickets)
* [logs](#logs)

### Table `transcoderJobs`

Tracks transcoders jobs to allow statistics and fair rewards payments for transcoders.

Column | Type | Description
---|---|---
createdAt | DATETIME DEFAULT CURRENT_TIMESTAMP | Time this row was inserted. 
transcoder | STRING | Address (ETH) of the transcoder that did the job.
pixels | INTEGER | Number of pixels in the job.
duration | DECIMAL | Duration of the job.
profiles | INTEGER | Number of profiles in the job.
faceValue | BLOB | Face value of the ticket, in wei.
roundJob | int64 | The round in which the job was done.
weight | DECIMAL | Weight of the job (calculated by the payment routine at the end of a round).

### Table `rewardsPayments`

Tracks payments to transcoders.

Column | Type | Description
---|---|---
createdAt | DATETIME DEFAULT CURRENT_TIMESTAMP | Time this row was inserted. 
transcoder | STRING | Address (ETH) of the transcoder that did the job.
value | DECIMAL | Amuont of reward.
gasFees | DECIMAL | Amuont of gas fees.
paymentStatus | STRING | Status of the payment (On hold/Paid).
txHash | STRING | Transaction hash of the winning ticket redemption on-chain.

### Table `winningTickets`

Tracks orchestrator winning tickets.

Column | Type | Description
---|---|---
createdAt | DATETIME DEFAULT CURRENT_TIMESTAMP | Time this row was inserted. 
wonAt | DATETIME DEFAULT CURRENT_TIMESTAMP | Date of the won ticket. 
value | DECIMAL | Amuont of reward (Should alwards be 0.18 ETH but might change if additional chains or tokens are added).
gasFees | DECIMAL | Amuont of gas fees (Optional).
paymentStatus | STRING | Status of the payment (Not claimed/Claimed).
txHash | STRING | Transaction hash of the winning ticket redemption on-chain (Optional).

### Table `logs`

Logs table to track pool process activities.

Column | Type | Description
---|---|---
createdAt | DATETIME DEFAULT CURRENT_TIMESTAMP | Time this row was inserted. 
key | STRING | Log identifier.
value | STRING | Log value.

## Rewards calculation
Reward calculation routine will be triggered once per round. If during the previous round a winning ticket was won, the split will be calculated.
Pool Fees cut need to be take in account. 
In order to fairly share the rewards, all aspect of a job needs to be take in account : Duration, Pixels, Number of profiles.
For each criteria a weight is going to be added, and the total will be saved in the db for each job.

Examples :
- A job with 10s will have a weight of 10 and a job of 5s will have a weight of 5. 
- Same for the pixels, the more pixels you transcode, the more weight you get. 
- This apply to profiles too, one profile -> weight 1 and 4 profiles -> weight of 4. 

Weights needs to be challenged to fairly equilibrate between the criterias.

Once the weight for each job has been calculated, we can calculate the split of the reward(s) : 

Share of the transcoder in the distribution = Total weights of jobs for the transcoder since last rewards calculation
 / Total weights of jobs since last rewards calculation

This will give us the percentage of the reward to distribute for each transcoder.

## Payment threshold 
Payments for transcoders will only be released when the total of payments come to a certain amount to avoid paying big gas fees for small amuonts.

## Performances of transcoder
One of the major concerns in having a public pool is the performances of the transcoders since it can make the orchestrator excluded of broadcasters selection list. In order to limit the impact on the pool, a benchmark at earch transcoder startup should be done when starting with the flag `-ethAcctAddr`. Benchmark should measure : 

- Transcoding performances (based on the existing benchmark from Livepeer)
- Latency performances
- Download and Upload performances

Depending on the results, the maxSessions will be automatically defined to bring the best performances to the pool. 
In a future release we can also define the transcoding options based on the performances for the different profiles.

Total duration of transcoding has also to be taken in account, one way could be to set a limit of (for example) 800 ms for a transcoder to download, transcode and upload the result. If the trancoder is performing above this limit, it can be excluded from the pool.

## Livepeer program update needed 
In order to collect jobs information, we need to update the Livepeer program to achieve the following. 

On the transcoder side : 
- Be able to start specifying a ETH address for payment (may already existe with the flag `-ethAcctAddr` but need to be verified). 
- Benchmark transcoding performance on each startup to define the maximum of sessions based on the results of the benchmark.
- Send result of transcoding tests to orchestrator

On the orchestrator side : 
- Handle the ETH address sent by the transoder along with informations regarding the job and send it to the pool API. Jobs informations should include the informations described in the `transcoderJobs` table (except for the weight, createdAt). 
- Add a flag `-poolAPI` to specify the endpoint that receive the jobs informations. If flag `-poolAPI` is set, orchestrator sent the relevant informations on the following events : transcoder jobs validated, winning ticket, rewards claimed.
- Be able to exclude a transcoder if it is performing under the limit (the exclusion take place on the transcoder registration when it sends the result of the benchmark)
