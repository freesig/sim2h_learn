## priv_check_incoming_messages
This is checking any incoming messages that are sent to sim2h.
```rust
/// if our connections sent us any data, process it
fn priv_check_incoming_messages(&mut self) -> bool {
```
Some metrics macro
```rust
    with_latency_publishing!(
        "sim2h-priv_check_incoming_messages",
        self.metric_publisher,
        || {
```
Simply send the messages to the debug log if there are any.
```rust
            let len = self.msg_recv.len();
            if len > 0 {
                debug!("Handling {} incoming messages", len);
                debug!("threadpool len {}", self.threadpool.queued_count());
            }
```
Iterate through all the messages
```rust
            let v: Vec<_> = self.msg_recv.try_iter().collect();
            for (url, msg) in v {
```
Convert from a Url2 to a Url and into a Lib3hUri
```rust
                let url: Lib3hUri = url::Url::from(url).into();
```
```rust
                match msg {
                    Ok(frame) => match frame {
```
Should never get a text message and if we do we drop this connection.
```rust
                        WsFrame::Text(s) => self.priv_drop_connection_for_error(
                            url,
                            format!("unexpected text message: {:?}", s).into(),
                        ),
```
This is where all the work happens.
This is a binary message that we will try and verify then send to the right place.
```rust
                        WsFrame::Binary(b) => {
                            trace!(
                                "priv_check_incoming_messages: received a frame from {}",
                                url
                            );
```
Turn this into a Vec of u8 in base64
```rust
                            let payload: Opaque = b.into();
```
Verify the payload is signed correctly.

