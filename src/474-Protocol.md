\[\[Category Packet\]\] \[\[Category Packet 474\]\] \[\[Category RS2\]\]

Not much progress has been made with this revision, don't worry about it
too much.

== '''Packet structure''' == When the client sends a packet to the
server, the first byte encapsulates its opcode. This specific opcode is
encrypted with a value generated by the ISAAC PRNG seeded with a
dynamically server generated key during the login block. The server
decrypts it and associates the opcode to the packet's respective
predefined size. If the packet does not contain a fixed size, the opcode
will be followed by either a byte or a word - varying per packet - for
its proper size. This is then followed by the payload.

== '''Login''' == ?

== '''Game Protocol''' ==

===Server -\> Client Packets===

  {
  ---

! Opcode ! Type ! Length (bytes) ! Name ! Description \|- \| 209 \|
VARIABLE\_BYTE \| N/A \| \[\[474 Send message\|Send message\]\] \| Sends
a server message (e.g. 'Welcome to RuneScape') or trade/duel request.
\|- \| 231 \| VARIABLE\_SHORT \| N/A \| \[\[474 Send string\|Send
string\]\] \| Replaces a string of text. (e.g. Replace: 'Click here to
Play' with 'Play Now!') \|- \|}

===Client -\> Server Packets===

  {
  ---

! Opcode ! Type ! Length (bytes) ! Name ! Description \|- \| 1 \| FIXED
\| 8 \| \[\[474 Remove ignore\|Remove ignore\]\] \| Sent when a player
removes a player from their ignore list. \|- \| 3 \| FIXED \| 6 \|
\[\[474 Fourth Interface Option\|Fourth Interface Option\]\] \| This is
triggered when a fourth interface option has been clicked. \|- \| 4 \|
FIXED \| N/A \| \[\[474 Fourth Object Option\|Fourth Object Option\]\]
\| This is triggered when a fourth object option has been clicked. \|-
\| 11 \| FIXED \| N/A \| \[\[474 Minimap Walk\|Minimap Walk\]\] \| Sends
walking data to the server. \|- \| 13 \| FIXED \| 12 \| \[\[474 Item on
NPC\|Item on NPC\]\] \| Sent when a player uses an item on an NPC. \|-
\| 21 \| FIXED \| 2 \| \[\[474 Npc action 2\|Npc action 2\]\] \| Sent
when a player clicks the second option of an NPC. \|- \| 22 \| FIXED \|
N/A \| \[\[474 Kick Clanchat Participant\|Kick Clanchat Participant\]\]
\| Indicates a friend of the clanChat owner attempts to kick a fellow
clanChatParticipant (non-owner). \|- \| 29 \| FIXED \| N/A \| \[\[474
Sixth Interface Option\|Sixth Interface Option\]\] \| Tells the server a
sixth interface option has been clicked. \|- \| 31 \| FIXED \| 6 \|
\[\[474 Object action 1\|Object action 1\]\] \| Sent when the player
clicks the first option of an object, such as "Cut" for trees. \|- \| 34
\| FIXED \| N/A \| \[\[474 Client Focus\|Client Focus\]\] \| Tells the
server the clients focus has changed. \|- \| 35 \| FIXED \| N/A \|
\[\[474 Use Magic On Player\|Use Magic On Player\]\] \| Indicates the
player wants to use a spell on another player. \|- \| 37 \| FIXED \| N/A
\| \[\[474 Third interface option\|Third interface option\]\] \| This is
triggered when one a third interface option has been clicked. \|- \| 203
\| FIXED \| 6 \| \[\[474 Object action 2\|Object action 2\]\] \| Sent
when the player clicks the second option of an object, such as
"Use-quickly" for Bank Booths. \|- \| 34 \| FIXED \| 1 \| \[\[474 Focus
change\|Focus change\]\] \| Sent when the game client window goes out of
focus. \|- \| 35 \| FIXED \| 8 \| \[\[474 Magic on player\|Magic on
player\]\] \| Sent when the player casts magic on another player.\
\|- \| 226 \| FIXED \| 2 \| \[\[474 Examine object\|Examine object\]\]
\| Sent when you examine an object.\
\|- \|}