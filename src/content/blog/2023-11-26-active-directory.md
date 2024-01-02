---
title: Active Directory Essentials
pubDatetime: 2023-11-26
draft: true
tags: ["AD"]
description: In this article i will be covering the active directory pentesting essentials.
---

## Table of contents

## What is active Directory

Active Directory is a directory service developed by Microsoft for Windows domain networks.

## Active directory structure

A **Windows Domain** is a group of users and computers under the administration of a business.

The server running the AD services is called **Domain Controller**.

AD is mainly done to centralize identity management and managing security policies.

There are different users, machines and security groups inside an AD.

Machines account will be name adding a `$` at the end of the machine name.

Here are the main default groups in a domain:

- Domain Admins: Users of this group have administrative privileges over the entire domain. By default, they can administer any computer on the domain, including the DCs.
- Server Operators: Users in this group can administer Domain Controllers. They cannot change any administrative group memberships.
- Backup Operators: Users in this group are allowed to access any file, ignoring their permissions. They are used to perform backups of data on computers.
- Account Operators: Users in this group can create or modify other accounts in the domain.
- Domain Users: Includes all existing user accounts in the domain.
- Domain Computers: Includes all existing computers in the domain.
- Domain Controllers: Includes all existing DCs on the domain.

Object are organized in **Organizational Units (OUs)**. OUs are handy for applying policies to users and computers. Security Groups are used to grant permissions over resources.

Domains -> Trees -> Forest -> Trust relationships

## Kerberos 

Kerberos is the authentication method in any modern Windows machines. It is a ticket-based mechanism, where each ticket is used as a proof of previous authentication.

Users with tickets can present them to a service to demonstrate they have already authenticated into the network before and are therefore enabled to use it.

The process is as follows:

1. Client request TGT from KDC.
2. Authentication service sends encrypted TGT and session key.
3. Client requests server access from TGS.
4. TGS sends encrypted session key and ticket.
5. Client sends service ticket.
6. Server sends encrypted time stamp from client validation.

## Group policies

Group Policies Objects (GPOs) are a collection of settings that can be applied to OUs.

GPOs are distributed to the AD computers via a network share called SYSVOL.
