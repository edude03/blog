I once worked at a med tech startup that integrated with Electronic Medical Record systems. One of our first big customers was a Java shop, and we - being hip and cool in the ~2010s were a ruby shop (ruby 1.8 and some version of rails). When we had our meeting about how our systems should integrate I proposed REST/json - they said they could do SOAP.

They said doing json would be nearly impossible for them - not elaborating on why - then one meeting they said "sure, REST/json is great for us" - so we started building our integration.

Back in these days, I/we were really into TDD, so we asked early on for samples of the payloads we could expect, we loaded them into VCR[^2] and we developed our integration in our own little worlds[^0].

Close to launch day, we deployed our service, they deployed theirs we sent some messages and our app would just crash. Not fail with an error, but segfault and exit the process - which is really un-rails like, and since the rails server was crashing we couldn't really do anything from within the code to see what they were sending us.

As we were both in the Toronto area we went over to their office to do a little onsite integration testing and long story short they told us what to this day is the most horrifying tech story I've been involved in.

As the legend went - this company had forked an open source framework (spring?) into their own internal framework and it had diverged so much that their current code wouldn't run without rewriting it. Whatever framework it was did not support JSON (de)serialization at the time they forked it so therefore they didn't support JSON. "Why not just grab some java json library and integrate it?" - yeah so, it turns out their open source policy is that before adding an open source dependency to the framework, you needed your managers approval, their managers approval, the VP of Engineerings' approval, the CTOs approval, LEGALS approval then if you got all that - your change would go into testing, then eventually into the next release of the internal framework - which was every quarter. So end to end ~6 months.

The time between us deciding to integrate and us being in their office was only like 3/4/5 MAYBE 6 weeks, much less than the 6 months it would take to get a json library in, so what did they do? Well, spring (or whatever) is a MVC web framework, and views generate text right? And json is just text right? (I'm sure you can see where this is going)

Yes, every team we needed to integrate with WROTE THEIR OWN JSON SERIALIZER IN $FRAMEWORKS's TEMPLATING ENGINE. literally just

```
// made up syntax, I don't remember what templating engine it was
<% for k,v in query_result >
{ "<%k%>": "<%v%>" }
%>

<% for user in user_query %>
{ "username": <% user.username.to_string() %> }
<% end %>

```

which is not just appalling but since every team wrote their own (and wrote their own per view as far as we could tell) the json we got was always crazy variations of `{ "username":, }` `{"username": "", }` `{"username": null} // or "null" or "Java blahblahblah exception ` - which was killing our poor json parser which was really strict (and fast, written in C and hand rolled assembly) and thus was blowing up when fed... well not JSON.

Now, that's the serializer, what about _de_ serializing since I said it went both ways yet they had neither right? If you guessed they wrote their own deserializers in _REGEX_ - you either worked with me and/or also worked with a bunch of ~~sick fuc-~~ ingenious peeps who could do a lot with a little.

The number of times I went into the office and phone started ringing promptly at 08:00:01 (am) with $TEAM mad at us because a (to us) really minor change to the API broke their integration probably legitimately gave me PTSD.

This did teach me one really cool trick however. In rails, you can import a controller into another controller and just add/override things in the new controller. So when we needed to make any change at all, we would simply create a API_V$(N+1) folder, with the new controller and implement the changes in there - to date its still the best backward compatibility strategy I've ever used, so thanks DHH.

As for what happened after, we launched the service it worked. I tried my hand at writing a json parser in C but I was too dumb (but seriously its hard with all the edge cases) and we settled on ruby's native slow af[^1] - but resilient - parser and everyone lived happily ever after.

[^0]: Obvious foreshadowing
[^1]: Fast was important because also the payloads they sent could be 100s of MBs to single digit GBs. I don't know how either but that's probably a topic for another war story episode
[^2]: For the younggins in the audience - VCR is/was this awesome framework that automatically generated mocks from real requests (or samples of the real request) combined with RSpec it made it made it super easy to write RPC code since you wrote it like normal and the request would be hijacked automatically so you didn't need to change the code - you could keep calling :get('google.com') or whatever and it would just work
