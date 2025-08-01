# Musings from my experience writing a toy kafka implementation

Recently I finally found the time to make use of my Code Crafters subscription and I figured since I've used Kafka/Redpanda in production for years it would be a good time to look under the covers and see how it works - while also writing some rust as I've become ... rusty after not writing anything substantial in the past two years. And let me tell you it was ~~hard as fu-~~ a really fun challenge.

For those not familiar, kafka is a "distributed log" - you send it messages, it stores them, and you can get them back later in the order you sent them. Kind of like a queue, except it has some guaranties around "atleast once delievery" and "durability" which makes it really great if you need to store some immutable facts and process them at a later time.

## A little about code crafters

Code crafters is an online subscription service that allows you to build a tool that exists from scratch. It's guided, but they're also fairly hands off. The goals are things like "bind to a port" or "decode a Fetch request" with a description of how they'll test your code and a link to the spec for whatever thing you're building. Its done this way (I assume) encourage you to do your own research and not just implement the code to spec.

## Design decisions

So, knowing nothing about kafka, or how I should structure my code - my plan was basically to start dumping the minimum amount of code to pass each step in main.rs and figure out things as I went. To begin (IIRC) I just needed to bind to a port, easy peasy, then I needed to return a particular payload - and this is where the design (or lack thereof) immediately started to show.

### Choosing a parsing strategy

You see, for the kafka course, codecrafters (sometimes) gives you [binspecs](https://binspec.org/kafka-api-versions-response-unsupported-version) - an interactive guide to the bytes in a payload that you can use to get an idea of how the protocol works. My first thought was to go with old reliable - nom - but then I thought - well, nom style parsers are hard to reason about without understanding the spec - they're not declarative. Typically parsers are implemented by grabbing x number of bytes in order until all the input is consumed, but seeing a stream of `read_u32(&mut buf, &mut out)?` sounded really unappealing (and spoiler alert - this is actually how the two reference implementations do it.)

So I looked into what existed for declarative parsing - and found Zerocopy(.rs)

### An aside, wtf is zerocopy?

Typically when writing programs that perform IO, the IO is handled for you in the kernel and you're given a pointer or whatever to where to find the data. For network applications, typically in the background packets are accepted then put into memory for you somewhere. In high level programming languages it's normal to take these bytes, do some processing and then store the result in memory somewhere - thus copying it from the input to the output. But, if you know the layout of data in memory, you can simply operate on it as if you had already parsed it. IE, in C for example it's typical to have something like

```
typedef struct {
    int32_t x;
    int32_t y;
    int32_t z;
} MyStruct;

int main() {
    uint8_t network_buffer[sizeof(MyStruct)] = {
        1, 0, 0, 0,   // x = 1
        2, 0, 0, 0,   // y = 2
        3, 0, 0, 0    // z = 3
    };

    // trust me bro, network buffer is a point that points to some memory laid out
    // exactly like the struct
    const MyStruct* s = (const MyStruct*)network_buffer;

    printf("x=%d, y=%d, z=%d\n", s->x, s->y, s->z);
    return 0;
}
```

where you basically just tell the computer, yeah trust me, this block of memory is this struct. The benefit to zerocopy is that it cuts down both on CPU time and memory required for your program to run; you save the time it takes to copy the bytes, and you save some number of traversals of the bytes. And in fact, when combined with streaming you can potentially cut this time down even further - just read enough bytes to know if you need to read more bytes, or skip them if you don't.

The downside however is you're allowing an external party to tell your computer what should be in memory which is kind of risky as you can imagine - in fact this is basically one of the most common way C programs are hacked[^hacked].


Rust of course, mostly gives you the option to do it either way - you can read the bytes and copy them into a struct very safe like - or you can use `unsafe { transmute! } to do the same thing as (const MyStruct*) in the above example.

So back to Zerocopy.rs - I don't trust myself enough to use `unsafe` unless I really really need to - and I thought I was going to really really need to here, but google saved the day with zerocopy.rs. Long story short, while it _does_ internally call `unsafe { transmute! }` it does many compile time checks to ensure that what you're doing is as safe as possible and won't compile if certain traits aren't implemented such as `KnownLayout`

### #[Repr(C, Packed)]

The first thing I learned is that by default rust will _not_ put structs into memory in the same order as you write them by default. So for example

```
//input is like [0x00, 0x00, 0x00, 0x01, 0x02]
struct KafkaHeader {
  message_size: u32,
  magic_byte: u8
}

who knows if the first 4 (u32 = 4*8 bytes) start from the beginning of this input or not? Solution? Force rust to represent the structure like C, with repr(C) - though that doesn't immediately work because by default rust adds paddings between struct fields in memory so you also need `packed`.

Unfortunately this only works if your types are statically sized - and since I started with the kafka header which _is_ statically sized, I was lulled into a false sense of success. The problem is, 90% of the kafka protocol looks something like

```
{
  message_size: u32,
  num_records: u32,
  records: [Record {
    record_size: u32,
    key_size: ...
  }
}
```

Which means I can't simply simply cast the bytes into a struct since the fields are almost always arrays of Dynamically Sized Types -  DSTs.

### Slice DSTs

So,  decoding the header was easy, but then DSTs bit me. Rust, unlike dynamic languages, needs to know the size of every object, and if the size of kafka objects were fixed, you could do simple math like in C, size_of(Object) * Num objects and be done. However as shown, kafka has dynamic numbers of dynamic sized objects which gives us two problems:

1) You need to decode the struct to know how much to read
2) You "can't" initialize DSTs with data, which makes serializing objects to reply with next to impossible.

