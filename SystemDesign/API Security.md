
## TLS Not Just Encryption (The "Three Pillars")

TLS actually provides three critical things that TCP cannot:

1. **Encryption (Confidentiality):** Hides data from eavesdroppers.
    
2. **Authentication (Identity):** Uses certificates to prove you are talking to `google.com` and not a hacker's "Man-in-the-Middle" server.
    
3. **Integrity:** Uses a Message Authentication Code (MAC). If a hacker tries to flip even one bit of the data while it's traveling over TCP, TLS will detect the mismatch and drop the connection.
    

---