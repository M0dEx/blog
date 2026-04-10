+++
title = "Rolling your own crypto can make sense (sometimes)"
date = 2026-04-09T22:40:31
description = "A short rationale about why I decided to roll my own crypto for Quincy."
[taxonomies]
tags = ["quincy", "reishi", "security", "cryptography"]
+++

A couple of months ago, I took on the endeavour to add support for the [Noise handshake](https://noiseprotocol.org/noise.html) into [Quincy](https://github.com/quincy-rs/quincy), for easier user identity handling and some anti-detection benefits. This took me down the rabbit hole of how sometimes implementing your own crypto can make sense, depending on your use case, and it is the story of how [Reishi](https://github.com/quincy-rs/reishi) came to be.

### Dying of thirst in the middle of the ocean
When Noise was initially brought up in [one of the issues](https://github.com/quincy-rs/quincy/issues/142) on the Quincy repository, I was prompted to do a quick search into existing implementations that could possibly fit what I needed without having to do too much manual labour. I was already familiar with Noise at that time, since it is the handshake protocol [WireGuard](https://www.wireguard.com/) uses and one of the reason its setup is so beautifully simple.

I had a pretty specific wishlist for an "ideal" Noise implementation, mostly based on the needs of Quincy and the needs of me, as a sole maintainer:
- well-tested implementation with strong security properties (proper zeroization of sensitive material, minimal security surface, secure defaults, ...)
- support for the IK mode ("server" public identity is known beforehand, "client" public identity is sent during the handshake)
- easy integration into [`quinn`](https://github.com/quinn-rs/quinn)
- support for post-quantum or hybrid handshake

There are a couple of both well-known and well-made crates implementing almost the entirety of the Noise protocol - [`snow`](https://github.com/mcginty/snow), for example. There are even some crates that take `snow` and implement it as a `quinn` "provider", so that you can use it in projects such as Quincy with relative ease. While `snow` is, at least in my opinion, well made and tested, it lacks support for post-quantum or hybrid handshake modes, mostly due to them not being standardized in the Noise protocol spec (yet).

Then there is a slew of barely-known crates attempting to implement the Noise protocol in slightly different ways, usually with near zero traction (GitHub stars, downloads, maintenance, ...), but they suffer from much of the same issue as `snow` - lacking post-quantum safety and generally not being easily integratable with `quinn`.

### xkcd.com/927

<img src="https://imgs.xkcd.com/comics/standards_2x.png" alt="xkcd.com/927" width="600" center=true/>
    
So, there is a dozen of crates that do most of what I want, a lot of what I do not need, and do not do some of the stuff I desperately need. 
In such a situation, the solution is, of course, making a 13th crate that fits my needs exactly. 

Contributing to an already existing crate? No, such things are for people with entirely too much time on their hands. I am only sort of kidding, getting post-quantum support into these crates would be a strenuous task, given that it is not standardized and I do not have a history of _any_ open-source cryptographic implementations. The more important reasoning at the time was to limit the maintenance and security surface of the implementation to only and solely what I needed for Quincy. This is, in my opinion, one of the best things you could do for the security of any product. In the end, the best code you can have in prod is no code at all.

It is not like I would be writing the primitives myself, at least not most of them. The underlying ciphers, hashing algoritms etc. are already provided by the perfectly capable folks from [rustcrypto](https://github.com/rustcrypto), which I trust and which have been proven, time and time again, to know what they are doing. The crux of the implementation would be "just" to wire the already existing pieces and rewrite the spec's pseudocode into Rust. 

See, my mistake was in thinking "Sounds reasonable, why not try?" instead of thinking "Fuck no."

### Is it secure?

TL;DR: possibly, maybe, I hope

This is a question that might sound basic, or easy to answer, but it is actually a very complicated topic. Generally speaking, it is very, very, extremely, unreasonably hard to prove that something is truly secure. You could have a 20 year old implementation of the most scrutinized algorithms, with all the bells and whistles - SAST, DAST, formal verification, extremely granular test coverage - and all it would take for someone to prove that it is not secure is to find _one_ vulnerability, _one_ logic bug, to destroy that confidence.

While this might sound scary, and that there is no point, do not lose hope. Even the largest, most scrutinized cybersecurity projects in the world have new vulnerabilities found each year, yet people still use them and trust them. This trust was not always present, and it needs to be built from the ground up. Trusting them is, ultimately, the best we can do.

What precautions did I take with with Reishi? As many as humanly could, given that it is not a PhD. thesis, nor a funded security project:
- minimal surface - only the stuff I needed and nothing more
- denying any unsafe code at the crate root with `![deny(unsafe_code)]`
- zeroizing all cryptographic material
- using constant time comparisons for any parts that could be subject to timing or side channel attacks
- granular test suite testing not only the happy paths, but the edge cases, weird states, etc.
- integration tests to verify the security properties of the established handshake
- interop tests with `snow` to ensure spec compliance without formal verification

Can I, with absolute certainty, guarantee that Reishi is secure? No, I do not think anybody can guarantee that about anything. But I trust it with my own data, and I have done the learning and the work to earn that trust for myself. Should you trust it, and by extension, me? Read the code, try to break it, and file an issue if you do.

### Famous last words

So, does rolling your own crypto make sense? Generally speaking, no. But "generally" is doing a lot of heavy lifting. If your needs are narrow enough, if you use already battle-tested primitives, and you are actively trying to make the attack surface as small as possible, it can be the right call. The less you implement, the less you can get wrong. Reishi does exactly what Quincy needs and not one thing more, and that is a feature, not a bug or room for improvement.
Of course, maybe the next post will be titled "How someone pwned me by breaking Reishi" and I will be proven an absolute fool, but oh well.