As the Rustonomicon says:

```
Unfortunately, such a type is largely useless without a way to construct it. Currently the only properly supported way to create a custom DST is by making your type generic and performing an unsizing coercion [...]
(Yes, custom DSTs are a largely half-baked feature for now.)
```


And, that felt like it was going to be a PITA. Not because its technically complicated or hard to implement but as a reminder - the goal with the parsing logic is that someone could look at the structs and immediately understand what's going on.

That plus another issue that's not coming to mind made me skip that and see if there wasn't something else.

## Deku to the rescue (?)

So I found out there are two other libraries people sometimes use for declarative parsing - Scroll and Deku. I won't talk too much about scroll, but I did find a great [comparison to zerocopy here](https://swatinem.de/blog/magic-zerocopy/) for those interested


The TLDR for deku is provides a nice declarative api for parsing bytes (or bits even) into structs, at the cost of not being zero copy. While this is disappointing, it was much easier to work with, and achieved my goal of (purely) declarative parsing - or so I thought; unfortunately the kafka protocol had other plans for me.

For the deserializing side of the project, I needed to read in what objects to expect, how many of them, then once I started reading them, how big they were. This got kind of messy and I ended up with a bunch of matches and partial reads with manual buffer manipulation (the exact thing I wanted to avoid)

The serialization side was not much better. Somewhat unexpectedly the kafka protocol works like a burrito - or Matryoshka[^doll] doll or something. Instead of being able to generate the structs and return them, you need to recurse to the deepest struct, generate it, measure its size, pass that size to the outer struct as a parameter so you can get your foo_size parameter etc etc until you read the top.

In kafka_protocol, they do this by passing a buffer down from function to function - but again, I was trying to avoid this sort of buffer manipulation. So I ended up with a worse solution - generate separate buffers, fill them in different ways depending on where in the code, them combine them by writing them to a final output buffer - its as gross as it sounds.

Anyway - TLDR; serializing and deserializing was the second hardest part - I'd do it differently next time.

## The hardest part

As mentioned, codecrafters was fairly hands off. Although they did provide binspecs, they often led me astray since most values in kafka are contextual, yet the binspec provides them as fact. So it'll say things like, records_len 0x04, the number of records minus one so the length is 3. I thought that was super weird until I learned later its contextual - some values are nullable, and null is either -1 or 0xff, some values are `VarInt` - which is an algorithm from protobufs that encodes a number in the least number of bytes possible - thus the field can shrink or grow. Some numbers are a fixed length, then they're unsigned int - 0 to 0xff or whatever. "Just read the kafka spec bro" - you'd think it'd be that easy but the kafka spec has been built overtime, and is scattered. Worse - the spec doesn't describe the actual bytes that go over the network, just the data you can expect to see in them, which highlighted the hardest part - framing.

Binary protocols and generally not self describing - they don't provide the schema to read them. Kafka's proto is no different, the bytes just flow back to back, which means, if your math is off or you skip parsing a field (or the field is too small etc) your data ends up decoded wrong and you get a hard to debug error.

You do develop a knack for noticing it though - in the last round I saw

```
foo: [00, ff]
producer_id: [ff, ff, ff, 02]
magic_byte: [00]
```

I know magic byte should be 0x02, and producer_id should be [ff,ff,ff,ff] which means, somewhere we missed decoding 1 byte. Yep it was the "tag buffer" - not documented but usually comes at the end of all the records.

## Compact Arrays and how I learned to love HRTB

Recall, (I think I mentioned) that kafka typically tells you the number of items before the type of item - it calls this concept a CompactArray. CompactArray (typically) takes a VarInt for the length, and of course <T>s back to back. When I first started writing the code I'd do

```
#[derive(DekuRead, DekuWrite)]
struct ThingContainer {
  thing_array_length
  #[deku(count=thing_array_length.saturating_sub(1))] // <- to counteract the -1 off by one thing from before
  things: Vec<Thing> // <- Vec is why Zerocopy went out the window
}
```

