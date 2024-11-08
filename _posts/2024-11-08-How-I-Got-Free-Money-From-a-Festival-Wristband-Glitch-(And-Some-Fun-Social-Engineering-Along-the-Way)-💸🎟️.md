## Intro

The story kicks off with yet another wild twist in my bizarre life.
Last summer, I hit up one of Europe’s biggest metal festivals, ready for the chaos.
As I’m standing in line, waiting to slap on my festival wristband, I noticed they were running a cashless system. Now, I’ve seen this tech before—nothing new there. But this time… it felt different. Something clicked. This wasn’t just a festival; it was a challenge. And I was *all in* for the game. 🎸💸

![chilling_festival](/assets/img/chilling_destival_50.jpg)

Before we dive deeper, I have to mention the following—the company behind this cashless system ignored all four of my attempts to reach them. I waited months, offering free advice over a call to help them, but got nothing but silence in return. So, for that reason, I’ll be keeping some specific details under wraps.

Secondly—let’s just say this all went down in the middle of the festival, fueled by way too much beer and nonstop moshing, having my phone as my only weapon of choice. So, apologies in advance for any loose ends in the story! 🍻🤘

## First Impressions

The moment I got my wristband, I noticed two serial-like numbers stamped on the back of the RFID chip—interesting detail.

![censored_wristband](/assets/img/censored_wristband.jpeg)

As I queued up to load cash onto it, I watched closely: after each credit card charge, the staff would link every wristband to a generic Android device running a proprietary app. Nothing too flashy, but here’s where it got interesting—the app displayed the logo of the company running the cashless system. First clue unlocked.

![image-cashless](/assets/img/image-cashless.jpg)

I checked out the company’s website—just a static page, nothing too thrilling. But things got interesting as I dug a little deeper. I started exploring subdomains and uncovered a handful of odd, loosely secured sites: an API portal, a dev site (somehow accessible to the public), a cashless user system (we’ll get to that soon), and a broken CMS loaded with error messages exposing server file paths.

![cms_login_page](/assets/img/cms_login_page.png)

![api_hello](/assets/img/api_hello.png)

And the strangest part? Every one of these sites was hosted on the same IP address. Judging by the file paths from those CMS errors, it looked like each subdomain was just a folder away from the others. All it would take is a single vulnerability for the entire setup to come crashing down—at least from the outside, that’s how it seemed.
That’s when I called it quits on that branch. I love this festival, and I’m here to have fun, not to wreck the place!

## The Users System

Now, let’s talk about the user subdomain. This one took me to a login portal where wristband holders could enter the longer code stamped on the RFID chip to access their accounts. It was a read-only system, so there wasn’t much room for meddling—but still, a bit of fun to explore.

![login_rfid](/assets/img/login_rfid_50.jpg)

I tried logging into my own account, and it worked smoothly. Then, out of curiosity, I checked my friend’s wristband and noticed something intriguing: our codes were nearly identical, only differing by five nibbles. Some sections matched in a way that definitely didn’t seem random.

### This  Can't Just Be It

Yes, just like that—we found the first flaw! The tech felt like a throwback to the early 2000s, like I’d stumbled into a Vans Warped Tour 2005 instead of a 2024 festival.

Diving deeper into the code structure, it seemed likely that the wristband ID was composed of some "festival ID" plus a unique identifier. This unique part wasn’t even a full 5 nibbles, as certain characters didn’t use the full range of options.

Even if we stick with that—5 nibbles, or 24 bits (around 16.7 million combinations)—that’s just too limited for a festival that can easily hold 100,000 wristbands.

In practice, almost every second "random" attempt to modify a code actually yielded a legitimate account. Just like that, we could access other users’ accounts and view their action logs.

![guess_montage](/assets/img/guess_montage.jpg)

The website tried to enforce some rate limiting—after every few attempts or so, there’d be a pause. But even after a huge number of attempts, it kept working, meaning one could theoretically dump all user data in just about one day.

#### Bypassing Rate Limiting?

This raised an obvious question: does the system have any other way to verify if a wristband ID is legitimate?

The festival had an app with a feature that allowed access to livestream of the shows. Interestingly, the login for the livestream relied on the same wristband ID system, but here, I did not feel any rate limiting. To be fair, I only had my phone on me, so maybe the rate limiting was just subtle enough not to interfere with my manual entries. But this already showed enough cracks in the system to reveal some major flaws.

![rate_limit_bypass_50](/assets/img/rate_limit_bypass_50.jpg)

At this point, I downloaded a packet sniffer on my Android phone and confirmed that the requests were being sent to the same API I found in the first impressions part. It was tempting to start tweaking those requests, but I decided to hold off! I swear I am not malicious, just having fun... 😅

