# Observability

> Observability refers to understanding a system through observation…
> 
> (Brendan Gregg, [BPF Performance Tools][bpf-perf-tools])

[bpf-perf-tools]: https://www.brendangregg.com/bpf-performance-tools-book.html


One of the most important properties of well-designed applications is their *observability*. We have to be able to inspect the relevant state of our application in detail, and we have to do it both in the past and present. 

Let's say our program is a basic web server hosting some CRUD app. It's running normally, and then our continuous monitoring tool reports abnormally slow response times. We see this alert an hour after it comes in. By then, response times are back to normal.

Ideally we would be able to answer the following questions.

- **Why were the response times slow?**
  - It was due to a spike in traffic that led to high CPU utilization.
- **When did this spike in traffic occur?**
  - Between 10–11 AM.
- **What IP addresses were making the queries causing the spike?**
  - It was two IP addresses `X` and `Y`.
  - Investigation shows they're both hosted on Amazon EC2.
- **What endpoints were queried by those IPs during the spike?** 
  - Both `/login` and `/signin` were queried in approximately equal volume.
- **Which endpoint ate up the CPU?**
  - The `/login` endpoint caused the high CPU utilization.
- **For the same number of requests, why does one endpoint use more CPU?**
  - The `/login` endpoint was performing `argon2i` password hashes
  - The `/signin` endpoint was instantly returning `404 Not Found`.
- **Were any of those requests successful?**
  - No, they all returned `401 Unauthorized`.

From this information, we would be able to understand that the first-order cause of our slow responses was high CPU usage. The second-order cause was that an attacker was attempting to gain access to an account (probably) by trying a password list. The third-order cause was either that our rate limit on `/login` attempts is too high, or our settings on `argon2i` hashing are too computationally expensive.

We don't usually know all of the exact questions we'll be asking about our application in the future. An *observable* application is one that supports asking novel questions without needing to be re-programmed.

This can lead to a balancing act. We want to be able to answer critical questions about an application's past and present state, but observability comes with a small, but non-zero runtime cost. If we capture every packet sent to and from a web server, we'll have incredible visibility into exactly what requests and responses we're sending out, the rate they're being sent, to whom and when, and so on. But we'll also have a massive amount of data to continously transmit to a central logging repository, and to store once the data get there.

Always profile your application so that you're aware of the runtime cost (or lack thereof) you pay for using various observability methods.


## Observability in Rust

- [tracing](./tracing.md): emit and collect event-based data