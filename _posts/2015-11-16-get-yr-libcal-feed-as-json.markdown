---
layout: post
title: Get your LibCal feed(s) as JSON
tagged: [libcal, json, calendars]
---

[LibCal][libcal] is [Springshare's][springshare] event/room booking service that's
part of their **LibApp**s package. If you're paying for [LibGuides][libguides] or
any of their other services, I believe you're automatically set up to use it in a
limited capacity. Maybe give it a shot if it's already there?

Anyway, they provide the ability to: embed your calendar(s) as widgets, publish
an RSS feed of events, or subscribe to a calendar with an `.ics` file. Notice
anything missing? No [JSON][json-voorhees]. 

(also there's no way I can find to merge calendars, but bear with me for a second)

"But wait a minute", I can hear you say, "there's gotta be _some_ way of getting
`json` since that's what the widgets are probably using."

**The good news:** there is, they are.

**The bad news:** you gotta go digging for it.

## tldr: how I dug for it

I was gonna write this whole process out. But that's super boring. Here's how you
do it (using Firefox/Chrome dev tools):

* Open the Calendar view/edit screen
* From the **Settings** drop down in the upper right corner, select 
  **Embed/Export Options**
* Copy the url value from the Full Calendar `<iframe>`. Visit this page in another tab.
  * It'll look something like: `api3.libcal.com/embed_calendar.php?iid=<number>&cal_id=<number>&w=800&h=600`
  * Remove the `//` before `api3`
* Open DevTools in your browser, find a **Network** tab, and refresh the page. 
  This will display all of the connections your browser is making for the widget's
  various parts.
* Copy the full url that points to `process_cal.php`. This is what we're looking for!

Here's the url scheme:

```
http://api3.libcal.com/process_cal.php?c=<calendar id>&iid=<institution id>&start=<YYYY-MM-DD>&end=<YYYY-MM-DD>&sp=<0 or 1>
```

### Some deets

#### `c`

This is the specific calendar ID. There's no way that I can find to request multiple
calendars. To do that, you'll have to make a request for each calendar and concatenate
the results.

#### `iid`

From what I can gather, this is an ID specific to your institution. Makes sense 
to me. Also, excluding it sends you to the LibCal main page.

#### `start` and `end`

Date ranges to grab, formatted `YYYY-MM-DD`. For `start` to work, however, you're
gonna have to use the `sp` key/val.

#### `sp`

My best guess is that this stands for something like "Show Past/Previous". Setting this
to `0` excludes events prior to today; `1` starts at the `start` date.

#### `_`

The [jQuery FullCalendar plugin][jq-cal] being used appends this prevent the browser
from caching the response so each request has fresh data. If you're planning on
using the JSON server side, you can probably safely leave this off.

### The results

So, the info sent back is specific to the widgets, since we get back properties 
like `backgroundColor`, `borderColor`, and `className`. However, we do get some 
useful event info: 

  * `title`
  * `start` and `end` timestamps
  * `short_desc`
  * `location`
  * `campus`
  * `pres` (presenter)
  * `categories`

Here's a full example:

```json
{
  "id": "2239909",
  "title": "SPN 101 - Motorcycle Diaries",
  "start": "2015-11-17T20:30:00",
  "end": "2015-11-17T22:45:00",
  "url": "http://muhlenberg.beta.libcal.com/event/2239909",
  "allDay": false,
  "backgroundColor": "#B3C8EF",
  "borderColor": "#8fa0bf",
  "textColor": "#222",
  "short_desc": "",
  "location": "B-02",
  "campus": "",
  "seats": "",
  "pres": "",
  "className": [],
  "categories": ""
}
```

## Epilogue: make yr own feed

<aside>
<strong>Note:</strong> this will work for all calendars, but if you want to include 
branch/building hours, you'll have to do a bit of fiddling — the url for the those is:
<code>https://api3.libcal.com/api_hours_today.php</code>
</aside>

I'm currently toying with [a way to module-ize this][libcal-github], but essentially,
this is how you'd do it:

```javascript
var https = require('https')
var calIds = [2633, 2636]
var iid = 814
var all = []

;(function getItems () {
  var id
  if (id = calIds.shift()) {
    https.get(getUrl(id, iid, '2015-09-01', '2015-12-18'), function (res) {
      var body = ''
      res.setEncoding('utf8')
      res.on('data', function (d) { body += d })
      res.on('end', function () {
        var parsed = JSON.parse(body)
        all = all.concat(parsed)

        return getItems()
      })
    })
  } else {
    console.log(all)
  }
})()

function getUrl(calId, iid, startDate, endDate) {
  var url = 'http://api3.libcal.com/process_cal.php?c=%d&sp=%d&iid=%d&start=%s&end=%s'
  
  return require('util').format(url, 
          calId,     // c=
          1,         // sp=
          iid,       // iid=
          startDate, // start=
          endDate    // end=
        )
}
```

[libcal]:        http://www.springshare.com/libcal/
[springshare]:   http://www.springshare.com/
[libguides]:     http://www.springshare.com/libguides/
[json-voorhees]: http://www.json.org/
[jq-cal]:        http://arshaw.com/fullcalendar/
[libcal-github]: https://github.com/malantonio/libcal