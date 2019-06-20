---
layout: post
title:  "Dismissing Bootstrap Alerts in Rails"
date:   2019-06-20 10:00:00 -0500
---

I tend to lean on Bootstrap for the default look and feel of my various side projects. Over the years, I have established a small handful of patterns and helpers that I automatically copy from one project to the next that make using Bootstrap with Rails more enjoyable. On of my favorites patterns revolves around notification alerts.

I fix Bootstrap alerts to the bottom of the page to provide feedback on the success or failure of important actions, like logging in or updating a record.

```ruby
# app/views/layouts/application.html.haml
...
.fixed-bottom.zindex-popover
  - if notice = flash[:notice]
    .alert.alert-primary.fade.show{role: 'alert'}
      = notice
      %button.close{'type' => 'button', 'data-dismiss' => 'alert'}
        %span &times;
...
```

It may be a personal bias, but I like to keep these notifications around until the user asks them to go away, rather than fading them out after a time interval. I want users to have an opportunity to read everything. On the other hand, I don't want to force them to manually close every alert. The best of both worlds seems to be dismissing alerts when the user continues interacting with the application. Since the messages come from the flash object, navigation to a new page already clears them. Scrolling is another simple case, and can be handled with a bit of jQuery.

Opening a modal is a little less obvious. Because I'm displaying alerts over the page, they don't blur into the background with everything else. Fortunately, Bootstrap issues a `show.bs.modal` event that can be handled just as easily as scroll.

```js
// app/assets/javascripts/alert.js
$(document).on('scroll show.bs.modal', function () {
    $('.alert').alert('close');
});
```

With this approach, alerts are extremely predictable. They hang around until the user triggers their removal, either by directly clicking a close button, or by naturally continuing to interact with the application.
