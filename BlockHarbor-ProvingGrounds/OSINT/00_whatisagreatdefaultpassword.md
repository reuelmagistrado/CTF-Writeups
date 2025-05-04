
# BlockHarbor Proving Ground - what is a great default password?

**Category:** OSINT<br>
**Points:** 1

## TL;DR

This simple OSINT challenge asked for a common default password. I searched for the most frequently used passwords worldwide and found "password" on NordPass's list of common passwords. Simple but effective reminder of why default passwords are dangerous!

## Challenge Description

Minus 1 million points if this is your actual password.

## Initial Analysis

When I first saw this challenge, I immediately recognized it as a classic OSINT task requiring me to find commonly used default passwords. Given the challenge name "what is a great default password?" and the warning about losing points if it's my actual password, I knew I needed to find the most obvious, commonly-used default password.

## Solution Process

### Reconnaissance

My first thought was to look for authoritative sources that track password usage statistics. These kinds of lists are published annually by cybersecurity companies and password managers who analyze leaked credentials.

I decided to check NordPass's list of most common passwords since they regularly compile comprehensive data on this topic. Their research typically analyzes millions of passwords from various data breaches.

Google: nordpass most common passwords

 - [Top 200 Most Common Passwords | NordPass](https://nordpass.com/most-common-passwords-list/)

When I visited the NordPass website, I found an interesting note that really stood out:

> “123456” wins (again)
“123456” has once again claimed the title of the world’s worst password! In fact, during this our six-year study, it topped the charts as the most common password 5 out of 6 times. “Password” held this not-so-noble title just once.

This was particularly interesting because it showed that while "123456" was generally the most common password over time, "password" had actually taken the top spot once during their studies. This gave me confidence that "password" would be recognized as one of the quintessential default passwords that security experts warn against.

Looking at their most recent rankings, I could see that both "123456" and "password" consistently appeared in the top positions, making them prime candidates for the challenge answer.


### The Breakthrough

The lightbulb moment came when I decided to systematically work through the password list by NordPass. I began trying the passwords in order of their ranking on NordPass's list. After trying the first few passwords like "123456" and "123456789", I hit success when I tried "password" - which was ranked #4 on their list. This methodical approach paid off, confirming that the challenge creator had indeed used one of the most notoriously common default passwords as the answer.

### Verification

To be thorough, I also checked some other sources to confirm this information. While the rankings change slightly year to year (with "123456" sometimes taking the top spot), "password" consistently ranks at or near the top of these lists, making it a classic example of a terrible default password.

## Flag

The flag for this challenge was: `password`

## Key Takeaways

This simple challenge was a good reminder about the dangers of using obvious default passwords. It's shocking how many people and even businesses still use "password" as their actual password!

The fact that this was worth only 1 point shows how basic this security knowledge should be, yet millions of accounts are still vulnerable because people ignore this advice. This challenge reinforced for me that even the most basic OSINT skills can be crucial for security work.

Pro tip: Always change default credentials and use a password manager to generate and store unique, complex passwords for each account. Your future self will thank you when you don't lose a million points... or worse, your actual data!