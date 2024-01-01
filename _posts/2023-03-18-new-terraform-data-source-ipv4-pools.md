---
title: 'New AWS Terraform Data Source: IPv4 Pools'
date: 2023-03-18 00:00:00
---

It’s been a while since I posted anything, so I thought I might mention one of the minor public (FOSS) projects I’ve pushed recently: [the IPv4 BYOIP pool(s) data source for the AWS Terraform provider](https://github.com/hashicorp/terraform-provider-aws/pull/28245).

My current organization has some really interesting requirements:

- We have a number of IPAM pools (groups of BYOIP IPv4 addresses which we lease directly from ARIN and provide to our customers so they can open a pinhole for our reserved address space)
- Our networking team parcels these out to us in /26 blocks (for those of you who don’t remember your v4 subnetting, that’s 64 addresses, or 62 if you omit the network and broadcast addresses)
- We have Terraform code set up to create new machines in AWS (using BYOIP addresses from out IPAM pools), and we need to create significantly more than 62 machines with this code

Given existing Terraform AWS provider code, we had to create a separate terraform file for each BYOIP/IPAM pool, because we had no way to reference either a) all of our various IPAM pools, or b) any one of them.

For this reason, I created a module that accesses an existing API endpoint in the AWS SDK to provide either a single BYOIP pool or (conceivably) all BYOIP pools connected with a particular account. Though the API endpoint has existed for a little while now, it was not exposed to Terraform previously, so I added:

1. A new Finder method (to look up BYOIP pools)
2. A new IPv4 Pool Data Source (that can find a BYOIP pool by ID)
3. A new IPv4 Pools Data Source (that will find all BYOIP pools)

This was really fun to work on, and I hope it makes life easier for someone else out there!