!!! note
    Jump to [verify_payload](./#verify_payload)

```rust
                            match Sim2h::verify_payload(payload.clone()) {
                                Ok((source, wire_message)) => {
                                    if let Err(error) =
```
The message was valid now handle it.

!!! note
    Jump to [handle_message](./#handle_message)

```rust
                                        self.handle_message(&url, wire_message, &source)
                                    {
                                        error!("Error handling message: {:?}", error);
                                    }
                                }
                                Err(error) => {
                                    error!(
                                        "Could not verify payload from {}!\nError: {:?}\nPayload was: {:?}",
                                        url,
                                        error, payload
                                    );
                                }
                            }
                        }
                        // TODO - we should use websocket ping/pong
                        //        instead of rolling our own on top of Binary
                        WsFrame::Ping(_) => (),
                        WsFrame::Pong(_) => (),
                        WsFrame::Close(c) => {
                            debug!("Disconnecting {} after connection reset {:?}", url, c);
                            self.disconnect(&url);
                        }
                    },
                    Err(e) => self.priv_drop_connection_for_error(url, e),
                }
            }
            false
        }
    )
}
```

## verify_payload
Verify that this is a signed message and convert to WrieMessage.
A WireMessage is a typed message between the client and lib3h.
The caller gets back the agents id who sent it plus they know it's cryptographically signed and the message structure is valid to the WireMessage enum types.
```rust
fn verify_payload(payload: Opaque) -> Sim2hResult<(AgentId, WireMessage)> {
```
try to turn this from JSON to a provenance and a payload.
```rust
    let signed_message = SignedWireMessage::try_from(payload)?;
```
Verify this message by putting it into insecure memory and checking the signature matches the payload.
```rust
    let result = signed_message.verify().unwrap();
    if !result {
        return Err(VERIFY_FAILED_ERR_STR.into());
    }
```
Convert the payload from JSON to a WireMessage. This converts the payload to an actual type. This is how we know it's a valid message atleast in structure.
```rust
    let wire_message = WireMessage::try_from(signed_message.payload)?;
    Ok((signed_message.provenance.source().into(), wire_message))
}
```

## handle_message
```rust
fn handle_message(
    &mut self,
    uri: &Lib3hUri,
    message: WireMessage,
    signer: &AgentId,
) -> Sim2hResult<()> {
    with_latency_publishing!("sim2h-handle_messsage", self.metric_publisher, || {
        trace!("handle_message entered for {}", uri);

        MESSAGE_LOGGER
            .lock()
            .log_in(signer.clone(), uri.clone(), message.clone());
```
Get the connection state in either Limbo or Joined.
```rust
        let (uuid, mut agent) = self
            .get_connection(uri)
            .ok_or_else(|| format!("no connection for {}", uri))?;

        conn_lifecycle("handle_message", &uuid, &agent, uri);

        // TODO: anyway, but especially with this Ping/Pong, mitigate DoS attacks.
```
Simple ping message, respond with pong
```rust
        if message == WireMessage::Ping {
            debug!("Sending Pong in response to Ping");
            self.send(signer.clone(), uri.clone(), &WireMessage::Pong);
            return Ok(());
        }
```
Send a status message that sums up the network state.
```rust
        if let WireMessage::Status = message {
            debug!("Sending StatusResponse in response to Status");
            let (spaces_len, connection_count) = {
                let state = self.state.read();
                (state.spaces.len(), state.open_connections.len())
            };
            self.send(
                signer.clone(),
                uri.clone(),
                &WireMessage::StatusResponse(StatusData {
                    spaces: spaces_len,
                    connections: connection_count,
                    redundant_count: match self.dht_algorithm {
                        DhtAlgorithm::FullSync => 0,
                        DhtAlgorithm::NaiveSharding { redundant_count } => redundant_count,
                    },
                    version: WIRE_VERSION,
                }),
            );
            return Ok(());
        }
```
I think this is a discovery message. So when a node comes online it sends this.
```rust
        if let WireMessage::Hello(version) = message {
            debug!("Sending HelloResponse in response to Hello({})", version);
            {
                let mut state = self.state.write();
                if let Some(conn) = state.open_connections.get_mut(uri) {
                    conn.version = version;
                }
            }
            self.send(
                signer.clone(),
                uri.clone(),
                &WireMessage::HelloResponse(HelloData {
                    redundant_count: match self.dht_algorithm {
                        DhtAlgorithm::FullSync => 0,
                        DhtAlgorithm::NaiveSharding { redundant_count } => redundant_count,
                    },
                    version: WIRE_VERSION,
                    extra: None,
                }),
            );
            return Ok(());
        }

        match agent {
            // if the agent sending the message is in limbo, then the only message
            // allowed is a join message.
```
Check if the connection isn't joined and then add messages to pending messages on the heap.
If it's a join message then join the connection.
```rust
            ConnectionState::Limbo(ref mut pending_messages) => {
                if let WireMessage::ClientToLib3h(ClientToLib3h::JoinSpace(data)) = message {
                    if &data.agent_id != signer {
                        return Err(SIGNER_MISMATCH_ERR_STR.into());
                    }
                    self.join(uri, &data)
                } else {
                    debug!("inserting into pending message while in limbo.");
                    // TODO: maybe have some upper limit on the number of messages
                    // we allow to queue before dropping the connections
```

!!! warning
    Flagging this as a potential spot for memory to climb.
    What happens if the client stays in limbo and the messages keep rolling in.

```rust
                    pending_messages.push(message);
                    // MDD: TODO: is it necessary to re-insert the data at the same uri?
                    // didn't we just mutate it in-place?
                    let _ = self
                        .state
                        .write()
                        .connection_states
                        .insert(uri.clone(), (uuid, agent));
                    self.send(
                        signer.clone(),
                        uri.clone(),
                        &WireMessage::Err(WireError::MessageWhileInLimbo),
                    );
                    Ok(())
                }
            }
            // if the agent sending the messages has been vetted and is in the space
            // then build a message to be proxied to the correct destination, and forward it
            ConnectionState::Joined(space_address, agent_id) => {
                if &agent_id != signer {
                    return Err(SIGNER_MISMATCH_ERR_STR.into());
                }
                self.handle_joined(uri, &space_address, &agent_id, message)
            }
        }
    })
}
```