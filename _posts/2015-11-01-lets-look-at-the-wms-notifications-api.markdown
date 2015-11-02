---
title: Let's look at the WMS Notifications API
layout: post
tagged: [wms, oclc, notifications]
---

If you're like me, [and I know I am][jatb], you've found yourself spending far
too much time in the [OCLC Web Services documentation][oclc-apis] scheming plans
for all sorts of cool things. You may have even stumbled into the the [Circulation
API][circ-api] page and found yourself drawn to the [Notifications API][notification-api]
area. The docs are pretty sparse and in their absence you're probably thinking
this is some way to circumvent the barebones notification system included in WMS and
roll out your own 21st-century-fancypants emails. Maybe built using a templating
system? With links?? Maybe with a working respond-to email address???

(**Spoiler alert:** it's not that.)

What it appears to be is a way to collect print notices for patrons without an 
email address. 

## Side-note

The majority of [our][lib-home] patrons are Students, Faculty, and Staff, whose
data is pulled from our campus Capstone directory. Essentially, everyone has an
email address associated with their accounts. So our sample pool is basically nil.
To test the API, I gave myself a dummy address and removed my campus email.

## Let's take a look

The docs point you to two endpoints:

`/circ/notification?from={YYYYMMDD}` to get a list of notifications starting at
the specified date. (You can also shorten it to `/circ/notification`, but I can't 
figure out how far back that queries by default.) This one's refered to as the
[**search** route.](#the-search-route)

And `/circ/notification/get?id={fileId}`, which will return the specific notification.
This one's refered to as the [**read** route](the-read-route).

## The search route

So what does a response from this route look like?

```json
{
  "entry": [
    {
      "notificationSearchCriteria": {
        "createDateFrom": 1445313600000,
        "createDateTo": null,
        "notificationType": null,
        "mimeType": null,
        "topic": ""
      },
      "notificationMetaData": [
        {
          "institutionId": "128807",
          "fileId": "CIRCULATION%2FNOTIFICATION_LOAN_OVERDUE%2F003_20151027_0701AM.pdf%3Fid%3DXXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX",
          "createDate": 1445425278172,
          "mimeType": "application/pdf",
          "fileName": "003_20151027_0701AM.pdf",
          "topic": "NOTIFICATION_LOAN_OVERDUE",
          "description": "Your library materials are now overdue"
        }
      ]
    }
  ]
  //...
}
```

> **Note:** I'm excluding the metadata that's included at the root level of the 
> response. Fields like `startIndex`, `totalResults`, `title`, and `itemsPerPage`
> that are included in most of OCLC's API responses.

I wasn't able to come up with parameters to append to the url that would affect the
`notificationSearchCriteria` section, but the `notificationMetaData` is what we're
really looking for. These are our notifications that weren't able to be emailed out.
In order to actually retrieve them we'll have to utilize the other api route...

## The read route

To access a specific notification record, we'll have to pass the `fileId` we found
in the above response. You probably noticed the `"mimeType": "application/pdf"`
entry as well as the `.pdf` extension of the `fileName`. What we're getting back
isn't a JSON or XML response with populated fields, but actually a pre-generated
pdf notice that's stuffed with the language provided in the WMS config. And _actually_
it's abstracted one more degree. The reponse from `/circ/notification/get?id=<fileId>`:

```json
{
  "entry": [
    {
      "metaData": {
        "institutionId": "128807",
        "fileId": "CIRCULATION%2FNOTIFICATION_LOAN_OVERDUE%2F003_20151027_0701AM.pdf%3Fid%3DXXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX",
        "createDate": null,
        "mimeType": "application/pdf",
        "fileName": "003_20151027_0701AM.pdf",
        "topic": null,
        "description": null
      },
      "content": "VBERi0xLjQKJeLjz9MKMyAwIG9iago8PC9MZW5ndGggNTE5L0ZpbHRlci9GbGF0ZU..."
    }
  ],
  // more meta data...
}
```

Yup, the `entry[0].content` is the pdf data base64 encoded. So to get the file, 
we'll have to do something like:

```javascript
var https = require('https')
var join = require('path').join
var WSKey = require('oclc-wskey')
var key = new WSKey('wskey-public', 'wskey-secret')
var fileId = 'CIRCULATION%2FNOTIFICATION_LOAN_OVERDUE%2F003_20151027_0701AM.pdf%3Fid%3DXXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX'
var hostname = 'circ.sd00.worldcat.org'
var path = '/circ/notification/get?id=' + fileId
var opts = {
  hostname: hostname,
  path: path,
  method: 'GET',
  port: 443,
  headers: {
    'Accept': 'application/json', // need to specify, otherwise we'll get XML
    'Authorization': key.HMACSignature('GET', 'https://' + hostname + path)
  }
}
var outDir = join(__dirname, 'pdfs')

https.request(opts, function (res) {
  var body = ''
  res.setEncoding('utf-8')
  res.on('data', function (d) {body += d})

  res.on('end', function () {
    var parsed = JSON.parse(body)
    if (parsed.entry) {
      return writePdf(parsed.entry[0].metaData.fileName, parsed.entry[0].content)
    }
  })
})

function writePdf (filename, b64data) {
  var buf = new Buffer(b64data, 'base64')
  require('fs').writeFileSync(join(outDir, filename), buf)
}
```

And you'll get something like this:

![overdue-item-notification][redacted-notice]

## So now what?

I don't really know. Maybe use [Electron][atom-electron] to build a desktop app 
that pulls and prints all of your new notifications each morning? Maybe a web app
to download and store the pdfs? I believe there's already a WMS way of handling
print notifications (although I can't find any documentation on it.)

In an ideal world, it'd be rad to be able to nice-ify the automated email notices
that are sent out, especially in a modern HTML-y way. From what I've heard, creating
them currently requires several sacrifices to a not-so-cool deity. Some links (to access
your account, especially) would be welcome, as the ability to provide a `From:` 
address is lacking. But like, you could also use a service like [twilio][twilio] to
send text notifications because who reads emails anyway?

Maybe something's coming down the pipe. I'll keep my fingers crossed.

## Notes

1. The script I used for testing made a request for the list of notifications and
   a request for each item listed. [Here's a link to the gist][test-gist].
2. Because the patron's address isn't provided within the API response, my only 
   guess as to how to mail the notice out is to use an envelope with a front-window?


[jatb]:             https://www.youtube.com/watch?v=J13kfIHyltQ
[oclc-apis]:        http://www.oclc.org/developer/develop/web-services.en.html
[circ-api]:         http://www.oclc.org/developer/develop/web-services/wms-circulation-api.en.html
[notification-api]: http://www.oclc.org/developer/develop/web-services/wms-circulation-api/notification-resource.en.html
[lib-home]:         http://trexler.muhlenberg.edu
[redacted-notice]:  {{ site.baseurl }}/pictures/redacted-wms-notification.png
[atom-electron]:    http://electron.atom.io
[test-gist]:        https://gist.github.com/malantonio/e663716f1d9159fc1589
[twilio]:           https://www.twilio.com/