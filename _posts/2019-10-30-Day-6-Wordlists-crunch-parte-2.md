---

layout: post

title: Day 6 of 100 - Wordlists with Crunch - Part 2 of 2
---

Moving on with our crunch+wordlists subject (from now on, writing in English). Since we discuessed the main basic aspects of wordlists, crunch and some special situations concerning them, this post shall be shorter - I intend to present a little mopre complex method to generete wordlists: using _charset.lst_.

### First of all, charset dot what?!

The _charset.lst_ file (which comes compiled on Kali - check /usr/share/crunch to see it) is meant to offer a predefined character lists, so you don't need to specify all the characters you want to use in your wordlist by hand in full - all you have to do is specify the reference name in the charset.lst file.

Some of the character sets offered by _charset.lst_ are:

* _lalpha_: lowercase letters only;
* _ualpha_: uppercase letters only;
* _lalpha-numeric_: lowercase letters and numbers;
* _ualpha-numeric_: uppercase letters and numbers;
* _lalpha-numeric-all-space_: lowercase letters, numbers and special characters like ?, ;, :, _space_, etc;
* _ualpha-numeric-all-space_: uppercase letters, numbers and special characters like ?, ;, :, _space_, etc;
* _mixalpha_: uppercase and lowercase letters;
* _mixalpha-numeric-all-space_: lowercase letters, uppercase letters, numbers and special characters and space.

There are some other predefined sets of characters, feel free to use `cat /usr/share/crunch/charset.lst` and see something like this:

<img src="http://euriconicacio.github.io/blog/images/d6of100_img1.png" width="100%"><br />
###### _Example of output for cat charset.lst_

### Using _crunch_ with _chartset.lst_

Thus, in order to use one of these predefined charset.lst groups to generate, for example, an wordlist of possible combinations of mixalpha characters (upper and lowercase) with 6 to 10 characters long and save its output into "WList.txt", you should use option "-f charset.lst" into your _crunch_ command, just like this:

```bash
$ crunch 6 8 -f charset.lst mixalpha -o WList.txt
```

In order to use _"@"_ with charset.lst, an example of _crunch_ command would be _(exercise: what would be the output saved into 'huge-wordlist.txt'?):

```bash
$  crunch 8 8 -f charset.lst mixalpha-numeric-all-space -t @@@@1983 -o huge-wordlist.txt
```

The "Master Wordlist" with crunch + charset.lst, grouping possible combinations between 8 and 12 characters long would be the output of:

```bash
$  crunch 8 12 -f charset.lst mixalpha-numeric-all-space -o master-wordlist.txt
```

### Conclusion

Today, we've explored the usage of crunch command to generate wordlists, potentially enhanced by its combination with the file charset.lst - even creating a colossal wordlist file. On our next post, we will discuss Google Hacking/Dorks. 😉

In case of any issues, feel free to contact me at any time.

#100daysofcode #day5of100 #programming #coding #code #python #developer #coder #programmer #peoplewhocode #hacking #ethicalhacking #hacktheworld #wordlist #crunch
