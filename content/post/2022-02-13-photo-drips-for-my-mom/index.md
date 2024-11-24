---
title: "Yak shaving - Photo drips for my mom"
description: "TL;DR: I finally organized my photo archive and in an evening created a service to e-mail my mom a photo from the last 20 years every morning."
image: 2022-02-13-photo-drips-for-my-mom.png
date: 2022-02-13 00:00:00 +0000
categories:
    - technology
    - side-quests
excerpt_separator: <!--more-->
---

# Yak shaving - Photo drips for my mom

Update: Check out https://github.com/StianOvrevage/photo-drips for ugly but working code.

> Update May 2022: My mom told me this week that I must NEVER stop sending these daily photos <3

## Background

__TL;DR: I finally organized my photo archive and in an evening created a service to e-mail my mom a photo from the last 20 years every morning.__

This started out a few weeks ago when I decided to reinstall Windows on my laptop.

The first hurdle was that my 2-3-4 different cloud storage subscriptions were all overdue for a clean-up. The one best suited had me throttled to 1Mbit/s since I was storing 20TB+ of data.

I’ve been taking a lot of photos and videos since I got my first digital camera 22 years ago. Even though I use Lightroom to keep some order there was some duplication and discontinuity. So (after upgrading my fibreoptic internet to 1Gbit, optimizing my home network and cleaning up space on my machine) I started to clean up and organize the various catalogues.

Looking at memories from 20 years ago made me realize I’m better at taking photos than “utilizing” them afterwards. What really is the point, then? I took a picture of one on the screen with my phone and sent on snapchat to my mom, not thinking much about it. But she was really thrilled, which made me really happy as well.

For my current client I’m nearing the end of my contract and for the last two months I’ve mainly been maintaining, documenting and handing over and I really miss building things and solving problems.

Recently a potential client asked about Python and AWS Lambda competence. It’s not something I work with daily and it’s not on my CV. But it made me think about all the various languages and tools I’ve used during the last decade.

My subconscious brain offers up an idea on how to a) build something b) brush up some Python and Lambda knowledge and c) brighten my moms day.

### Concept

Every morning a photo from my archives is e-mailed to my mom, a photo drip.

There is also a gallery where she can look at the previous photo drips.

### Process and goals

The primary objective was to have a working prototype as quickly as possible. So no premature optimization, refactoring or anything.


## Photo selection and preparation

At around 3pm I started browsing photos from year 2000. It took me about 75 minutes to pick out about 300 photos from the first 20.000.

Tip: When browsing in Lightroom, press B to add to Quick Collection.

Export the pictures in the quick collection with a custom filename format like this `0003-2000-05-14.jpg`.

This format ensures filenames are ordered from oldest to newest, and the date can later be extracted directly from the filename without doing any EXIF stuff.

Photos resized to 1500x1500. Never enlarge. Sharpen for screen.

## Photo storage

It would have been quicker to set up a AWS EC2 virtual machine running all the components but that would be too easy.

The requirements for storage is: cheap, reliable, publicly available. So a standard AWS S3 bucket should do just fine.

Using the browser I upload all the photos in batch to a new bucket. In a sub-folder with a random name. That should provide an appropriate level of security and avoid strangers on the internet stumbling upon it. Since the bucket is publicly available.

I verify that I can load the pictures in my browser with the public S3 URL. It took a few attempts at getting the permissions right, and I suspect adding this policy was required even though everything in the settings was set to “Full public access”.

    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Sid": "PublicRead",
                "Effect": "Allow",
                "Principal": "*",
                "Action": [
                    "s3:GetObject",
                    "s3:GetObjectVersion"
                ],
                "Resource": [
                    "arn:aws:s3:::memorydrops/*"
                ]
            }
        ]
    }

## Sending the e-mail

Sending e-mails directly with SMTP is really not an option anymore because of all the spam blocking systems. I know we need to use a third party service or something.

I first looked at AWS SES (simple-email-service). But nothing about it seemed simple. Then looked at a few of the stablished ones such as SendGrid and Mailgun. These tools have evolved a lot and now has a plethora of features aimed at marketing, transactional e-mails etc. Probably overkill and potentially time consuming while having no transferable knowledge or code when hitting a dead-end since they are all different APIs.

