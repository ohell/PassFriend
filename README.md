# PassFriend
is an authentication protocol enabling clients to securely authenticate with multiple services without needing to choose unique password per service

This text formally describes the protocol. The accompanying [Illustration](./Illustration.md) contains a more readily accessible explanation.

# Background
All current authentication protocols require the client to maintain a confidential token for each service that she has been granted access to. Sharing of tokens is stridently discouraged because the client does not trust any server beyond the parameters of the service being provided. This creates the need for the client to maintain a secured map of authentication tokens, unwieldy when the client has multiple possible channels of communication with the servers:  the tokens map must be replicated across all channels, in a fully secure manner. This is only possible for channels where the client endpoints support secure communication. Else the client has to individually replicate the authentication tokens when using new channels. This creates unreasonable burden on the clients’ storage, retrieval and transmission capabilities, as well as creating eavesdropping opportunity for interested observers.

In order to keep the storage and retrieval overhead manageable for clients, it is common for clients to generate the token used for secure communication while services mandate constraints on sequence of symbols comprising acceptable tokens, intended to minimise the chance of a token being duplicated by a stochastic trials process. These constraints are often ill-motivated due to the mismatch between the distribution over alphabets being used by the service to posit pattern constraints, and by a potential attacker to replicate the token (e.g. knowledge of the constraints decreases the entropy of the conditional distribution to be sampled from at each position). Pattern constraints also have the unintended consequence of clients getting overwhelmed by the complexity of storing multiple complex tokens, and reverting to recycling tokens for multiple services. 

# Outline
I propose a flexible protocol for the generation and use of authentication tokens intended to minimise a) storage burden on the client, b) eavesdropping utility during transmission, and c) chance of stochastic inference. I will also provide reference implementation of the software system to generate, manage and transmit these tokens.

The key idea is that instead of storing multiple tokens for different services, the client stores a single long master sequence of symbols with lax constraints on the entropy (thus allowing efficient storage and retrieval). Service-specific tokens are generated from subsequences sampled from this master sequence (response), paired with the sequence of sampled indices (challenge). Both sequences are obfuscated as described below and transmitted to the server as challenge-response pair: authentication is performed by the client requesting the server for a challenge, and transmitting the appropriate response. Subsampling algorithm is designed to generate challenges following a stochastic procedure aiming to maximise response entropy.

The service never gets to know the actual subsequence comprising the response due to obfuscation. Further, obfuscation algorithm utilises a unique identifier for the service, as well as the client identifier specific to the service. So different services do not get the same token, even when the challenge and response are identical. 

The protocol allows multiple challenge-response pairs to be generated for the same service, allowing it to choose unique tokens for proximate sessions order to minimise the potential for eavesdropping. Should a service get compromised, it only needs to alter its unique service id to render the previous tokens database obsolete, and without requiring the client renew tokens for other services. If the client needs to use a new channel for a known service, the challenge-response pairs stored with the service remain valid and can be used to authenticate.

# Caveats
If client loses the stored master sequence, there is no way to authenticate with any service: there would have to be a separate transaction with each service to reestablish trust. However, the premise is that the loss of the single master sequence would be a very unlikely event.

There is also no provision to authenticate the server - it is possible for an imposter with access to the server’s token storage to communicate with the client. There are other well established protocols to guard against this, e.g. certification by trusted identification services.

# Concepts 
1. The key concept ensuring the security of transactions under this protocol is Obfuscation (aka hashing), This is a function that deterministically maps any given sequence to another fixed-length sequence such that the there is no operation to recover the original sequence from the obfuscated sequence.  i.e. |Oi(s)| = i,  s1 = s2 => Oi(s1) = Oi(s2). There does not exist any O’ s.t. O’(Oi(s)) = s. 
2. Obfuscation can be reversible if the operation uses an unrelated sequence to derive the obfuscated sequence, and the same key can be used to map the obfuscated sequence to the original sequence (e.g. XOR). i.e. s1 = s2 <=> P(s1, sk) = P(s2, sk) and there exists at least one Q s.t. Q(P(is, sk), sk) = si
3. Checksum of a sequence is similar to obfuscation, but the operation cj, with j being the length of the checksum, is chosen such that P(cj(s1) == cj(s2) | s1 <> s2) —> 0 as j —> min(|s1|, |s2|)
4. Judicious sampling is the operation to select a given number of elements from a sequence subject to the constraint that the distribution of the symbols in the selected subsequence is wider than the distribution of symbols in the sampled sequence  (duplicates permitted). J(s, m) = r, |r| = m, and if H denotes the entropy of a distribution,  
for all k, j H(s[rj] | s[r[1…j-1]]) >= H(s[k] | s[1…k-1]) 
5. Interlacing is the operation to combine k sequences into a single sequence such that the jth element of each sequence is placed after the first j-1 elements of all sequences. i.e. Int(s1…sk) = f where f[j] = sb[j/k], b = j%k. Interlacing is useful to counter attempts to infer the hashed text by use of rainbow tables.

# Protocol 
## Registration: 
1. Client initiates the registration transaction by transmitting its desired client_id (and optionally the service_id) to the server
2. Server checks if this registration request is acceptable, and replies either with denial, or consent accompanied by the public service_id and the maximum and minimum number of challenge-response tokens it expects, Nt and nt
3. Upon receipt of consent client transmits to the server a list of Nt >= N >= nt challenge-response tokens generated following this strategy:
   1. Generate a short checksum from the master sequence  qm = ck(Ms)
   2. Select a token length L at random between min and max token sequence lengths lt and Lt (lt is chosen for sufficiently large number of possible sequences, Lt for the ease of input) 
   3. Generate a sequence pair (It, St) where It = J(Ms, L) contains L indices in the client’s master sequence Ms and St = Ms[It]
   4. Construct the challenge token Ct = P(It+ck(It), qm). i.e. reversible obfuscation of the It appended with its checksum, with qm as key.
   5. Construct the response token Rt = Ok(Int(client_id,St,service_id)) i.e. obfuscation of St, client_id & server_id. k should be reasonably large to avoid collisions
   6. Token Ti = (Ct, Rt)
4. Server stores the client_id and the list of tokens as client credentials, and transmits the registration success to the client

## Authentication:
1. Client initiates the authentication transaction by transmitting its client_id  (and optionally the service_id) to the server
2. Server finds the appropriate stored credentials for the client_id, and select a token Ti = (Ct, Rt) from the list of tokens stored as credentials, and transmits (service_id, Ct) as challenge to the client
3. Upon receipt of C’t, client validates the server and transmits a response following this strategy:
   1. Invert the obfuscation Q(C’t, qm) = It+Its
   2. If Its <> ck(It), abort transaction because server can’t be trusted
   3. Construct the response token R’t = Ok(Int(client_id,Ms[It],service_id))
4. Server authenticates the client if Rt = R’t and transmits the authentication success to the client
5. Obsolescence: Server can respond to the authentication request with an indication that the stored credentials are no longer valid
   * Server responds with an ‘obsolete’ response accompanied by the current service_id and the maximum and minimum number of challenge-response tokens it expects, Nt and nt
   * Client has the option of abandoning the authentication attempt, or continue Registration step 3

## Deregister:
1. Client initiates the deregister transaction by transmitting the client_id to the server
2. Server validates the request by continuing the authentication steps 2, 3, 4
3. Server deletes the client_id and the associated tokens from its list of stored credentials and transmits acknowledgement to the client
