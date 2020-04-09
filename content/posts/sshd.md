---
title: "sshd attack traffic"
date: 2020-04-08T20:25:03-05:00
draft: false
---
I firmly believe that security through obscurity is a fail.  However, I do
believe that all things being equal, making it a bit more obscure is better
as long as you aren't introducing more failure points, like a port knocker that
has it's own security bugs.
Thus I've always run my `sshd` service on an alternative port.  It's simple, and keeps
my logs clean and shouldn't cause any additional security risks.  Of course I
use a secure configuration and keep my software up to date.  However, I found
out that in the past few weeks that my port of choice has been discovered.

After the sad realization that I would need to pick a more random port I decided
to look at the attempts to brute force my sshd service and summarize what I
found.

#### Some basic numbers
* 161928 total attempts over the past 30 days
  * 5398/day, 225/hour, 3.75/sec
* 21380 unique user names

#### Most common user name used in login
|          Username|   #attempts|
|------------------|-----------:|
|             root |      64124 |
|            admin |       2008 |
|             test |       1846 |
|             user |       1774 |
|         postgres |       1110 |
|           ubuntu |       1004 |
|           deploy |        889 |
|              www |        862 |
|           oracle |        434 |
|             mail |        423 |
|          ftpuser |        270 |
|              git |        253 |
|            guest |        213 |
|            mysql |        206 |
|           nagios |        203 |
|           hadoop |        194 |
|        minecraft |        190 |
|           tomcat |        189 |
|        teamspeak |        170 |
|          student |        167 |
|              ts3 |        161 |
|              dev |        150 |
|         sinusbot |        143 |
|          jenkins |        132 |
|           server |        127 |
|            steam |        125 |
|              ftp |        125 |
|              vnc |        125 |
|         testuser |        122 |
|               ts |        120 |
|          support |        119 |
|             demo |        119 |
|             work |        115 |
|              web |        115 |
|             alex |        109 |
|        developer |        108 |
|           apache |        101 |
|           tester |        101 |
|            user1 |        100 |
|         www-data |        100 |
|             odoo |         97 |
|        webmaster |         96 |
|          vagrant |         96 |
|           debian |         94 |
|              tom |         94 |
|             uftp |         93 |
|               es |         90 |
|           upload |         88 |
|    administrator |         86 |
|        ts3server |         85 |
|       teamspeak3 |         84 |
|              bot |         83 |
|            david |         82 |
|           centos |         81 |
|             temp |         78 |
|          michael |         78 |
|           backup |         76 |
|             vbox |         76 |
|           ts3bot |         76 |
|           daniel |         75 |
|          mailman |         74 |
|             plex |         74 |
|         sysadmin |         74 |
|             john |         73 |
|           zabbix |         72 |
|         informix |         71 |
|     amandabackup |         70 |
|             jira |         70 |
|           robert |         69 |
|            nginx |         69 |
|               mc |         69 |
|         ec2-user |         68 |
|          testftp |         68 |
|            proxy |         67 |
|         asterisk |         67 |
|            nexus |         67 |
|              bin |         67 |
|            radio |         66 |
|          redmine |         66 |
|            sammy |         65 |
| cpanelphppgadmin |         65 |
|          couchdb |         64 |
|    gitlab-runner |         64 |
|             news |         64 |
|           cpanel |         64 |
|               at |         64 |
|          ftptest |         64 |
|               lp |         64 |
|            teste |         63 |
|     debian-spamd |         62 |
|      gitlab-psql |         62 |
|            test2 |         61 |
|             uucp |         61 |
|            test1 |         61 |
|           daemon |         60 |
|              irc |         60 |
|       gmodserver |         60 |
|             list |         59 |
|            kafka |         59 |
|             info |         59 |
|         deployer |         58 |
|               pi |         57 |
|            gnats |         57 |
|          testing |         57 |
|            vmail |         57 |
|              sam |         57 |
|           packer |         57 |
|           admin1 |         56 |
|       confluence |         56 |
|         rabbitmq |         56 |
|            chris |         56 |
|          libuuid |         56 |
|              man |         56 |
|           remote |         55 |
|            games |         55 |
|             HTTP |         54 |
|               nx |         54 |
|           prueba |         54 |
|           ts3srv |         54 |
|           master |         53 |
|               rr |         53 |
|            bruno |         53 |
|            sinus |         53 |
|       csgoserver |         53 |
|           system |         52 |
|           dspace |         52 |
|          wp-user |         52 |


#### User names that look like passwords

Seriously, if you're going to go through the hassle of trying to break in to
peoples computers, at least you could do is debug your scripts to ensure you
are sending the user name and password correctly.  These are not user names,
they are passwords and parts of passwords that have been published on
on the internet.  These all happen to be from the same IP address which is
from the *University of Dhaka, Bangladesh*.  Stay in school people, use your
skills for good, not evil!


| User name          | Number of times attempted|
|--------------------|-------------------------:|
| !qaz2WSX#edc$RFV5tgb | 1 |
| administrator@1234567890 | 1 |
| computerunabh\303\244ngig | 3 |
| qazwsxedcrfvtgbujm | 1 |
| qazwsx123456!@#$%^ | 1 |
| 1q2w3e4r5t6y7u8i9o | 1 |
| !qazxsw@#edcvfr$%t | 1 |
| 3#4$5%6^7&8*9(0). | 1 |
| 1a2s3d4f5g6h7j8k | 1 |
| 249SEPSyiae@net@IDC | 2 |

#### Top 10 attack hosts IP address and count

|    IP ADDRESS        |   Number of attempts   |  Country IP block location |
|----------------------|------------------------|-----------------------------|
|    36.153.0.228     |   591                  |  chinamobileltd.com, China, Nanjing|
|    194.61.26.34     |   181                  |  erahost.pro, Netherlands, Amsterdam|
|   185.202.1.164     |   177                  |  awesouality.com, Russian, St Petersburg|
|   185.202.1.240     |   177                  |  awesouality.com, Russian, St Petersburg|
|   123.235.36.26     |   166                  |  chinaunicom.com, China, Qingdao|
| 178.217.169.247     |   159                  |  aknet.kg , Kyrgyzstan, Bishkek          |          
| 106.253.177.150     |   150                  |  uplus.co.kr, Korea, Seoul|
|    103.35.64.73     |   137                  |  fpt.com.vn, Vietnam|
|    101.71.2.165     |   136                  |  chinaunicom.com Hangzhou, Zhejiang |
|   178.62.36.116     |   130                  |  digitalocean.com, London|


It only took 20 years for people to find my port number of choice.  I don't think
I'll have nearly that much time before they find the next one.  In the mean time I'm going to run a honeypot on port 22 and automatically collect the username, password, IP address etc. and generate automatic results.  Maybe something like [this](https://github.com/droberson/ssh-honeypot). That should keep them distracted from
looking for other ports, and maybe I'll make it into a tarpit, to slow them down too :-).
