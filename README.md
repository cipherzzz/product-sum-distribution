
# Efficient Distributions in Defi

Building in defi requires constant attention to the compute and storage limits of the particular chain you are deploying to. In some ways, you are running your code on an IBM x86 machine with very small resources.

![Remember these….?](https://cdn-images-1.medium.com/max/2000/0*EwkcIFlfpTEAZL58.jpg)*Remember these….?*

Oftentimes, the naive approach for distributing rewards, debt, liquidation proceeds, etc. is to iterate over the vaults or principals and assign their proceeds to a mapping in storage. This works fine on a bounded set, but what happens when you are working with an unbounded set? Enter efficient distributions.

## The Basics

There are two primary mechanisms for handling efficient distributions.

### Sum

This is the simplest version, where you are calculating a point in time distribution based on a balance that is static.

### Product-Sum

This is similar to the **Sum **approach but allows you to calculate a point in time distribution based on a balance that is changing.

## The Maths

There are already some great resources that explain the math and understanding behind the above two topics.

*Rick Pardoe* has a very comprehensive post on the **Liquity **model and you should really read and try to digest it before continuing in this article.
[**Scaling Liquity’s Stability Pool**
*A central component of Liquity is the Stability Pool - the system’s first line of defense against liquidations. It acts…*medium.com](https://medium.com/liquity/scaling-liquitys-stability-pool-c4c6572cf275)

The original whitepaper written by Rick can be found here —

[https://github.com/liquity/liquity/blob/master/papers/Scalable_Reward_Distribution_with_Compounding_Stakes.pdf](https://github.com/liquity/liquity/blob/master/papers/Scalable_Reward_Distribution_with_Compounding_Stakes.pdf)

## The Setup

My project, [bsd.money](http://bsd.money) is a lending protocol built on the Stacks chain. We utilized a lot of the mechanisms of the liquity project, but since stacks is not EVM compatible — they had to be written in Clarity. While the above documentation is really good, it only got me about 80% of the way there when it came to translating it to a smart contract. What I decided to do was to prototype a simple project in Typescript to verify the concepts and then translate to Clarity.

## The Prototype

I built the following project to help cement my understanding and to run scenarios to check the calculations
[**GitHub - cipherzzz/product-sum-distribution: A Simplified Product/Sum Efficient Rewards…**
*A Simplified Product/Sum Efficient Rewards Distribution example in Typescript - cipherzzz/product-sum-distribution*github.com](https://github.com/cipherzzz/product-sum-distribution)

    git clone https://github.com/cipherzzz/product-sum-distribution
    cd product-sum-distribution && npm i
    npm run smoketest

The output of the above smoketest command will yield

    ┌───────────────────────────────────────┐
    │                                action │
    ├───────────────────────────────────────┤
    │                 Added provider: Alice │
    │                   Added provider: Bob │
    │ Deposited 100 BSD for provider: Alice │
    │    Deposited 50 BSD for provider: Bob │
    │     Withdrew 15 BSD for provider: Bob │
    │       Liquidated 50 BSD for 0.01 SBTC │
    │       Liquidated 25 BSD for 0.01 SBTC │
    │   Deposited 100 BSD for provider: Bob │
    │       Liquidated 10 BSD for 0.01 SBTC │
    └───────────────────────────────────────┘
    ┌──────────┬─────────────────────────┬───────────────────────┬─────────────────────────┬─────────────┐
    │ provider │                 rewards │               balance │             poolRewards │ poolBalance │
    ├──────────┼─────────────────────────┼───────────────────────┼─────────────────────────┼─────────────┤
    │    Alice │ 0.017592592592592592593 │ 41.666666666666666667 │ 0.024814814814814814815 │         150 │
    │      Bob │ 0.007222222222222222223 │ 108.33333333333333334 │ 0.024814814814814814815 │         150 │
    └──────────┴─────────────────────────┴───────────────────────┴─────────────────────────┴─────────────┘

This is a simple example of two stability providers (Alice and Bob) depositing and withdrawing from the pool while liquidations are occurring.

## Product-Sum

I’ll just highlight the important parts of the product-sum approach in the prototype and let you peruse the rest of the repo at your leisure.

### Global Product & Sum

You must maintain a global product and sum variable that is ONLY changed during liquidation.

![](https://cdn-images-1.medium.com/max/2000/1*N4pwcc6HhehbV3Uqw42AVQ.png)

As you can see the default values for product and sum are 1 and 0 respectively and are changed during liquidation as shown below.

![](https://cdn-images-1.medium.com/max/2000/1*_FE3tTj4il8ZIBweeYDOHw.png)

Note that it is important for these values to be calculated BEFORE the subsequent balance updates.

### Provider Product/Sum

We also must maintain an individual provider’s product and sum value based on their current balance and share of the pool. The initial values for product and sum for a provider are always the global values.

![](https://cdn-images-1.medium.com/max/2000/1*BCNa_aNOYT2utyYL-mV5lw.png)

Additionally, these provider-specific values are changed whenever a provider adds or removes from their stability balance. The current rewards owed to that provider must be earmarked or sent to the provider at every balance change.

![](https://cdn-images-1.medium.com/max/2498/1*59GOrHi2a1f-cRWLD-s7gA.png)

![](https://cdn-images-1.medium.com/max/2528/1*e4x4vvhoURObVcpZUQDWKQ.png)

Note that the general flow here is to

* Calculate Existing Rewards owed to the provider(Implicitly Send to them)

* Update the Pool’s rewards balance

* Calculate the Provider’s new balance

* Update the Provider’s balance and reset the Product and Sum

The calculations are given below for your convenience:

![](https://cdn-images-1.medium.com/max/2000/1*rnf_EnFxIT2VYaZC5HHHMQ.png)

## Summary

Note that this project is not production code — it is hopefully just some working code to help you get your head around efficient distributions in the wild.
