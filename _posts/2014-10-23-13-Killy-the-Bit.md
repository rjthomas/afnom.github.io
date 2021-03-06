---
layout: post
title: "Killy the Bit"
date: 2014-10-23
ctf: Hack.lu 2014
author: Joe Gardiner
---
## Background

This challenge came with the following description:

"Killy the Bit is one of the dangerous kittens of the wild west. He already
flipped bits in most of the states and recently hacked the Royal Bank of
Fluxembourg. All customer of the bank are now advised to change their password
for the next release of the bank's website which will be launched on the
23.10.2014 10:01 CEST.

Killy the Bit stands in your debt and sent the following link. Can you break
the password generation process in order to get access to the admin account?"

Following the provided link leads to a single page
(https://wildwildweb.fluxfingers.net:1424/)

## Page

The user is then presented with a web page. The page consists of some simple
text, and then a basic form consisting of a single text box and submit button.

The user is asked to submit their username, and their new password will then
be sent to them. If a correct username is enetered, a page with a confirmation
message is displayed, stating that the new password has been sent by email. If
the user enters an incorrect username, the page returns a list of username
that are similar to the provided name.

This page is serviced by a php script, which is provided with the code that
generated the password redacted. The index.phps file can be found
[here](http://afnom.net/assets/2014/killy-the-bit_index.phps) (right click -&gt; save as)

## SQL Injection

It was clear after looking at the php file that SQL injection is the key here.
There are two different queries that are used.

#### Query 1

The first query is a simple SELECT query, in which a straightforwad lookup for
the username returns the name and email.

	$res = mysql_query("SELECT name,email FROM user where
	name='".$_GET['name']."'");

  
The script then checks to see if ANY result is returned, and if so the
confirmationmessage is displayed.

#### Query 2

If no result is returned, as second query is performed. This makes use of the
"sound like" operator to find similar usernames

  
	$res = mysql_query("SELECT name,email FROM user where name sounds like '".$_GET['name']."'");

  
If any exist, they are printed to the screen, else an error message is
displayed:

  
	while($row = mysql_fetch_object($res)) {
		echo $row-&gt;name;
		echo "&lt;\br&gt;"; ("\" added by me)
	}

#### Attack

At first glance, the second query seems to be the one to attack, by somehow
getting the code to print out the passwd column as well(the column name was
revealed in a hint).

One limitation is a line near the top of the file:

	if(isset($_GET['name']) &amp;&amp; $_GET['name']!='' &amp;&amp;
	!preg_match('/sleep|benchmark|and|or|\||&amp;/i',$_GET['name'])) {

  
This line prevents certain strings being passed as parameters. Most
annoyingly, this include `and` and `or`, which would be useful here.

A join seems like one solution, but we cannot use this as we can only append
to the current query after the `WHERE` clause. Therfore, a `UNION` seems like a
better option.

The first attck string that we tried was as follows:

  
	' UNION SELECT email, passwd AS name FROM user WHERE#

  
In theory, this should work, but we kept finding that this attack string
results in Query 1 passing, and Query 2 not being run so there is no scope to
print the passwords. We discovered that we would then need to form some form
of attack string that would cause the first query to fail, but the second to
pass.

This proved difficult. Instead, Chris realised that we could use ther
pass/fail nature of the first query to our advantage to brute force guess the
password.

We made use of the SUBSTR function in SQL to achieve this. If we can match a
substreing of a password to a string provided by us, we could confirm if it
wss correct.

We tested this with the following attack string:

	' UNION SELECT email, passwd FROM user WHERE SUBSTR(passwd,1,5)='flag{'#

We used this as an intial guess as in all of the challenges we had completed,
the flag beings with "flag{". Inputting this into the username box resulted in
the confirmation screen! We then manually tried to guess the next letter by
brute force, and it revealed th enext letter was k.

Obviously the flag could contain any character, so me and Chris simultaneously
worked on our own scripts to do this manually. My script was writted in Java,
Chris' in Perl. The Java script made a request to the challenge domain, with a
constructed GET request. The key line was: 

	url = new URL("https://wildwildweb.fluxfingers.net:1424/?name=%27+UNION+SELECT+email%2C+passwd+FROM+user+WHERE+SUBSTR%28passwd%2C" + pos + "%2C1%29%%27" + look + "%27%23&amp;submit;=Generate#");

(This is now slightly incorrect but you get the idea:))

The response was then parsed to look for the key string "A new password was
generated", if this was found the current letter was added to the flag. The
request included the previosuly found letters at each step.

This worked fine, and we found the partial flag
`flag{killy_the_bit_is_wanted_fo`. The script then stopped working at this
step, and we were confused as to why. EWe guessed the next letter was R, but
maually trying this resulted in the php code not even being run. We soon
realised the issue. When the "r" is tried, the regular expression matcher the
"or" of the "for", and rejects the request! So we then had to guess the rest
of the password ingoring the first half. This resulted in the full flag:

	flag{killy_the_bit_is_wanted_for_9000_$$_for_flipping_bits}

Chris was the first to get the full flag by seconds. But this was not accepted
by the form. We suspected that it was an issue with capitilisation. The
problem is that in SQL, the = operator is case-insensitive, meaning that we
cannot match the exact capitilaisation using this. We then remembered the
ImageUpload challenge, where we used thr ASCII operator to fetch the passord.
We then modified out attack string to resemble:

  
	' UNION SELECT email, passwd FROM user WHERE ASCII(SUBSTR(passwd,32,1))=ASCII('r')#1

  
This checks that the 32nd character is a lower case r, and only returns a
result if so. We modified the script to check individual characters, and soon
found the capitalisation for the 15th character onwards. For the first 15
characters, the request matched a different password in the table, so we had
to manually guess the capitilisation for these (which I got first time). The
final flag was:

  
	flag{Killy_the_Bit_Is_Wanted_for_9000_$$_FoR_FlipPing_Bits}

  
In this step, the Java script won the race. The Java script can be found
[here](http://afnom.net/assets/2014/killy-the-bit_URLFetcher.java).

A later hint by the admins stated that a single query can return the password
and brute forcing is not necessary, but we were still pleased to solve this
challenge. Overall it was a nice challenge to complete even though at first it
seemed a lot easier than it actually was.

This challenge was largely completed as a joint effort by myself and Chris
Novakovic (csn), who was working remotely. Also involved on site were Tom
Chothia and Jiangshan Yu who were valuable in solving this challenge, which
earned us 200 points plus 70 bonus points.