What about just using my own Gmail SMTP to ship?

The first tutorials about that required turning on “Less secure app access” for my account.

Not accepting that trade-off, I found https://levelup.gitconnected.com/an-alternative-way-to-send-emails-in-python-5630a7efbe84 where I learned that I can generate an “App password”, similar to an API Key for specific Google applications without disabling other security features. Jackpot!

The article also has working code that I shamelessly used as a starting point.

First I attached the photo of the day, but I really want it embedded. Even though I don’t really like linking images on a public URL (S3 bucket) because of potential browser and client issues that’s what I ended up with. The alternative of base64 encoding the data inline seemed like a chore. Besides I know mom uses Gmail in Chrome anyway so if it worked for me it would probably work for her.

I don’t want the system to be dependent on any state management, databases, etc.

Picking the right photo each day is simply select file N, where N is the days since the first deployment.

Ideally I would use AWS S3 API to list the contents of the bucket to get the available files. But to save some time the “photo index” is simply a text file listing the files:

    ls > filelist.txt

I create a new Lambda function and paste the python script and filelist.txt directly in the Lambda code editor and deploy and test it. Working!

The Lambda function is the configured with a EventBridge trigger with the schedule `cron(0 5 * * ? *)` that should trigger the function at 0500 UTC every day.

## Gallery

I also want to have a gallery where she can view the previous memory drops without having to shuffle through endless emails.

Started out by looking at VueJS for the frontend, which I have used before. But I have not used the new version yet and I suspected it might take a long time getting a project set up from scratch since unfortunately tutorials, tips, documentation in the frontend / JavaScript world has a tendency to be chronically outdated and unreliable.

Dropped that and opted for a very simple native JS photo gallery called [zoomwall.js](https://github.com/ericleong/zoomwall.js/).

Only showing a selection of photos depending on the day however requires a bit more engineering. (Writing this I realize another acceptable approach would be to use native JS to manipulate the DOM directly.)

I implemented this by inlining the CSS and JS into one template HTML file that is rendered on-demand in a Python Lambda function. Then using AWS API Gateway to expose it as a normal webserver.

I wanted to use Jinja2 for templating instead of the built-in Python templating functions. Doing that causes some headache since the Lambda environment does not have Jinja2 installed.

Luckily creating a custom deployment package (.zip) including dependencies is rather trivial.

## Conclusion

All in all this project was started with filtering photos at 3pm, having dinner from 5pm to 6pm and by 8.30pm everything was deployed. The following morning I had the memory drop in my inbox to great delight. An unexpected benefit of using my own Gmail for sending the e-mail is that mom can reply directly.

Expected costs

    S3 offers 5GB of standard storage for free for 12 months. After that I expect the cost to be in the $0.x range per month.

    API Gateway offers 1 Million API calls per month for free for 12 months. After that I expect the cost to be in the $0.x range per month.

    Lambda offers 1 Million requests per month forever.

Performance has been sacrificed for the gallery to make it simple. Server-side rendering is not going to be as fast as client-side. Python is not the fastest alternative. Using a FaaS such as Lambda also introduces penalties and unknowns (cold starts, etc). Yet the gallery HTML loads in ~200ms and and seems instant.

## Improvements

After deciding to share the code I spent an hour or two writing this document as well as some necessary clean-up and changes from the prototype.

The `filelist.txt` is not bundled with the code. It’s hosted in the S3 bucket along with the photos. That means I can update and add photos later without touching code.

That requires the `requests` package, so now both modules are packaged with dependencies and uploaded via AWS CLI instead of browser.

Some hard-coded URLs etc have been converted to environment variables.

Added the `exif` Python package to extract the original time and date of the photo to include in the e-mail.

Added an URL redirect from a prettier domain to the auto generated API Gateway hostname of the gallery. Did not bother with proper custom domain since that requires a lot of fiddling with SSL certificates.

## Bugs and future improvements

- The JS gallery seems slightly broken. Might replace with a better one.
- The e-mail HTML is not pretty.
- Make sender and recipient e-mails environment variables (but recipient is currently a Python list, so).
- Make start date env var. Requires parsing date from user.
- The usual: Check that required env vars are set on startup. Improve error handling and logging.
