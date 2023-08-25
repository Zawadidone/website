---
name: Using metadata to perform reconnaissance
date: 2019-03-01
header: 
  teaser: /assets/img/metagoofil.png
---

Before hacking (red teaming, pen-testing, etc.), you must carry out a recon on the company. Almost every company has a public website with documents. These documents also contain metadata about the document itself, for example, names, emails, software etc. This kind of information could be useful in a red teaming assignment. So let's start harvesting data.

### Metagoofil  - the metadata collector

To obtain the metadata of the pubic documents we will use the tool [Metagoofil](http://www.edge-security.com/metagoofil.php). Metagoofil is a tool for extracting metadata from public documents (pdf,doc,xlsx,ppt,etc) belonging to a target. The tool is available on [Github](https://github.com/laramies/metagoofil) and on the [Kali Linux repo](https://tools.kali.org/information-gathering/metagoofil). I used Metagoofil with the following command:
```bash
metagoofil -l 200 -t pdf,doc,xls,ppt,odp,ods,docx,xlsx,pptx -d {host} -o {host} -f {host.html}
```

### Results

The results of the scan show which software packages were used to create the documents:

![Metagoofil results]({{site.url}}/assets/img/metagoofil-results.png)

With this information, you can prepare yourself making an attack plan on the employees of a company. Suppose the result of the Metagoofil scan shows that the company uses LibreOffice / OpenOffice. Then you could use a hyperlink in an ODT file to [execute code (RCE)](https://thehackernews.com/2019/02/hacking-libreoffice-openoffice.html) on the system as soon as the employees open the file and clicks on the link. So it is so easy to get valuable information from a target (company).

With this information, you can obtain valuable information about your target using "innocent" documents that are publicly available. Hopefully, this blog will help you with hacking!
