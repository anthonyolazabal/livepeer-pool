# Livepeer Pool

This page describe the proposal for pool implementation in Livepeer protocol. 

## Big picture

![Services diagram](https://github.com/anthonyolazabal/livepeer-pool/blob/main/Pool%20Services.png)

## API

All writable endpoints require a HMAC authentication.
All readable endpoints will be accessible without authentication.
Self-signe certificated, with the DNS Name of the API. Generated a new each time the API start.

### Available endpoints:

`/recordTranscoderJob` save a transcoder job in DB (sent by an orchestrator). Parameter should be named `jobDetails`.

`/getTranscodersJobs` returns the transcoders jobs for the last 24h.

`/getTranscoderJobs` returns jobs for a transcoder for the last 24h. Parameter should be named `transcoderAddress`.

`/getRewards` returns all rewards for the last 30d.

`/getTranscoderRewards` returns rewards for a transcoder. Parameter should be named `transcoderAddress`.

`/getPoolStatistics` returns pool statistics (Jobs, Payments, Transcoders, Winning Tickets, optionnally but nice to have Orchestrator informations).


## Tables
* [transcoderJobs](#transcoderJobs)
* [rewardsPayments](#rewardsPayments)

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
value | DECIMAL | Number of pixels in the job.
paymentStatus | STRING | Status of the payment (On hold/Paid).
txHash | STRING | Transaction hash of the winning ticket redemption on-chain.

## Rewards calculation
Reward calculation routine will be triggered once per round. If during the previous round a winning ticket was won, the split will be calculated.
Pool Fees cut need to be take in account. 
In order to fairly share the rewards, all aspect of a job needs to be take in account : Duration, Pixels, Number of profiles.
For each criteria a weight is going to be added, and the total will be saved in the db for each job.

Examples :
- A job with 10s will have a weight of 10 and a job of 5s will have a weight of 5. 
- Same for the pixels, the more pixels you transcode, the more wieght you get. 
- This apply to profiles too, one profile -> weight 1 and 4 profiles -> weight of 4. 

Weights needs to be challenged to fairly equilibrate between the criterias.

Once the weight for each job has been calculated, when can calculate the split of the reward(s) : 

Share of the transcoder in the distribution = Total weights of jobs for the transcoder since last rewards calculation
 / Total weights of jobs since last rewards calculation

This will give us the percentage of the reward to distribute for each transcoder.

## Payment threshold 
Payments for transcoders will only be released when the total of payments come to a certain amount to avoid paying big gas fees for small amuonts.

## Livepeer program update needed 
In order to collect jobs information, we need to update the Livepeer program to achieve the following : 
- On the transcoder side, be able to start specifying a ETH address for payment (may already existe with the flag `-ethAcctAddr` but need to be verified)
- On the orchestrator side, handle the ETH address sent by the transoder along with informations regarding the job and send it to the pool API. Jobs informations should include the informations described in the `transcoderJobs` table (except for the weight, createdAt).
