---
layout: post
title:  "User System Design"
date:   2020-10-30 16:30:00
published: true
---

### Overview

Design a user system is proabably the most important part of an internet or SaaS application, In the last decade, I have seen wrong designs again and again, which costs a lot when you realize the design flaw later. For example, I used to work with a system running over 8 billion USD, which mixed user and account into one table, and it is almost impossible to separate the monolith code into two services like the user service and account service. Eventually we have to workaround on top of the wrong design, and make the system even more hacky and difficult to maintan. So, I just hope I can write something to share with you, to avoid this happen again in your early design stage.


### Naive User System

Let's start with a very simple application. This applications never requires financial accounts or customer identity, this will be a typical user system for personal customers (we will talk about enterprise customers later). Here a customer or user can register via Email or Twitter OAuth2, as the diagram below. Everything looks good enough, right?

<img src="/images/2020-10-30/user-system-naive.png" width="500px">

The problem here is, we don't know who they are. This introduces obstacles for anti-fraud. Suppose you have a marketing campaign and only new customers are eligible to some coupons. Without identity information, customers can register from both twitter and email. To stop that, you need extra algorithms to detect duplicated customers.

Another problem is, this design is often too short-sighted for the business growth. At the beginning stage you think your application does not need clear separation of customer/user/account, but once your application gets evolved with new features, especially financial features, you will run into big troubles. 

To provide financial features, you probably need KYC (know your customer) compliance before you can set up any financial accounts for you customers. That means, later you will need identity information like business license, or SSN from your customer, and only after the customer is identified and verified, you can create financial accounts for them.

### Simple User System

Once you have customer identity verified, before opening accounts for your cusotmer, you may run into a user merging problem. Because two users could be the same customer, you can know that after the user identity verification, maybe you reject to open accounts for the second user, maybe you need some sort of user merging process to be designed and implemented. The latter option is often very painful to do. E.g, if your application is a sales system and one of your customer has registered two users, for domestic and oversea sales. He already has some customers in the two users, now he wants to use some financial features requires accounts, your system need to guide him to provide migration wizards to merge the two users into one.

Anyway, the new design looks like the diagram below.

<img src="/images/2020-10-30/user-system-simple.png" width="500px">



### eTOM User System

With user identification, merging and account systems, are we done? NO. Actually we need to separate the concepts of customer and user as below.

<img src="/images/2020-10-30/user-system-better.png" width="500px">

Why is that? The reason is, as the customer's business grows, he/she could hire other people to help and it will you could provide services to more complicated customer - enterprise customers. And you'd better consider such use cases and implement a more generic user system at the early stage.

Enterprise customers are much more complicated than personal customers, because an enterprise normally is operated by multiple users, it often requires RBAC and IAM, or more complicated authentication and authorization system. Now, you are actually implementing the eTOM user system.

eTOM (enhanced Telecom Operations Map) is a framework for enterprise processes in the telecommunications industry. It is a set of standards, models and best practices. It first introduces the separation of the three concepts: Customer/User/Account.

A customer is defined as a legal entity, it can be an enterprise customer or a personal customer. A customer has to be identified or verified. There are different ways to identify a customer. E.g, a personal customer could be verified by his/her drive license or SSN against the state system, while an enterprise customer could be verified by its business license against the state tax system. In some countries, the state systems are not stable or ready, you probably also need other solutions, maybe manually. But identify verification is extremely important in SaaS, especially when you are working on financial systems, where KYC compliance is mandatory.

In most cases customers always sign up to create users in your system to login. Then the customer could create different users for different purposes in your system as the diagram below. In another word, a user could be the employees of an enterprise, group or employer, and behave/operate for the legal entity.

Below is a typical design for enterprise users, actually most IaaS/PaaS providers follow this pattern when designing their user systems.


<img src="/images/2020-10-30/user-system-etom.png" width="600px">

### Identity Verification Difficulties

If your system requires identity verification of your customers, it's actually often very difficult to implement in many cases. E,g. for personal customer, he/she could sign up several times with different identity informations like SSN, passport, drive license. Unless you have a cross verification mechanism you cannot link them, your system will end up with 3 customers and actually they are from the same person. Most countries or states do not provide such cross check services or APIs. For small business SaaS applications, it's better to keep only one method of verification; for large business entities it's not a big deal, because the ARAP per customer is worthy for you to manually verify them one by one with different identifications.


### Conclusions

Having a carefully designed eTOM-like user system at the very beginning of your business will benefit you maybe the next decade, without it, you could take much more effort to fix it in future. Hopefully this article finds you well and help you avoid mistakes that is made in so many companies.