### Charging Other Accounts

Now, you might be wondering—*is that code really all it takes to authenticate an account?* I had the same thought. So, I grabbed an RFID scanner app on my phone, and sure enough, the longer code was simply the tag’s ID. Surprise, surprise.

![rfid_scanned_50](/assets/img/rfid_scanned_50.jpg)

Interestingly, the tags had some extra "garbage" data configured on them. This raised the question—could I tweak a nibble on my wristband and accidentally (or purposely) charge someone else’s account instead? The answer begged to be tested. But, let’s be real—I was way too drunk to go any further, and as much as I love exploring, I wasn’t about to mess with a festival I was thoroughly enjoying. Let’s just hope that "garbage data" kept everyone safe... but no promises 😅

## The Social Engineering

Two intriguing questions came to mind about what else could be done with this fascinating piece of tech.

First, how hard would it be to get physical access to the device used to load funds onto the RFID wristbands—and, by extension, how difficult would it be to obtain the app running on it?

The second question struck me during one of the most chaotic mosh pits I’d ever been in. As I was jostling with the crowd, one massive guy hoisted me up, and I ended up crowd-surfing to the front. After being pulled down, I was led backstage and told to head back into the crowd. That’s when I noticed something: the workers and musicians had different wristbands, but they also had RFID chips! I was instantly curious about grabbing one of their codes to see what secrets might be hiding in their accounts.

Later, I even snapped a photo of the guide that showed staff how to distinguish between the different wristband colors.

![all_kinds](/assets/img/all_kinds.jpg)

### Accessing Physical Devices

The first question was pretty straightforward: get access to a physical device.

As I wandered through the various booths at the festival, I spotted a potential weak link in the security chain. At the booth where festival-goers could recharge their RFID wristbands, the staff seemed like they had little connection to the metal scene or the festival vibe in general. They didn’t look like fans or insiders, making them prime targets for social engineering—people who might be more inclined to believe a convincing story or overlook a security protocol.

I headed straight over and caught the attention of one of the staff. I spun a story about how we had similar tech back home and how I wanted to see how it compared to what they were using here at the festival. Surprisingly, she just handed over her device while she got on the phone—so there I was, holding the festival’s RFID proprietary device! (You can even see my reflection on the screen 😆)

![rfid_50](/assets/img/rfid_50.jpeg)

I stood there, casually navigating the app, poking around in settings, and clicking through random options. And then, boom! I found myself in a fully functional browser that could easily upload files (and the proprietary apps) from the device. Mission success was within reach… but I stopped myself. I was just in this for fun, not to cause any harm.

### Accessing Admin Account

This part was a bit trickier: how was I going to get a code from the back of a staff member's chip?

After brainstorming with my friend over way too many beers, a worker happened to pass by our tent area. My friend sprang into action, asking, “Hey, do you work here?”—starting a friendly conversation that quickly deepened. We tossed out a bunch of casual questions, from whether they got to meet any musicians to the different levels among staff.

Then my friend went for it: he commented on how unique the staff wristband looked, grabbed the worker’s hand for a closer look, and—bam!—whipped out his phone to snap a photo of the code.

![worker](/assets/img/worker.png)

The worker, clearly caught off guard, started to say, “What the fu—,” but my friend cut him off with, “Oh, I don’t have my glasses; I need to zoom in.” While he grew suspicious, I quickly distracted him by asking about his festival shirt design. After a few minutes, we left with exactly what we needed: a staff RFID code.

So, what would this code show in the user interface? I was hyped to find out… only to discover that it led to an empty account. Anticlimactic, but still a wild ride!

![admin_account](/assets/img/admin_account_50.jpg)

## The Free Money Glitch

About one week after the festival me and my good friend sat to drink a beer and reminisce memories from the drunk times we had in the festival and we decided to try and log in to our accounts once again, only this time - a refund mechanism was offered through the site!

This makes sense - left with any change on your wristband? reclaim the money after the festival.
I remind you - we can log into random accounts.

Digging deeper, I discovered that you could reclaim the funds straight into any PayPal account of your choosing. So, with my friend’s full permission, of course, I tested it out—and just like that, his reclaimed funds landed in my own PayPal. A working proof-of-concept… FREE MONEY!

![refund_full](/assets/img/refund_full.jpg)

## Thanks and Prologue

A big, heartfelt thank you to the cashless system company for ghosting all my messages across multiple platforms, despite my multiple (and genuinely friendly!) attempts to reach out. I even offered free consulting on how they could tighten up their system—no response whatsoever.

It’s moments like these that make me think the internet just begging to be a horrible place. Stay safe out there, everyone!