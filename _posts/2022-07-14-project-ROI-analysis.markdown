---
layout: post
title:  "Project ROI Analysis"
date:   2022-07-14 16:00:00
published: true
---

ROI analysis is not only the foundation of business decisions, it's also crucial in making technical decisions,
especially when deciding if we need to develop a new microservice or make major architectural changes. In this article, we will focus on the ROI analysis
for technical decisions.

# Investment

### Development Cost (one-time cost)

- Developers & QA
- Security, communication cost, license cost of the scan tools.
- Ops, hours to deploy it, enable firewall, apply new VMs or K8S resources.
- PM and managers, coordination and communication cost
- Arch, review and PoC.

### Maintenance Cost (annual cost)

- Developers & QA, enhance and fixes
- Ops to monitor the service
  - the incidents caused by this service, the cost for setting up monitoring and alert.
- Hardware cost on the deployment
  - if on-premises, calculate the hardware cost (suppose the hardware need to be replaced every 3-years)
  - if on-cloud, calculate the service cost (e.g., MySQL, EC2, ALB, network bandwidth)
  - if hybrid-cloud, multi-cloud, or multi-DC, sum them up.

# Return

### Operation Productivity

- Hours saved from Ops, Developer, PMs, and Managers, calculate the human resource cost

### Customer Satisfaction & Branding

- reliable and faster delivery, better reputation, adds up to our brand. If this is a beneficial feature, e.g., speed up a feature from hours to minutes and make us special among our competitors. Suppose increase xxx of our branding value.
- collect customer feedbacks, how many customers will stop the subscription if we don’t have this feature, calculate the revenue loss.
- potential referral from customers, the chance of new customers next year, increase revenue at xxx. e.g., average subscription fee times the chance of new customers

### Incident Cost Savings

- cost for solving incidents
  - Each P1 incident costs 8 hours from Ops, 16 hours from developers, 8 hours from PMs, 8 hours from Customer Support, 2 hours from Managers.
  - Each P2 incident costs ….
  - Estimate how many incidents can be avoided, and calculate the cost savings on human resource.
- cost of SLA compensation
  - If we are below SLA, how much do these incidents cost as the ratio of the compensation to customers. e.g., we compensate $10000 last year, and 30% of them can be avoided by this feature, then we save $3000 annually.
- potential loss
  - P1 incidents increase 2% of chance of customer attrition (deduce 2% of the average subscription fee)
  - P2 incidents increase 1% …
  - Estimate how many incidents can be avoided, and calculate the potential loss of customer subscriptions.

### Human Resource

- The low attrition rate, if we have developers and ops working on more challenging work than repetitive work. If we have papers published or patents filed, we can further lower the attrition rate of the related developers. Calculate the cost for recruiting, learning, and up-skilling for new employees.

# Calculation

Finally, calculate the total R and total I, calculate (R-I)/I

| timespan      | return       | investment                                                             | ROI                         | remark |
| ------------- | ------------ | ---------------------------------------------------------------------- | --------------------------- | ------ |
| 1st year ROI  | e.g., $20000 | e.g., $10000, calculated by one-time investment plus annual investment | ($20000-$10000)/$10000=100% |        |
| long term ROI | e.g., $20000 | e.g., $2000, calculated by the annual investment only                  | ($20000-$2000)/$2000=900%   |        |
