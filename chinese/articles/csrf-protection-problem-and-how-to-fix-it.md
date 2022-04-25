> -  原文地址：[CSRF Protection Problem and How to Fix it](https://www.freecodecamp.org/news/csrf-protection-problem-and-how-to-fix-it/)
> -  原文作者：[Jakub T. Jankiewicz](https://www.freecodecamp.org/news/author/jcubic/)
> -  译者：
> -  校对者：

![CSRF Protection Problem and How to Fix it](https://www.freecodecamp.org/news/content/images/size/w2000/2022/03/laptop-security-virus-protection-internet-malware-1588329-pxhere.com.jpg)

One day I was working on a feature at work. I had many branches created in JIRA tickets, so I wanted to open a bunch of PRs (Pull Requests) all at once in different tabs.

This is how I usually work – I have a lot of tabs open and this speeds things up, because I don't need to wait for the next page to load.

But after I'd created the first PR in BitBucket and tried to go on to the next page, I was welcomed with an error message about an invalid CSRF token. This is a common problem with web applications that have CSRF protection.

So in this article you'll learn what CSRF is and how to fix this error.

## Table of contents:

-   [What is CSRF?](#what-is-csrf)
-   [Standard CSRF protection](#standard-csrf-protection)
-   [The Problem with Tokens](#the-problem-with-tokens)
-   [Cross-tab Communication Solution](#cross-tab-communication-solution)
    -   [Sysend library](#sysend-library)
    -   [Broadcast Channel](#broadcast-channel)
-   [Conclusion](#conclusion)

## What is CSRF?

CSRF is an acronym for **Cross-Site Request Forgery**. It is a vector of attack that attackers commonly use to get into your system.

The way you usually protect against CSRF is to send a unique token generated by each HTTP request. If the token that is on the server doesn't match with the one from the request, you show an error to the user.

## Standard CSRF protection

This is one way you can protect against CSRF with a token:

```javascript
const inital_token = '...';

const secure_fetch = (token => {
    const CSRF_HEADER = 'X-CSRF-TOKEN';
    return (url) => {
        const response = await fetch(url, {
            method: 'POST',
            headers: {
              [CSRF_HEADER]: token
            }
        });
        response.then(res => {
           token = res.headers[CSRF_HEADER]
        });
        return response;
    };
})(inital_token);
```

This code uses the [fetch API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API) to send and receive a secure token in [HTTP](https://en.wikipedia.org/wiki/Hypertext_Transfer_Protocol) headers. On the backed, you should generate the first initial token when the page loads. On the server, on each [AJAX](https://en.wikipedia.org/wiki/Ajax_(programming)) request, you should check to see if the token is valid.

## The Problem with Tokens

This works fine unless you have more than one tab open. Each tab can send requests to the server, which will break this solution. And power users may not be able to use your application the way they want.

But there is a simple solution to this problem which is cross-tab communication.

## Cross-tab Communication Solution

### Sysend library

You can use the [Sysend library](https://github.com/jcubic/sysend.js), an open source solution that I've created specifically for this purpose. It simplifies cross-tabs communication.

If you want, you can use a native browser API like [Broadcast Channel](https://developer.mozilla.org/en-US/docs/Web/API/BroadcastChannel) to do the same. More on how to do that later in this article.

But the **Sysend** library will work for browsers that don't support Broadcast Channel. It also works in IE (it has some bugs, which is not a surprise). You may also need to support some old mobile browsers. It also has a much simpler API.

This is the simplest example:

```javascript
let token;
sysend.on('token', new_token => {
    token = new_token;
});

// ...

sysend.broadcast('token', token);
```

Simple example of using base function of sysend library

And this is how you would use this library to fix CSRF protection:

```javascript
const inital_token = '...';

const secure_fetch = (token => {
    const CSRF_HEADER = 'X-CSRF-TOKEN';
    const EVENT_NAME = 'csrf';
    sysend.on(EVENT_NAME, new_token => {
        // get new toke from different tab
        token = new_token;
    });
    return (url) => {
        const response = await fetch(url, {
            method: 'POST',
            headers: {
              [CSRF_HEADER]: token
            }
        });
        response.then(res => {
           token = res.headers[CSRF_HEADER];
           // send new toke to other tabs
           sysend.broadcast(EVENT_NAME, token); 
        });
        return response;
    };
})(inital_token);
```

secure\_fetch function with CSRF protection using sysend

All you have to do is to send and receive a single message from other tabs when sending the request. And your CSRF protected app will work on many tabs.

And that's it. This will let advanced users use your app that has CSRF protection when they want to open many tabs.

### Broadcast Channel

Here is the simplest possible example of using Broadcast Channel:

```javascript
const channel = new BroadcastChannel('my-connection');
channel.addEventListener('message', (e) => {
    console.log(e.data); // 'some message'
});
channel.postMessage('some message');
```

Basic usage of Broadcast Channel

So with this simple API you can do the same thing that we did before:

```javascript
const inital_token = '...';

const secure_fetch = (token => {
    const CSRF_HEADER = 'X-CSRF-TOKEN';
    const channel = new BroadcastChannel('csrf-protection');
    channel.addEventListener('message', (e) => {
        // get new toke from different tab
    	token = e.data;
    });
    return (url) => {
        const response = await fetch(url, {
            method: 'POST',
            headers: {
              [CSRF_HEADER]: token
            }
        });
        response.then(res => {
           token = res.headers[CSRF_HEADER];
           // send new token to other tabs
           channel.postMessage(token);
        });
        return response;
    };
})(inital_token);
```

secure\_fetch function with CSRF protection using BroadcastChannel

As you can see from the above example, Broadcast Channel doesn't have any namespace for events. So if you want to send more than one type of event you need to create types of events.

Here is an example of using Broadcast Channel to do more than the CSRF protection fix we've discussed so far.

You can synchronize login and logout for your application. If you login into one tab, your other tabs will also sign you in. In the same way, you can synchronize the shopping cart in some e-commerce websites.

```javascript
const channel = new BroadcastChannel('my-connection');
const CSRF = 'app/csrf';
const LOGIN = 'app/login';
const LOGOUT = 'app/logout';
let token;
channel.addEventListener('message', (e) => {
    switch (e.data.type) {
        case CSRF:
            token = e.data.payload;
            break;
        case LOGIN:
            const { user } = e.data.payload;
            autologin(user);
            break;
        case LOGOUT:
            logout();
            break;
    }
});

channel.postMessage({type: 'login', payload: { user } });
```

Using Broadcast Channel with different type of messages

## Conclusion

It's great if you protect your app against attackers. But keep in mind how people will be using your application, too so you don't make it unnecessarily hard to use. This applies not only to this particular problem.

The **Sysend** library is a simple way to communicate between open tabs in the same browser. And it can fix major issues with CSRF protection. The library has more features, and you can check its [GitHub repo](https://github.com/jcubic/sysend.js) for more details.

**Broadcast Channel** is also not that complicated. If you don't need to support old browsers or some older mobile devices, you can use this API. But if you need to support older browsers, or want to make your code simpler, you use can the sysend library.

If you want to see browser support for Broadcast Channel, you can see [Can I Use](https://caniuse.com/broadcastchannel).