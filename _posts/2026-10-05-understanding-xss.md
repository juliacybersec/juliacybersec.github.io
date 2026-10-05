
"Understanding Cross-Site Scripting (XSS) in Plain English"
 What is XSS? Cross site Scripting is a vulnerability where an attacker puts malicious java script into a page viewed by others, by putting a script tags in a forum
 comment that then gets run like it executes on every viewers individuals browser.
 
 what happens, and why it's called "cross-site scripting". Its called cross site scripting because it executes in a different site than authored. Cross site meaning that
 it runs in other site or sites than say the original forum. So the victims browser can tell the injected script from its own. 

 An analogy
So we have a restaurant and the kitchen is the site. The customer slips a fake ticket to the waiter telling the kitchen to bring vodka but the kitchen cant tell if
it's from the legitimate tickets. 

 Types of XSS
Stored XSS:** <!-- Where is it stored, who does it hit? Use your forum example. -->Stored XSS 
The malicious script is sored in the database it gets stored along with every other comment when the attacker posts it on the forum. 
Its so sneaky here is why it hits every visitor who views that comment on the forum.

Reflected XSS:** <!-- How does it reach the victim? This one is not as sneaky. the payload only fires when a visitor clicks on it.
The attacker put up the link up once and it only stops when someone realizes and removes it so no one clicks on it again. Not as sneaky as Stored XSS

DOM-based XSS: the script is triggered entirely in the browser by the page's own JavaScript, never touching the server.
The vulerability lived in the client server js and payload never reaches the server. The https server response is clean. It runs the attack after the page loads.
DOM based is the hardest one to detect

 What attackers can do with it
 Can steal session cookies and login as them the httpOnly blocks this, keylogging, fake login box attacker can modify the page content, act as the user 
Redirect can send the user to a phishing page.
Install a back door even if the payload is removed that backdoor can still be wide open. 

XSS is so nasty it doesn't need to get to the server it just need the victim to view it and then it can infect others or piviot. 

How to defend against it

Input validation: If your site asks for age only allow numbers but not 3 or 110 limit it to only what you want the user to put. 
Input validation is the type of data i expect but any one can bypass with curl so expect 
Output endcoding is this safe to render in the specific context. encoding is the actual security boundary to prevent scripts running

Content Security Policy (CSP):One rule to own them all if we were in middle earth no really its just a http response header that tells the browser what to run. which
scripts its allowed to run. 
Like a rule dont run stupid programs. i have a CSP Dont do stupid stuff but do run anything you like in my VM. 
 

Cookie flags: <!-- HttpOnly, Secure, SameSite, and why HttpOnly limits damage but doesn't stop XSS. -->

 What I learned
We are more vulnerable than we think and most people have no idea how exposed they are. Most people treat the internet as Im safe behind my screen or i have done
my software update. I see people entering their correct personal details on every website and I wonder. 

It reinforces why i spend as little time on the internet and on web pages why my dog is on the internet and social media and i am not. 
