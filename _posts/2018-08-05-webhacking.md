---
name: Webhacking
date: 2018-08-05
header: 
  teaser: /assets/img/The%20Web%20Application%20Hacker's%20Handbook.jpg
---

After reading the book [The Web Application Hacker's Handbook](https://www.amazon.com/Web-Application-Hackers-Handbook-Exploiting/dp/1118026470). I thought why not make a checklist for hacking websites with tools, tips and write-ups. And so a few months later I made a checklist. It is still in progress and it needs some more information about different kind of websites like WordPress, Drupal, Jira and so on. So if you have some tips send a [pull request](https://github.com/Zawadidone/webhacking/pulls).

It covers 6 sub-tasks recon and analysis, session management, authentication, authorization,
client-side attakcs, miscellaneous tests and information disclosure. In every sub-tasks, there are tools, tips and some write-ups about the vulnerabilities of that sub-task. Here a snippet from the recon and analysis sub-tasks. If you want to see more [click here](https://github.com/Zawadidone/webhacking).

# Information Gathering
- [ ] Harvesting public information
- [ ] Automated discovery
- [ ] Automated application discovery

### Harvesting public information

Command | Description
--------|------------
Go to [Shodan](https://www.shodan.io/) -> Insert company name or domain -> Search -> Results | Use Shodan to find public ip 
Go to [Arin.net](https://whois.arin.net/ui/query.do) -> Insert company name or domain -> Under the tab Network -> Net Range | Use American Registry for internet numbers
Go to [Hurricane Electric](https://bgp.he.net/) Insert company name or domain -> Search -> Results | Use the Internet Backbone and Colocation Provider
