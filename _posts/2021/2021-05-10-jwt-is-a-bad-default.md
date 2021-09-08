---
title: "JWT should not be your default for sessions"
date: "2021-05-10 20:37:00 UTC"
tags: jwt, security
geo: [43.686510, -79.328419]
location: "East York, ON, Canada"
---

Cookies
-------

When designing web applications, (especially the traditional HTML kind),
you will at one point have to figure out how to log a user in and keep them
logged in between requests.

The core mechanism we use for this are cookies. Cookies are small strings sent
by a server to a client. After a client receives this string, it will repeat
this in subsequent requests. We could store a 'user id' in a cookie, and
for any future requests we'll know what user_id the client was.

```
Cookie: USER_ID=123
```

But this is very insecure. The information lives in the browser, which means
users can change `USER_ID` and be identified as a different user.


Sessions
--------

The traditional way to solve this is what's known as a 'session'.
I don't know what the earliest usage of sessions is, but it's in every web
framework, and has been since web frameworks are a thing.

Often, sessions and cookies are described as 2 different things, but
they're really not. A session needs a cookie to work.

```
Cookie: MY_SESSION_ID=WW91IGdvdCBtZS4gRE0gbWUgb24gdHdpdHRlciBmb3IgYSBmcmVlIGNvb2tpZQ
```

Instead of a predictable user id, we're sending the client a completely random
session id that is impossibly hard to guess. The ID has no further meaning, and
doesn't decode to anything. This is sometimes called an opaque token.

When a client repeats this session id back to the server, the server will look
up the id in (for example) a database, which links it back to the user id.
When a user wants to log out, the session id is removed from the data storage,
which means the cookie is no longer associated with a user.

### Where is the session data stored?

Languages like PHP have a storage system for this built in, and will by default
store data in the local filesystem. In the Node.js ecosystem, by
default this data will be in 'memory' and disappear after the server restarts.

These approaches make sense on developer machines, or when sites were hosted
on long-lived bare-metal servers, but these days a deploy typically means
a completely fresh 'system', so this information needs to be stored in a place
that outlives the server. An easy choice is a database, but it's common for
sites to use systems like Redis and Memcached, which works for tiny sites, but
still works at massive scales.


Encrypted token
---------------

Over 10 years ago, I started working a bit more with OAuth v1 and similar
authentication systems, and I wondered if we could just store all the
information in the cookie and cryptographically sign it:

<figure>
  <a href="https://stackoverflow.com/questions/3240246/signed-session-cookies-a-good-idea">
    <img src="/assets/posts/jwt/stackoverflow.png" alt="Question about session tokens on Stack Overflow" style="max-width: 100%"/>
  </a>
</figure>

Despite getting some good answers, I didn't go through with it as I didn't
feel confident enough in making this secure, and I felt it required a better
understanding in crypto than I did.

A few years later, we got [JWT][8], and it's hot shit! JWT itself is a standard for
encrypting/signing JSON objects and it's used a LOT for authentication.
Instead of an opaque token in a cookie, we actually embed the `user_id`  again,
but we include a signature. The signature can only be generated by the server,
and it's calculated using a 'secret' and the actual data in the cookie.

This means that if the data is tampered with (the `user_id` was changed), the
signature no longer matches.

So why is this useful? The best answer I have for this, is that it's not
needed to have a system for session data, like Redis or a database. All the
information is contained in the JWT, it means your infrastructure is in
theory simpler. You're potentially making fewer calls to a data-store on
a per-request basis.


Drawbacks
---------

There are major drawbacks to using JWT.

First, it's a complicated standard and users are prone to get the settings
wrong. If the settings are wrong, in the worst case it could mean that *anyone*
can generate valid JWTs and impersonate anyone else. This is not a
beginners-level problem either, last year [Auth0][2] had a bug in one of
their products that had this very problem.

Auth0 is(or was? they just got acquired) a major vendor for security products,
and ironically sponsor the [jwt.io][3] website. If they're not safe, what
chance does the general (developer) public have?

However, this issue is part of a larger reason why many security experts
[dislike][4] JWT: it has a ton of features and a very large scope, which
gives it a large surface area for potential mistakes, by either library
authors or users of those libraries. (alternative stateless tokens to JWT
[exists][7], and some of them do solve this.)

A second issue is 'logging out'. With traditional sessions, you can just
remove the session token from your session storage, which is effectively
enough to 'invalidate' the session.

With JWT and other stateless token this is not possible. We can't remove
the token, because it's self-contained and there's no central authority that
can invalidate them.

This is typically solved in three ways:

1. The tokens are made very short lived. For example, 5 minutes. Before the
   5 minutes are over, we generate a new one. (often using a separate refresh
   token).
2. Maintain a system that has a list of recently expired tokens.
3. There is no server-driven log out, and the assumption is that the client
   can delete their own tokens.

