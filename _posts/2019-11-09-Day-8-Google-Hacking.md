---

layout: post

title: Day 8 of 100 - Google Hacking - Part 1 of 2
---

Hi, guys! First of all, I would like to apologize for being so absent from posting; I'm um the middle of two big projects - one of them even includes preparing an eBook and video material for a Forensic Analysis in Linux Systems class I will be teaching in a graduate course here in Brazil (unfortunetelly, this material "won't be available for everyone" - that's what they say. :P), and te other involves preparing presentation material for an international event I will attend next week from my PhD area. However, our work here should not stop, and I have even some major projects in mind that I intend to bring you here - my own version of a Rubby Ducky with Micro SD support and my own version of Wifi Pineapple.

Next, moving forward, we'll start talking about a simple and useful method common in _OSINT Ops_ (Open Source Intelligence Operations): Google Hacking or Google Dorking. It is, in my point of view, the easiest way for _"hacking"_ and it consists basically of a computer hacking technique that uses Google Search and other Google applications to find security holes in the configuration and computer code that websites use.

All you need to perform a "Google Dork" technique usage is a computer, an internet connection and knowledge of the appropriate search syntax through Google. A number of examples are given here and if you need more, you can visit checkout [this](https://gbhackers.com/latest-google-dorks-list/) cheat sheet, [this](https://github.com/BullsEye0/google_dork_list/blob/master/google_Dorks.txt) file or ~~[this](https://bin.jvnv.net/file/qj792)~~ masterpiece book "Google Hacking for Penetration Testers", by Johnny Long. In this first part, we will talk about *"White Google Hacking"*.

---

### Brief history of Google Hacking

The concept of **"Google Hacking"** dates back to early 2000s, i think, when Johnny Long, aka _j0hnnyhax_, started collecting som interesting Google search queries that uncovered vulnerable systems and sensitive information, labeling them as _**googleDorks**_. He first posted his definition of the newly term in 2002: 

<img src="https://exposingtheinvisible.org/ckeditor_assets/pictures/595/content_googledork.jpg">

In a 2011 interview, Johnny Long said:

> “In the years I've spent as a professional hacker, I've learned that the simplest approach is usually the best. As hackers, we tend to get down into the weeds, focusing on technology, not realizing there may be non-technical methods at our disposal that work as well or better than their high-tech counterparts. I always kept an eye out for the simplest solution to advanced challenges.”

### "To DORK or not to DORK?"

If you are thinking about using Google Hackking as a regular investigative technique, there are _several_ precautions to take. Although you are free to search at-will on search engines like Google, accessing certain webpages or downloading files from them can be a prosecutable offense, according to your country laws. Moreover, if you're dorking in a country with heavy internet surveillance (i.e. any country), it's possible that your searches could be recorded and used against you in the future.

As protection, it is recommended to use Tor Browser or Tails when Google Dorking on any search engine. Tor masks your internet traffic, "separating" your computer's identifying information from the webpages that you are accessing. Using Tor will often make your searches more difficult, since Google and other search engines might ask you to solve captchas to prove you're a human. If your Tor exit node has recently been overrun with bots, search engines might block your searches entirely -  in this case, you should workaround Tor circuit until you connect to an exit node that's not blacklisted.

### Which data can we get through Google Hacking?

Basically, it is possible to get through the following data, and much more, from a Google Hacking approach:
* Usernames and passwords;
* Admin login pages;
* E-mail lists;
* Personal data;
* Vulnerable websites; etc.

**A Google Hack is nothing more than a well-placed search using one or more  advanced techniques to reveal something interesting** - something important to keep in mind: _the web can be crawled by anyone_. Google automatically indexes a website, and unless sensitive information is explicitly blocked from indexing (nofollow, robots.txt), all of the content can be searched via Dorks or advanced search operators.

Most of the time, users might post a link having no idea of what they’ve just shared. This information will be exposed in the _"referrer"_ header. Consider a web page: _"wp-content/uploads/private"_; if the browser needs to make a request to another domain to render this web page, such as an image download, a header will be included: "Referer: http://yourdomain.com/wp-content/uploads/private".

### HANDS ON: how to use these dorks?

Please, check ahead a list of dorks and their impact/result:

* **allintitle**: if you start a query with *[allintitle:]*, Google will restrict the results to those with all of the query words in the title. For instance, [allintitle:google search] will return only documents that have both "google" and "search" in the title;
* **cache**: Google will highlight words within the cached document. For instance, *[cache:www.google.com] web* will show the cached content with the word "web" highlighted. This functionality is also accessible by clicking on the "Cached" link on Google’s main results page. The query *[cache:]* will show the version of the web page that Google has in its cache;
* **define**: The query *[define:]* will provide a definition of the words you enter after it, gathered from various online sources. The definition will be for the entire phrase entered (i.e., it will include all the words in the exact order you typed them);
* **info**: The query *[info:]* will present some information that Google has about that web page. For instance, *[info:www.google.com]* will show information about the Google homepage;
* **intitle**: If you include *[intitle:]* in your query, Google will restrict the results to documents containing that word in the title. For instance, [intitle:google search] will return documents that mention the word "google" in their title, and mention the word "search" anywhere in the document (title or no);
* **inurl**: If you include *[inurl:]* in your query, Google will restrict the results to documents containing that word in the url. For instance, [inurl:google search] will return documents that mention the word "google" in their url, and mention the word "search" anywhere in the document (url or no). Note there can be no space between the "inurl:" and the following word. Putting "inurl:" in front of every word in your query is equivalent to putting "allinurl:" at the front of your query: [inurl:google inurl:search] is the same as *[allinurl:google search]*;
* **link**: The query *[link:]* will list web pages that have links to the specified web page. For instance, [link:www.google.com] will list web pages that have links pointing to the Google homepage. Note there can be no space between the "link:" and the web page URL;
* **related**: The query *[related:]* will list web pages that are "similar" to a specified web page. For instance, *[related:www.google.com]* will list web pages that are similar to the Google homepage. Note there can be no space between the "related:" and the web page URL;
* **site**: If you include *[site:]* in your query, Google will restrict the results to those websites in the given domain. For instance, *[help site:www.google.com]* will find pages about help within Google. [help site:com] will find pages about help within ".com" URL;
* **stocks**: If you begin a query with the *[stocks:]* operator, Google will treat the rest of the query terms as stock ticker symbols, and will link to a page showing stock information for those symbols. For instance, [stocks:intc yhoo] will show information about Intel and Yahoo. (Note you must type the ticker symbols, not the company name);

### Some examples of basic google dorks and their response

```
site:tacticaltech.org filetype:pdf
```
* search https://<span></span>tacticaltech<span></span>.org for all PDF files hosted under that domain name;

```
inurl:exposing inbody:invisible
```
* search websites with "exposing" in their url and with "invisible" term in body - notice that you should use quotation marks for multiple words, like ``` inurl:exposing inbody:"the invisible" ```;

```
budget filetype:xlsx OR budget filetype:csv 
```
* bring you all excel spreadsheets OR csv files that contain the word budget.

```
filetype:doc “security plan” site:gov.in
```
* locate any documents containing the words “security plan” on Indian government websites, below are the results from the following query we entered into four different search engines.

### GRAY GOOGLE HACKING EXAMPLE

```
inurl:(htm|html|php) intitle:"index of" + "last modified" +"parent directory" +description +size +(pdf) "hacking"

inurl:(htm|html|php) intitle:"index of" + "last modified" +"parent directory" +description +size +(pdf) "python"
```
* Finding PDF Files with Google Dorks:

```
db_password ===
```
* Search for website information as per passwords, website credentials details and even login of payment systems such as PayPal. It is common to find *.env* files through this search, like [this](http://crbiomed.org/.env).


### Partial Conclusion

Today, we've explored the definition, brief history and usage of google hacking, addressing yet some "white/gray" examples and pointing out references for further research on Google Hacking/Dorks. 😉

On our next post on this theme - the second and last one on this theme - we will discuss some *darker* approaches on Google Dorks. In case of any issues, feel free to contact me at any time.

#100daysofcode #day8of100 #programming #coding #code #python #developer #coder #programmer #peoplewhocode #hacking #ethicalhacking #hacktheworld #googlehacking #googledorks
