---
layout: post
title: >
  Let people invite themselves to Google Calendar entries using AppScript
feature_image: "/images/calendar-with-blue-denim-jeans.jpeg"
---

If you want to organise an event with a group of people within your Google Workspace, you can invite the whole workspace or ask around who wants to attend.
It has been the norm at my current workplace to post in Slack and let people react with an emoji if they wish to attend. This was convenient as any attendee only needed to click once. 

![Calendar invite via ticket emoji](/images/calendar-invite-ticket.png)

Sadly, this approach burdens the speaker with tracking who has reacted and manually adding everyone to the invite.
Beyond a particular scale, it also has the disadvantage that Slack only lists 50 people per emoji reaction.
Thus, for more significant events, you need some workaround.
One approach was to have a central calendar where everyone could add themselves to the invite.
This sadly adds an overhead to joining an event that you, as the interested speaker, would like to avoid.

To overcome this overhead, I looked into Google Apps Script.
While it is mainly used to add automation to Google Documents, you can create standalone links to run a script with a click.

# Getting the necessary information.

To get started with Apps Script, you can open the developer console at [script.google.com](https://script.google.com/).
App Script is using JavaScript as its underlying language.
Thus, fundamental JavaScript skills are needed to write functions.
For the sign-up to work, we only need to write a few lines of code.
Therefore, it is sufficient to use a single code file; we don't need to dive into how libraries work.

As our primary goal is to add someone to an event, we want to execute the following code:
```javascript
CalendarApp.getCalendarById(…).getEventById(…).addGuest(…)
```

For this to work, we need to have the calendar ID, the event ID, and the attendee's mail address.

To get the calendar ID, one needs to open the Google Calendar web interface, click on the three dots next to the calendar (you need to hover over it in the left column), and select `Settings & Sharing`.
Here, you can scroll down the page until you read `Integrate Calendar`.
There, you can find the calendar ID directly.
If it is your calendar, it is usually the mail address associated with the Google account.
Otherwise, it probably ends in `@group.calendar.google.com` for shared calendars.

Sadly, it is not straightforward to get the event ID.
For this, we need to write some Apps Script code.
With the above calendar ID, we can list all events in a specific time frame using the code below.
You can run this code directly in the editor by selecting the `listEvents` function in the top bar and clicking on `Run`. 

```javascript
function listEvents() {
  let startDate = new Date("2024-05-22");
  let endDate = new Date("2024-05-23");
  let calendarId = "…@group.calendar.google.com";
  let calendar = CalendarApp.getCalendarById(calendarId);
  if (calendar === null) {
    // Calendar not found
    console.log('Calendar not found', calendarId);
    return;
  }

  // Get the events within the specified timeframe
  let calEvents = calendar.getEvents(startDate,endDate);
  console.log(calEvents.length); // Checks how many events are found
  // Loop through all events
  for (var i = 0; i < calEvents.length; i++) {
    let event = calEvents[i];
    console.log(event.getTitle() + " " + event.getId());
  }
}
```

For the attendee's mail address, we assume that they want to sign up with the Google account they are currently logged in.
As we expect this to run in the context of a corporate Google workspace, it will automatically select the address from the workspace if you are logged in with multiple accounts.
We can get this information by calling the function `Session.getActiveUser().getEmail()`.

# The sign-up link

While the most common usage of an Apps Script is inside an app, you can also deploy them as a standalone URL endpoint.
For that, you will need to provide a `doGet(e)` function.
The request parameters are passed in as the `e` argument.
As we don't make use of them, we can ignore them here.
At the end of the function, you need to provide some content that is displayed to the user.
For simplicity, we are going to show them a small JSON response.

Similarly to the above code, we are using get the relevant calendar and then the relevant event using `getEventById`.
If someone opens a link to this endpoint, Google will ensure that they are logged in to the right workspace.
Because of that, you can get their mail address using `Session.getActiveUser().getEmail()`.
We use that address to invite them to the event then.

```javascript
function doGet(e) {
  // This is listed in the "Settings & Sharing" section
  let calendarId = "…@group.calendar.google.com";
  // Use the listEvents function to find the right ID
  let eventId = "…@google.com";
  let calendar = CalendarApp.getCalendarById(calendarId);
  if (calendar === null) {
    console.log('Calendar not found', calendarId);
    return;
  }

  let event = calendar.getEventById(eventId);
  if (event === null) {
    console.log('Event not found', eventId);
    return;
  }

  event.addGuest(Session.getActiveUser().getEmail());
  return ContentService.createTextOutput('{"ok": "Successfully invited you to the event."}').setMimeType(ContentService.MimeType.JSON);
}
```

We can now use the above code to provide an endpoint that everyone can open.
To do so, we need to create a new deployment using the `Deploy -> New Deployment` dialogue.
In this dialogue, we set a matching description.
I usually use the event I invite people to.
We then need to select that the app executes as `Me` (i.e. your user).
If you choose that the App should run as the caller, your users will be presented with three dialogues that request complete control over their calendars.
With the setting as `Me`, they only get the response once they have been added to the event.
This means, though, that you should be very careful what you do in the script, as otherwise, someone could also use this to hijack your account. 

Finally, select that everyone in your workspace can access this app and click on `Deploy`. You will now be provided with a link that you can share with everyone in, e.g. Slack to invite themselves.

![Apps Script Deployment Dialogue](/images/app-script-deploy.png)

*Title picture: Photo by [Gaining Visuals](https://unsplash.com/photos/person-in-blue-denim-jeans-wKu5yvAT0bg?utm_content=creditCopyText&utm_medium=referral&utm_source=unsplash) on [Unsplash](https://unsplash.com/?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)*

