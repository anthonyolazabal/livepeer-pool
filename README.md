# Livepeer Pool

This page describe the proposal for pool implementation in Livepeer protocol. 

## Big picture

[Services diagram](https://github.com/anthonyolazabal/livepeer-pool/blob/main/Pool%20Services.png)

## API

Server less API built in Go. 
All writable endpoints require a HMAC authentication.
All readable endpoints will be accessible without authentication.
Self-signe certificated, with the DNS Name of the API. Generated a new each time the API start.

### Available endpoints:

`/recordTranscoderJob` save a transcoder job in DB (sent by an orchestrator). Parameter should be named `jobDetails`.

`/getTranscodersJobs` returns the transcoders jobs for the last 24h.

`/getTranscoderJobs` returns jobs for a transcoder for the last 24h. Parameter should be named `transcoderAddress`.

`/getRewards` returns all rewards for the last 30d.

`/getTranscoderRewards` returns rewards for a transcoder. Parameter should be named `transcoderAddress`.


## Tables
* [transcoderJobs](#transcoderJobs)
* [rewardsPayments](#rewardsPayments)

### Table `transcoderJobs`

Tracks transcoders jobs to allow statistics and fair rewards payments.

Column | Type | Description
---|---|---
createdAt | DATETIME DEFAULT CURRENT_TIMESTAMP | Time this row was inserted. 
transcoder | STRING | Address (ETH) of the transcoder that did the job.
pixels | INTEGER | Number of pixels in the job.
duration | INTEGER | Duration of the job.
profiles | INTEGER | NUmber of profiles in the job.
faceValue | BLOB | Face value of the ticket, in wei.
roundJob | int64 | The round in which the job was done.
txHash | STRING | Transaction hash of the winning ticket redemption on-chain. 

### Table `rewardsPayments`

Tracks payments to transcoders.

Column | Type | Description
---|---|---
createdAt | DATETIME DEFAULT CURRENT_TIMESTAMP | Time this row was inserted. 
transcoder | STRING | Address (ETH) of the transcoder that did the job.
value | INTEGER | Number of pixels in the job.
txHash | STRING | Transaction hash of the winning ticket redemption on-chain.

## Rewards calculation

## Payment threshold 