Good systems will typically use the first two. An important thing to point out
is, in order to support logout, you'll likely still need a centralized storage
mechanism (for refresh tokens, revocation lists or both), which is the very
thing that JWT were supposed to 'solve'.

> Sidenote: some people like JWT because it's fewer systems to hit per request,
> but that contradicts with being able to revoke tokens before they expire.
>
> My favourite solution to this is keep a global list of JWTs that have been
> revoked before they expired (and remove the tokens after expiry). Instead
> of letting webservers hit a server to get this list, push the list to each
> server using a pub/sub mechanism.
>
> Revoking tokens is important for security, but rare. Realistically this
> list is small and easily fits into memory. This largely solves the
> logout issue.

A last issue with JWT is that they are relatively big, and when used in
cookies it adds a lot of per-request overhead.

All in all, that's a lot of drawbacks just to avoid a central session
store. It's not my opinion that JWT are universally a bad idea or without
benefits, but there is a lot to consider.

Why are they popular?
---------------------

One thing that's surprised me when reading tech blogs, is that there
is a _lot_ of chatter around JWT. Especially on Medium and subreddits like
[/r/node][5] I see intros to JWT on an extremely regular basis.

I realize that that doesn't mean that 'JWTs are more popular than
session tokens', for the same reason that GraphQL isn't more popular than
REST, or NoSQL than relational databases: it's just not that interesting
to write about the technology that's been tried and tested for a decade or
more (See: [Appeal to novelty][9]). In addition, subject experts that write
about new solutions are likely going to have different problems & scale than
the majority of their own readers.

However, these new technologies create a lot more buzz than their simpler
counterparts, and if enough people keep talking about the hot thing,
eventually this can translate to actual adoption, despite it being a
sub-optimal choice for the majority of simple use-cases.

This is similar to many newer developers learning how to build SPAs with
React before server-rendered HTML. Experienced devs would likely feel that
server-rendered HTML should probably be your default choice, and building
an SPA *when needed*, but this is not what new developers are typically
taught.

Adopting complex systems before the simple option is considered is
something I see happening more, but it surprised me for JWT.

As an  exercise, I looked up the top most popular (by votes) posts
on /r/node that mention JWT. I was going to go over the first 100, but got
bored after the top 12.

From these 12 articles and Github repos:

* **1** mentions using a revocation list, **3** mention refesh tokens.
  The remaining articles and github repositories simply have no means of
  logging out.
* **1** article mentions that it _might_ be better to use a standard session
  storage instead.
* **1** article uses _both_ a standard session storage and JWT, making JWT
  unneeded.
* **1** github repository ships with pre-generated private keys. (yup)
* Most of posts use expiry times of weeks or months and 3 posts **never**
  expire their JWTs.

Except 1, the quality of these highly upvoted posts was extremely low, the
authors were likely not qualified to write about this and can potentially
cause real-world harm.

All this at least confirmed my bias that JWT for security tokens is hard to
get right.

On JWT and scale
----------------

Going through tons of Reddit posts and comments also got me a more refined
idea of why people think JWTs are better. The top reason everywhere is:
"It's more scalable", but it's not obvious at what scale people *think*
they'll start to have issues. I believe the point where issues start to
appear is likely much higher than is assumed.

Most of us aren't Facebook, but even at 'millions of active sessions', a
distributed key->value store is unlikely going to buckle.

Statistically, most of us are building applications that won't make a
Raspberry Pi break a sweat.


Conclusion
----------

Using JWTs for tokens add some neat properties and make it possible in some
cases for your services to be stateless, which can be desirable property in
some architectures.

Adopting them comes with drawbacks. You either forego revocation, or you
need to have infrastructure in place that be way more complex
than simply adopting a session store and opaque tokens.

My point in all this is not to discourage the use of JWT in general, but
be deliberate and careful when you do. Be aware of both the security and
functionality trade-offs and pitfalls. Keep it out of your 'boilerplates'
and templates, and don't make it the default choice.


Acknowledgements
----------------

Thanks to [Nick Chang-Fong][10] and [Dominik Zogg][6] for providing feedback and
suggestions to this article!


[1]: https://crypto.stackexchange.com/questions/9336/is-hmac-md5-considered-secure-for-authenticating-encrypted-data
[2]: https://insomniasec.com/blog/auth0-jwt-validation-bypass
[3]: https://jwt.io/
[4]: https://paragonie.com/blog/2017/03/jwt-json-web-tokens-is-bad-standard-that-everyone-should-avoid
[5]: https://www.reddit.com/r/node/
[6]: https://twitter.com/dominikzogg
[7]: https://www.scottbrady91.com/JOSE/Alternatives-to-JWTs
[8]: https://jwt.io/ "JSON Web Tokens"
[9]: https://en.wikipedia.org/wiki/Appeal_to_novelty
[10]: https://twitter.com/nchangfong?lang=en