That can of course be made generic, `CompactArray<T> { length: VarInt, contents: Vec<T>}` - but the rust compiler doesn't like this. It says, you can't prove that every possible T can be De(Serialized) by Deku. OK fair enough, `Impl<T> for CompactArray<T: DekuWrite + DekuRead>, compiler says nope, Deku takes a reference to a buffer that has a lifetime 'a, you can't prove every 'a T lives long enough to be (De)serialized.

Fair enough - fortunately we can use Higher-Ranked Trait Bounds, where higher ranked just means like - trait bounds for trait bounds, or more specifically, generic trait bounds. So we have to tell rust - "OK, For all <T>, we must have a lifetime  (calling this lifetime 'a) that can be used as the lifetime that lives long enough for deku's 'a (calling it 'b because it's a parameter - I know confusing) lifetime. Or in rust:

```
impl<'a, T> DekuReader<'a, Endian> for CompactArray<T>
where
    T: for<'b> DekuReader<'b, Endian>,
```

And even though I cal wield it, and kind of parse it - its still like magic. In fact, the first example of using DekuRead + DekuWrite was my first attempt, then I was super confused why I couldn't serialize a type that only had DekuWrite. Yeah - because I literally told the compiler, [implement this trait for all <T> that are BOTH DekuRead and DekuWrite](https://github.com/edude03/deku-ctx/commit/44901329377271ae88c6f2ae9865b5ed5fa03617#diff-42cb6807ad74b3e201c5a7ca98b911c5fa08380e942be6e4ac5807f8377f87fcL21)

Anyway, with that resolved, it removed a bunch of counting bugs (since I didn't use varint or -1 consistently everywhere) and made the code a lot cleaner.

## Type Aliases & Partial Application

The one smart thing I did when I started this is refused to use magic numbers / "magic types". Kafka has a lot of places where in the documentation it says the type is `i32` or an array of 16 integers. Typically these numbers had a meaning like "BrokerID" or "Partition ID" so to keep myself sane I defined a bunch of type BrokerID = [u8; 16] and `FooID = i32`

This was great, because later I learned I could replace these type aliases with their proper types (like BrokerID = UUID), but better yet, I could come up partially applied aliases like


```rust
CompactArray<LengthType, InnerType>

type VarIntCompactArray<T> = CompactArray<VarInt, T>
type FooArray = VarIntCompactArray<Foo>
```

## Macro expansion

This feature used to be nightly only, now you can bring up the context macro on a random derive macro and see the code it generates. This was monumental to figuring out why deku didn't always do what I wanted (and gave me a template for what I needed to do to get it to do what I wanted)

## Cleaning up

My philosophy in rust is clone is part of the design, errors are not'. If you use `.clone()` it's not clear later if its a load bearing clone until you try and remove it and have to redesign and rewrite the surrounding code so I try to use them as sparingly as possible. `.unwrap()` `.expect()` `panic!()` however, I put that s*** on everything. After the code works as expected, I like to go back and find and replace the unhandled errors with `thiserror` and define errors for the things I know can go wrong.

In this project, I didn't end up going back for the `thiserror` pass (yet) because my other philosophy is errors are about routing information - in this case, being able to translate for example - a parse error to an application error, and that application error into a invalid request. In kafka land, I didn't implement enough of the API to really make use of that, Kafka errors tend to be not exactly errors in my option. For example should a get for a topic be `Result<Option<T>>` or `Result<T>`? I'd argue the former, I don't want to unwind the stack for something that's not exceptional, but the kafka protocol designers disagree



## Things I'd do better / differently next time

Everything. Honestly, writing a program from scratch with no spec and no planning and just ~~thugging that shit out~~ brute forcing a solution understandably leads to terrible code. With the gift of foresight, I'd say the biggest take away is to define all the types and their rust equivalents up front, with tests. The vast majority of my time was debugging when the input was a slightly different shape than expected.

With the types solid, next I'd have separate concepts for "Raw" types (types that came from decoding the bytes) and ... higher level "conceptual types". In the code I mixed both, but again it led to spaghetti code since I often needed to import the "raw" types all over the place to make their "conceptual types" - basically everywhere you see `pub` its like that.

Thirdly, I'd make my own tools / test harness sooner. Around step 17 of 18, I found out you can run the codecrafters test harness locally, AND wireshark has a built in kafka protocol inspector. I've already started testing my implementation against java kafka (for a secret part 2 of this post, stay tuned) and that's been way more helpful than dumping hex to console and hoping I could read the tea leaves

Forthly - contentious but I probably would just write my own (de)serialization framework. I'm a stanch believer in using the thing that's well tested and documented over rolling your own, but I ended up having to beat deku into shape implementing its traits by hand for everything but the build in rust types anyway - combined with the "buffer passing" approach I think the code would have come out cleaner, I would have finished faster and ended up with a proper separation between raw and conceptual types

## Fin

Anyway, hope you enjoyed and it wasn't too long and boring. I plan to tackle the codecrafters bittorrent course next and blog about it as well. If you're interested in signing up here's a [referral link](https://app.codecrafters.io/r/delightful-clam-462236).

Oh, and if you've made it this far and aren't scared to see how bad the code is, it's [here](https://github.com/edude03/toy-kafka-implementation)


[^hacked]: TLDR, some programs expect the input to be of a certain size, but will just read the input without making sure important memory locations, like flags or return addresses aren't overridden.
[^doll]: https://en.wikipedia.org/wiki/Matryoshka_doll
